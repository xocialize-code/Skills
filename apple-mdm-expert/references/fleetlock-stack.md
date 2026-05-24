# FleetLock stack

This file translates Apple MDM concepts into the actual FleetLock implementation — TypeScript on Cloudflare Workers + Durable Objects + D1 + R2 + Cloudflare Containers, with NanoMDM (Go) as architectural reference. Read this whenever working in the FleetLock codebase or designing changes to its server architecture.

## The stack at a glance

| Concern | Component | Notes |
|---|---|---|
| HTTP edge | Cloudflare Workers (TypeScript) | All check-in and command endpoints |
| Per-device command queue + push-token cache | Durable Objects (one DO per device) | Strong-consistency boundary per device |
| Relational state (enrollment, devices, profiles, audit) | D1 | SQLite-compatible |
| Binary storage (profiles, IPAs, certs) | R2 | S3-compatible |
| SCEP identity issuance | Cloudflare Containers running `micromdm/scep` | Stateless besides CA key |
| Admin API auth | Cloudflare Access | Zero-trust SSO for admin endpoints |
| Plist parsing/serialization | `@plist/plist` (npm) | TypeScript plist lib; supports XML + binary |

NanoMDM is the **architectural reference** — its design (push provider, command queue, push certificate, MDM service identity boundaries) shapes FleetLock — but the actual implementation differs substantially because Workers ≠ Go and Cloudflare's primitives don't map 1:1 to Go's stdlib + sqlite.

## Representing plists in TypeScript

The MDM protocol is plist-native. `@plist/plist` handles parsing and serialization in both XML and binary forms:

```ts
import * as plist from '@plist/plist';

// Parse a check-in body
const checkInBody = plist.parse(await request.text()) as Record<string, unknown>;
const messageType = checkInBody.MessageType as string;

// Build a command
const command = {
  CommandUUID: crypto.randomUUID(),
  Command: {
    RequestType: 'DeviceInformation',
    Queries: ['DeviceName', 'OSVersion'],
  },
};
const responseBody = plist.build(command);  // XML by default
return new Response(responseBody, {
  headers: { 'Content-Type': 'application/x-apple-aspen-mdm' },
});
```

Devices send XML plists. Responses can be XML or binary plist — XML is fine, devices accept both.

Use **TypeScript discriminated unions** for command and check-in messages: a `CheckInMessage` type union with `MessageType` as discriminator gives compile-time safety and exhaustiveness checks.

```ts
type CheckInMessage =
  | { MessageType: 'Authenticate'; UDID: string; Topic: string; ... }
  | { MessageType: 'TokenUpdate'; UDID?: string; EnrollmentID?: string; Token: ArrayBuffer; PushMagic: string; ... }
  | { MessageType: 'CheckOut'; UDID?: string; EnrollmentID?: string; ... }
  | ...;
```

For device key, prefer a polymorphic field `device_key: { udid?: string; enrollment_id?: string }` — User Enrollment has no UDID.

## Durable Object: per-device command queue

The core FleetLock pattern: **one Durable Object per device** (keyed by UDID or EnrollmentID). The DO owns:

- Pending command queue (ordered)
- Latest push token + PushMagic
- Reference to identity cert (for signature verification)
- Last-checked-in timestamp
- Pending status report subscriptions

Worker handles a check-in or command-poll, looks up the device's DO by name (`env.DEVICE_DO.idFromName(deviceKey)`), forwards the request, gets a response, returns to device.

Why DO instead of just D1 + Worker:

| DO advantage | Notes |
|---|---|
| Strong consistency per device | No race between "device polls" and "admin queues command" |
| Single in-memory queue per device | No D1 round-trip on every poll |
| Per-device addressing | DO ID maps cleanly to device identity |
| Persistence via DO storage | Survives Worker reloads |

Pattern: the DO stores its queue in `storage` (key-value), pulls into in-memory state on first activation. Each method (enqueue, pop, peek, recordStatus) updates both.

```ts
export class DeviceDO {
  private state: DurableObjectState;
  private queue: PendingCommand[] = [];

  async fetch(request: Request) {
    const url = new URL(request.url);
    switch (url.pathname) {
      case '/poll': return this.handlePoll(request);
      case '/enqueue': return this.handleEnqueue(request);
      case '/checkin': return this.handleCheckIn(request);
      ...
    }
  }

  async handlePoll(request: Request) {
    const statusReport = plist.parse(await request.text());
    // record status of the last-sent command if present
    if (statusReport.CommandUUID) {
      await this.recordCommandStatus(statusReport.CommandUUID, statusReport.Status);
    }
    // hand back next command
    const next = this.queue.shift();
    if (!next) return new Response('', { status: 200 });
    await this.state.storage.put('queue', this.queue);
    return new Response(plist.build(next.command), { ... });
  }
}
```

For a fleet of N devices: N DOs, each tiny. DOs idle to disk when not in use; they're cheap.

## SCEP via Cloudflare Containers

`micromdm/scep` is a small Go service that takes a CSR, validates a challenge, returns a signed cert. Run it in Cloudflare Containers:

| Container responsibility | Detail |
|---|---|
| Hold the CA's private key | Mount at startup from a secret (encrypted at rest) |
| Handle SCEP HTTP requests | Receive CSR from device, validate, sign, return |
| Issue identity certs | One per `$UDID` in CN; cert validity matches your policy |

The Container fronts behind a Worker route. The Worker can pre-validate (single-use challenge, rate limiting, audit log) before forwarding. The Container itself stays simple — no business logic.

For ACME (newer path), `smallstep/step-ca` is the equivalent reference. It supports Apple device attestation.

## NanoMDM divergences

NanoMDM is Go + per-request sqlite. FleetLock is TS + DO + D1. Where the patterns diverge:

| NanoMDM (Go) | FleetLock (TS/CF) | Why |
|---|---|---|
| Single sqlite DB | D1 for relational + DO storage for per-device queue | DO gives per-device locking without contention |
| Per-device mutex via sync.Mutex | DO is the mutex | Single-threaded actor model is the boundary |
| Sequential push provider | Per-DO push via Worker fetch | Push fan-out is per-device, not centralized |
| Go's crypto for plist signing | Web Crypto API (no OpenSSL CLI) | Workers can't shell out |
| sqlite triggers / FTS | D1 supports limited triggers; FTS via virtual tables | Use D1 for queries, DO for per-device state |
| Push provider holds APNs HTTP/2 connection | Workers do APNs HTTP/2 per-request | No long-lived sockets in Workers |

The Worker model means push isn't a long-lived APNs connection but a per-request HTTPS POST. Cloudflare handles connection pooling under the hood, but each push is a discrete fetch.

## D1 schema sketch

A starting D1 schema (refine per FleetLock spec):

```sql
CREATE TABLE devices (
  device_key TEXT PRIMARY KEY,           -- UDID or EnrollmentID
  device_type TEXT,                       -- 'standard' | 'user_enrollment' | 'account_driven_user' | 'account_driven_device'
  udid TEXT,
  enrollment_id TEXT,
  serial_number TEXT,
  model TEXT,
  os_version TEXT,
  topic TEXT NOT NULL,                    -- MDM push cert UID
  identity_cert_pem TEXT,
  enrolled_at INTEGER NOT NULL,
  unenrolled_at INTEGER,
  last_checkin_at INTEGER
);

CREATE TABLE push_tokens (
  device_key TEXT PRIMARY KEY,
  token_hex TEXT NOT NULL,                -- hex-encoded APNs device token
  push_magic TEXT NOT NULL,
  updated_at INTEGER NOT NULL,
  FOREIGN KEY (device_key) REFERENCES devices(device_key)
);

CREATE TABLE commands (
  command_uuid TEXT PRIMARY KEY,
  device_key TEXT NOT NULL,
  request_type TEXT NOT NULL,
  body_json TEXT NOT NULL,                -- full command serialized
  status TEXT NOT NULL,                   -- 'queued' | 'acknowledged' | 'error' | 'format_error' | 'not_now'
  response_json TEXT,
  created_at INTEGER NOT NULL,
  responded_at INTEGER,
  FOREIGN KEY (device_key) REFERENCES devices(device_key)
);

CREATE INDEX idx_commands_device ON commands(device_key);
CREATE INDEX idx_commands_status ON commands(status);

CREATE TABLE profiles (
  profile_uuid TEXT PRIMARY KEY,
  payload_identifier TEXT NOT NULL,
  r2_key TEXT NOT NULL,                   -- R2 object key for the plist
  signed BOOLEAN NOT NULL,
  created_at INTEGER NOT NULL
);

CREATE TABLE bootstrap_tokens (
  device_key TEXT PRIMARY KEY,
  token_encrypted TEXT NOT NULL,          -- encrypted with KV-stored key
  escrowed_at INTEGER NOT NULL,
  FOREIGN KEY (device_key) REFERENCES devices(device_key)
);
```

Real schema will have more — audit log, DDM declarations table, app inventory, etc.

## Profile signing in Workers (no OpenSSL CLI)

Workers can't shell out. To sign a `.mobileconfig`:

- Use **Web Crypto API** for the actual signing (RSA-PKCS1-v1_5 or ECDSA).
- Construct the CMS / PKCS#7 SignedData structure manually, or use a CMS library that targets browsers/Workers (e.g., `pkijs` works in Workers with some shimming).
- Signer cert + private key live in Workers secrets / KV (encrypted at rest).

For one-off signing, building CMS manually is doable but tedious. `pkijs` + `asn1js` give you the building blocks if you want a library.

Alternative: pre-sign profiles offline (CI) and store the signed bytes in R2. Worker just serves the bytes. Reduces complexity at request time, useful when profiles are mostly static.

## APNs HTTP/2 from Workers

Workers' `fetch` supports HTTP/2. APNs accepts JWT-based provider auth (no per-connection client cert needed — preferred). Flow:

1. Sign a JWT with your APNs Auth Key (ES256, key ID + team ID in claims).
2. Cache the JWT for ~50 minutes (APNs allows up to 1h; rotate before).
3. POST to `https://api.push.apple.com/3/device/[hex(Token)]` with `authorization: bearer [JWT]`, `apns-topic: [Topic]`, `apns-push-type: mdm`.
4. Body: `{"mdm": "[PushMagic]"}`.
5. Read `apns-id` header and HTTP status; map errors per `errors.md`.

The APNs Auth Key (`.p8`) lives in a Worker secret. Don't ever log it.

## R2 for profile and binary storage

R2 holds:

- Signed `.mobileconfig` profiles (keyed by `profile_uuid`)
- `.ipa` files for Enterprise In-House distribution
- IPSW caches if you're doing on-device restores
- Audit log archives

R2 access is via `env.R2.put/get/delete`. For profile delivery, the Worker fetches from R2 and serves with `Content-Type: application/x-apple-aspen-config`.

## Error mapping back to FleetLock domain errors

MDM errors arrive in two shapes: HTTP-level (from APNs, from ABM/ASM API) and protocol-level (in command responses). FleetLock's domain error types should normalize these:

| Source | Map to |
|---|---|
| APNs 410 Unregistered | `DeviceUnenrolledError` (and trigger ABM/ASM reconcile) |
| APNs 400 BadDeviceToken | `StalePushTokenError` (await next TokenUpdate) |
| Command Status: Error | `CommandFailedError` with full ErrorChain attached |
| Command Status: CommandFormatError | `MalformedCommandError` (server bug; alert) |
| Command Status: NotNow | not an error — log and defer |
| ABM/ASM 401 + `AUTH_SESSION_TOKEN_EXPIRED` | `SessionTokenExpiredError` (re-auth + retry) |
| ABM/ASM 4xx other | `DeploymentServiceError` |

Distinguishing transient (retry) vs permanent (alert / unenroll device) errors is the key reason to normalize.

## Gotchas specific to this stack

- **DO storage has a 128KB per-value limit.** A device with a huge pending queue could hit this. Page the queue across multiple DO storage keys if needed.
- **Workers have a 50ms CPU time limit per request by default.** Plist parsing of large bodies (InstalledApplicationList responses) can approach this. Stream or move to a Durable Object alarm for heavy parsing.
- **D1 has eventual consistency across regions.** Per-device state in DOs avoids this. Don't put per-device hot state in D1.
- **Workers can't hold long-lived APNs connections.** Each push is a fresh HTTP/2 request. Cloudflare's edge layer pools, but expect occasional cold-handshake latency.
- **`@plist/plist` returns plain JS objects** without type discrimination. Always validate after parsing.
- **Web Crypto's CMS support is non-existent** — you assemble PKCS#7 yourself or import a library. Plan for this in your CI signing pipeline.
- **R2 is eventually consistent for list operations** but strongly consistent for read-after-write on the same key. Keep that in mind when storing profile updates.

## Quick links

- NanoMDM (architectural reference): `https://github.com/micromdm/nanomdm`
- micromdm/scep (SCEP server for the Container): `https://github.com/micromdm/scep`
- @plist/plist: `https://www.npmjs.com/package/@plist/plist`
- Cloudflare Workers docs: `https://developers.cloudflare.com/workers/`
- Cloudflare Durable Objects: `https://developers.cloudflare.com/durable-objects/`
- Cloudflare Containers: `https://developers.cloudflare.com/containers/`
