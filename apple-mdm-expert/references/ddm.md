# Declarative Device Management (DDM)

DDM is Apple's newer device-management layer, introduced in iOS 15 / macOS 13. Where classic MDM is **server-imperative** (server says "install profile X"), DDM is **declarative**: the server publishes a target state, the device continuously reconciles to that state, and reports status when things change. It rides the same transport as classic MDM and the two coexist.

Apple doc — concept: `https://developer.apple.com/documentation/devicemanagement/leveraging-the-declarative-management-data-model-to-scale-devices`
Apple doc — protocol integration: `https://developer.apple.com/documentation/devicemanagement/integrating-declarative-management`
Apple doc — declarations list: `https://developer.apple.com/documentation/devicemanagement/devicemanagement-declarations`
Apple doc — status reports: `https://developer.apple.com/documentation/devicemanagement/status-reports`
Schemas: `https://github.com/apple/device-management/tree/release/declarative`

## The mental model

Classic MDM:

> Server: "Install this Wi-Fi profile."
> Device: (later) "OK, installed."
> Server: "Now install this app."
> Device: (much later) "Installed."
> Server: poll, query, poll, query.

DDM:

> Server: "Here's a manifest. Your activations say: at all times, have Wi-Fi declaration A and app declaration B in effect."
> Device: Continuously evaluates the manifest. When something changes (network state, time-of-day predicate, device state), the device reconciles and proactively reports status.

This means **less command traffic**, **more device autonomy**, and **richer status data**. The trade-off is that DDM is harder to debug: the device decides when to reconcile, not you.

## Declaration types — the four buckets

Every declaration has a `Type` field with a reverse-DNS identifier in one of four namespaces:

| Namespace | Purpose | Examples |
|---|---|---|
| `com.apple.activation.*` | Predicates that turn other declarations on/off | `com.apple.activation.simple` |
| `com.apple.configuration.*` | The actual desired state | `com.apple.configuration.account.mail`, `com.apple.configuration.passcode.settings`, `com.apple.configuration.softwareupdate.enforcement.specific`, `com.apple.configuration.management.test` |
| `com.apple.asset.*` | Referenceable resources (credentials, certs, app stores) | `com.apple.asset.credential.scep`, `com.apple.asset.credential.userpassword`, `com.apple.asset.useridentity` |
| `com.apple.management.*` | Server↔device control / migration declarations | `com.apple.management.organization-info`, `com.apple.management.app` |

Configurations reference assets; activations reference configurations. The graph is acyclic: activation → configuration → asset.

## The DDM protocol — synchronization tokens

DDM uses a **pull model with synchronization tokens**. The transport summary:

1. Server activates DDM by sending a `DeclarativeManagement` check-in command to the device, including the URL where DDM lives (typically the same MDM server) and an initial `Data` payload.
2. Device makes DDM requests to that URL with a `Endpoint` in `tokens` / `declaration-items` / `declaration/{type}/{identifier}` / `status` form.
3. Server responds with the appropriate payload.

Four endpoints the device hits:

| Device request | Server returns |
|---|---|
| `GET tokens` | `{ "SyncTokens": { "DeclarationsToken": "...", "Timestamp": "..." } }` — a token representing current state |
| `GET declaration-items` | A manifest listing all declaration identifiers + per-declaration server tokens |
| `GET declaration/{type}/{identifier}` | A specific declaration's payload |
| `POST status` | (status report from device) — server returns 200 OK |

Device-side workflow:
1. Fetch `tokens`. If `DeclarationsToken` changed, the manifest has changed.
2. Fetch `declaration-items` to get the new manifest.
3. Diff: which declarations are new, changed (server token differs), or removed.
4. Fetch each new/changed declaration individually.
5. Reconcile state.
6. Post status reports.

You drive this loop by **changing server tokens** when state changes. Bump the per-declaration token to signal "this changed"; bump the overall `DeclarationsToken` to signal "the manifest changed (declarations added or removed)".

## Status reports

The device posts status reports asynchronously — when reconciliation completes, when state changes, when a configuration takes effect. The shape:

```json
{
  "StatusItems": {
    "device": { "operating-system": { "version": "26.2" } },
    "management": { "declarations": {
      "activations": [...],
      "configurations": [{
        "identifier": "com.example.fleet.passcode",
        "valid": "valid",
        "active": true,
        "server-token": "...",
        "reasons": []
      }]
    }}
  },
  "Errors": []
}
```

Status reports replace the "send a query command, wait for response" pattern. Configure your status subscriptions via the `com.apple.configuration.management.status-subscriptions` declaration to receive specific paths automatically.

## Activation predicates

`com.apple.activation.simple` activates referenced declarations unconditionally. More interesting variants gate activation on predicates (device characteristics, network, time, etc.) so a single fleet can have device-class-specific configuration.

Example: a "Conference fleet" activation that activates the conference Wi-Fi declaration only when the device's serial number is in a list.

## When to use DDM vs classic profiles

Use DDM when:

- The capability has a DDM declaration available (check the schema directory)
- You want continuous reconciliation, not "fire and forget"
- You need rich status reports
- You're starting fresh (it's easier to design DDM-first than to migrate later)

Use classic profiles when:

- The payload has no DDM equivalent yet (most still don't)
- The capability is supervision-specific in a way DDM doesn't yet support
- You need the explicit user prompts of an OTA profile install (DDM operates silently)

Most production fleets are **hybrid**: classic profiles for the things that don't have DDM yet, DDM for what does. Apple has been steadily moving capabilities into DDM; expect this balance to shift each release.

## Common configurations (the high-value ones)

| Configuration type | What it does |
|---|---|
| `com.apple.configuration.passcode.settings` | Passcode policy (replaces `Restrictions` payload's passcode keys) |
| `com.apple.configuration.softwareupdate.enforcement.specific` | Force OS to a specific build, deadlines, defer reasons |
| `com.apple.configuration.management.organization-info` | Org branding shown in About screen |
| `com.apple.configuration.account.mail` | Mail account (DDM equivalent of `com.apple.mail.managed` payload) |
| `com.apple.configuration.account.exchange` | Exchange account |
| `com.apple.configuration.account.caldav` | CalDAV account |
| `com.apple.configuration.account.carddav` | CardDAV account |
| `com.apple.configuration.account.subscribedcalendar` | Subscribed calendar |
| `com.apple.configuration.account.ldap` | LDAP account |
| `com.apple.configuration.management.test` | No-op for testing DDM activation/reporting |
| `com.apple.configuration.diskmanagement.settings` | macOS external disk control |
| `com.apple.configuration.legacy` | Wraps a classic profile as a DDM declaration (bridge during migration) |

Full list: `https://github.com/apple/device-management/tree/release/declarative/declarations/configurations`

## App management in DDM

`deploying-apps-with-declarative-management` page describes how to deploy managed apps declaratively. The relevant declaration is `com.apple.configuration.app.managed` referencing a `com.apple.asset.app.managed` asset. This is the path forward for app deployment; classic `InstallApplication` still works and is still common.

See `apps-vpp.md` § Declarative app deployment.

## Gotchas

- **DDM activation is a one-way door per enrollment**: once you've sent `DeclarativeManagement` to a device, it expects DDM to keep working. If your server's DDM endpoints break, the device's state can drift unpredictably.
- **The `Data` payload in the initial `DeclarativeManagement` command** is a base64-encoded plist of initial state. Get it wrong and DDM never activates cleanly. Easiest path: send the minimal valid form (`{ "EnrollmentURL": "...", "Endpoint": "..." }`) and let the device fetch the actual manifest.
- **Server tokens must change when declarations change.** If you mutate a declaration but reuse the token, devices won't re-fetch and you'll wonder why nothing happens. Bump the token on every meaningful change.
- **Status reports can be large.** Subscribe selectively via `status-subscriptions` rather than asking for everything.
- **DDM and classic profile of the same thing**: if both target the same setting, the result is platform-dependent and historically not well-documented. Avoid overlap.
- **The bridge declaration `com.apple.configuration.legacy`** wraps a classic profile in a DDM declaration. Useful during migration but doesn't "translate" — it just delivers the same profile under DDM lifecycle management.

## Quick links

- Declaration schemas (root): `https://github.com/apple/device-management/tree/release/declarative/declarations`
- Status item schemas: `https://github.com/apple/device-management/tree/release/declarative/status`
- DDM protocol schemas: `https://github.com/apple/device-management/tree/release/declarative/protocol`
- `DeclarativeManagement` check-in: `https://github.com/apple/device-management/blob/release/mdm/checkin/declarative_management.yaml`
