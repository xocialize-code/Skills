# Deployment services (ABM/ASM API)

The umbrella term "Deployment Services" covers the three REST APIs that Apple Business Manager and Apple School Manager expose to your MDM server. They are **separate from the MDM protocol itself** — your MDM talks to Apple's services to learn about devices, classes, and app/book purchases; the MDM-to-device channel is a different protocol entirely.

Apple doc — Device Assignment: `https://developer.apple.com/documentation/devicemanagement/device-assignment`
Apple doc — Roster Management: `https://developer.apple.com/documentation/devicemanagement/roster-management`
Apple doc — App and Book Management: `https://developer.apple.com/documentation/devicemanagement/app-and-book-management`
Schemas: `https://github.com/apple/device-management/tree/release/other`

## The three services at a glance

| Service | What it does | When you need it |
|---|---|---|
| **Device Assignment** | Learn which devices ABM/ASM has assigned to your MDM; define and assign enrollment profiles; disown devices | Every MDM that supports ADE |
| **Roster Management** | Read student/teacher/class rosters; education only | School deployments using Classroom / Schoolwork |
| **App and Book Management** | Buy apps and books in bulk; assign VPP licenses to devices or users | Any MDM that needs to install paid App Store apps |

Each service has its **own service token** issued separately in ABM/ASM. A single MDM server typically holds three distinct tokens (or two — Roster Management is education-only).

## Auth and tokens — keep these straight

The token story is the most-confused part of Apple's deployment APIs. Three distinct tokens:

| Token | Lives | Used for | How to rotate |
|---|---|---|---|
| **MDM Server token (Device Assignment)** | `.p7m` you download from ABM/ASM | OAuth-equivalent for `mdmenrollment.apple.com` | Re-download from ABM/ASM, decrypt with your private key |
| **Apps and Books token (App/Book Management)** | Separate `.vpptoken` file | Auth for App and Book Management API | Re-download from ABM/ASM |
| **Roster Management token** | Yet another file | Auth for Roster Management API | Re-download from ABM/ASM |

Tokens are encrypted on Apple's side using the public key you uploaded to ABM/ASM. You decrypt with the corresponding private key, get a token, exchange it at the service's `/session` endpoint for a session token, use that session token in subsequent calls.

**Critical: the private key you upload to ABM/ASM is the master credential.** Losing it means re-generating a key pair, re-uploading, re-downloading all tokens. The old token files won't decrypt without the old key. Protect that key like a CA root key.

## Device Assignment service

Base URL: `https://mdmenrollment.apple.com/`

### Auth dance

1. POST `/session` with the decrypted server token. Response includes `auth_session_token`.
2. Use `X-ADM-Auth-Session: [auth_session_token]` on all subsequent calls.
3. Session tokens last ~24 hours. Re-do the dance when they expire.

### Endpoints

| Endpoint | Purpose |
|---|---|
| `POST /session` | Exchange server token for session token |
| `GET /server/devices` | Initial fetch of assigned devices, paginated |
| `POST /devices/sync` | Subsequent paginated fetch using cursor |
| `GET /devices` | Fetch specific device records by serial |
| `POST /devices/disown` | Release devices from your MDM's assignment |
| `POST /profile` | Define an enrollment profile |
| `GET /profile` | Fetch a defined profile by UUID |
| `PUT /profile/devices` | Assign profile to devices |
| `DELETE /profile/devices` | Un-assign profile from devices |
| `GET /account` | Get current account info |

Most endpoints return JSON. Errors include `code` and `message` fields; common codes are `AUTH_SESSION_TOKEN_EXPIRED`, `INVALID_CURSOR`, `MALFORMED_REQUEST_BODY`, `FORBIDDEN`.

Schemas under `https://github.com/apple/device-management/tree/release/other/device_assignment`.

See `enrollment-ade.md` for the full enrollment flow and profile structure.

## Roster Management service

Base URL: `https://mdmenrollment.apple.com/roster/` (education only — your account must be ASM).

### What's in a roster

- Students, teachers, and staff
- Classes (groups of students + teacher + course)
- Class membership and rosters
- Locations / sites
- Managed Apple IDs associated with each person

### Endpoints

| Endpoint | Purpose |
|---|---|
| `POST /roster/session` | Exchange roster token for session token |
| `GET /roster/students` | List students, paginated |
| `GET /roster/classes` | List classes |
| `GET /roster/teachers` | List teachers |
| `GET /roster/staff` | List staff |
| `GET /roster/locations` | List sites/locations |

Schemas under `https://github.com/apple/device-management/tree/release/other/roster_management`.

Roster data is typically synced into your MDM's directory so that Classroom and Schoolwork apps work. Skip this entirely if you're not doing education.

## App and Book Management service

Base URL: changes per service config — always fetch the current base URL from the `/config` endpoint rather than hard-coding.

### Auth

- POST to `/config` to get current service config (base URLs, supported features).
- POST to `/client/config` with the decrypted Apps-and-Books token to receive a session token.
- Use session token for subsequent calls.

### Endpoints (capabilities)

| Capability | What it gives you |
|---|---|
| `assets` | List all apps/books you've purchased, with remaining licenses |
| `assignments` | Assign licenses to devices or users, with async job tracking |
| `users` | List/manage Apple IDs that can be assigned user-based licenses |
| `clients` | The serial/userid of each device or user license holder |
| `service` | Service health |

License assignment is an **async job**: POST creates a job, you poll a job-status endpoint for completion. Failed jobs report which license assignments failed.

Schemas under `https://github.com/apple/device-management/tree/release/other/app_and_book_management`.

See `apps-vpp.md` for the license-to-install flow.

## Token lifecycle

| Event | Action |
|---|---|
| First-time setup | Generate key pair, upload public key to ABM/ASM, download tokens, encrypt store on your side |
| Token approaching expiry (annually) | ABM/ASM admin re-downloads tokens; replace your stored versions |
| Suspect key compromise | Generate new key pair, upload new public key, re-download tokens, **revoke old key in ABM/ASM** |
| MDM service migration | Old tokens stop working at the boundary; new MDM gets its own keys/tokens |

Apple sends email reminders when tokens are nearing expiration. Don't ignore them — an expired token means your MDM can no longer fetch device assignments, install paid apps, etc.

## Rate limits and pagination

- All services paginate. Always honor the cursor and follow until exhausted on initial sync.
- Subsequent syncs use the previously-stored cursor for delta semantics — only changed records come back.
- Rate limits are not formally published; in practice, sane polling (every few minutes for sync, on-demand for assignment ops) is fine. Don't tight-loop.
- Long-running async jobs (license assignment, profile assignment) — poll status, don't re-submit.

## Gotchas

- **Three different tokens, three different services.** Confusing these is the most common mistake. Label your stored token files clearly.
- **`POST /devices/disown` is irreversible** without admin intervention in ABM/ASM. Don't expose this to "delete device" UI without strong confirmation.
- **Session tokens are bearer credentials** — anyone with one has 24h of access. Keep them in memory only, never log them.
- **Profile assignment is async** — `PUT /profile/devices` returns immediately, but the device-side effect (next ADE attempt sees the new profile) lags by minutes typically.
- **The session token expiration error** (`AUTH_SESSION_TOKEN_EXPIRED`) is the only one that means "re-auth and retry". Other errors usually mean "fix your request."
- **Anchor cert PEM strings in `POST /profile`** must be base64-encoded. Double-base64 (PEM is already base64) is a common newcomer mistake.
- **`is_mdm_removable: false` in a profile assignment** is permanent from the device side. Treat the choice carefully.

## Quick links

- All deployment service schemas: `https://github.com/apple/device-management/tree/release/other`
- Device Assignment guide (admin): `https://support.apple.com/guide/apple-business-manager/intro-axmd344cdd9d/web`
- ABM/ASM portal: `https://business.apple.com/` / `https://school.apple.com/`
