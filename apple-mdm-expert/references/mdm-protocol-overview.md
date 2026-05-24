# MDM protocol overview

A unified mental model for Apple MDM. Read this first if you're new to MDM or coming back after a while; it explains how the pieces fit so the specific cheat sheets make sense in context.

## The core loop in seven steps

1. **MDM server has a push certificate** issued by Apple via the MDM Push Certificates portal. Embedded `UID` in that certificate is the **Topic** (e.g. `com.apple.mgmt.External.abcd1234-...`). One certificate per "MDM service identity" — multiple servers can share if they share the cert.
2. **Device enrolls.** Via ADE, Account-Driven, or OTA profile install. The result is the device installing an `MDM` payload that points back to the server's check-in and command URLs, signed with a SCEP- or ACME-issued identity certificate.
3. **Device check-in: `Authenticate`** — first contact. Body includes `UDID`, `Topic`, `BuildVersion`, `OSVersion`, `ProductName`. Server stores the device.
4. **Device check-in: `TokenUpdate`** — device hands the server its current `PushMagic` and `Token` (the APNs device token for the MDM topic). These two together are what the server uses to wake the device later. `TokenUpdate` fires repeatedly across the device's lifetime; treat the latest values as authoritative.
5. **Server wants to send a command.** Server constructs the command plist, queues it for the device, then sends an APNs HTTP/2 push to `api.push.apple.com` with topic = MDM topic and `mdm` header = the device's `PushMagic`. No payload — APNs just wakes the device.
6. **Device polls the server's command URL.** Device sends an empty (or `Status: Idle`) request. Server responds with the queued command. Device executes asynchronously.
7. **Device reports back.** Posts to the command URL again with `Status: Acknowledged | Error | CommandFormatError | Idle | NotNow`, optionally including return data. Server correlates by `CommandUUID`, updates state, queues next command if any.

The whole protocol is essentially "server queues plist, server pings APNs, device polls server, device returns plist." Everything else — profiles, DDM, ADE, ABM, VPP — sits on top of this loop or feeds it from the side.

## What lives at each layer

```
┌─────────────────────────────────────────────────────────────────┐
│ ABM / ASM (Apple Business / School Manager) — admin portal       │
│ Deployment Services API — buy apps, assign devices, manage rosters│
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ (assigns devices to MDM server)
┌─────────────────────────────────────────────────────────────────┐
│ ADE (Automated Device Enrollment) — server fetches device records│
│ from Apple's Device Assignment Service via the MDM server token │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ (device boots, contacts MDM)
┌─────────────────────────────────────────────────────────────────┐
│ Enrollment — device installs MDM payload, gets identity cert    │
│ (SCEP / ACME), starts trusting server                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ MDM core loop — check-in + commands over APNs                   │
│   - Check-in: Authenticate, TokenUpdate, CheckOut, ...          │
│   - Commands: InstallProfile, InstallApplication, ...           │
│   - Profiles: classic .mobileconfig delivered via command       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ DDM (Declarative Device Management) — newer layer on top        │
│   - Declarations, status reports, async predicates              │
│   - Activated by sending `DeclarativeManagement` check-in       │
└─────────────────────────────────────────────────────────────────┘
```

## The push identity triad

Three identifiers are easily confused. Get them straight or nothing else makes sense.

| Identifier | Where it lives | What it identifies |
|---|---|---|
| **Topic** | MDM Push Certificate's `UID` (`com.apple.mgmt.External.UUID`) | The MDM service identity. Used as the APNs topic when pushing. Used in `Authenticate` so the device knows which cert to trust. |
| **PushMagic** | Device generates it, sent in `TokenUpdate` | Wakes a *specific* device. Goes in the APNs HTTP/2 `mdm` header. Per-device, per-enrollment. |
| **Token** | Device APNs device token, sent in `TokenUpdate` | The APNs target. Goes in the `:path` of the APNs HTTP/2 push (`/3/device/[token]`). Per-device, per-enrollment. |

To wake a device: APNs HTTP/2 POST to `https://api.push.apple.com/3/device/[Token]` with `apns-topic: [Topic]` and body containing `{"mdm": "[PushMagic]"}`.

## Async vs sync commands

**Almost everything is async.** The server queues a command, pushes the device, the device polls and executes, the device reports back later — possibly minutes or hours later if the device is asleep, offline, or a user dialog needs dismissing. Do not design control flow that blocks on a single command's response.

The exception is the **command URL response itself**: when the device polls, you respond synchronously with the next queued command. That response is HTTP-synchronous but the *execution* is async.

When the device responds with `NotNow`, the command is **not** done — the device is busy and will retry. Re-queue or wait; do not treat `NotNow` as a failure.

## What changed with DDM

Classic MDM is **imperative**: server says "install this profile", device does it, server hopes the device is in the desired state. DDM is **declarative**: server publishes a target state (declarations) and the device continuously reconciles itself, reporting status when things change. Concretely:

- DDM is delivered over the **same** check-in transport — initial activation is a regular `DeclarativeManagement` check-in command. The protocol underneath is different (synchronization tokens, declaration items, status reports) but it rides the same socket.
- DDM and classic MDM coexist. Some payloads have a DDM-equivalent declaration; some don't yet. Apple is steadily migrating capabilities.
- Status reports replace the "command-per-piece-of-info" pattern. The device proactively reports what's in its status channel.

See `ddm.md` for the protocol details.

## Trust and supervision tiers (affects what's possible)

| Tier | How achieved | What it unlocks |
|---|---|---|
| User-installed profile | Device user manually opens a `.mobileconfig` | Limited payload set; can be removed by user |
| User Enrollment | Managed Apple ID + Account-Driven flow | Work data is partitioned; less device control than full enrollment |
| Standard MDM enrollment | Profile-based OTA, user accepts | Most commands; user can remove MDM unless device is supervised |
| Supervised | ADE or Apple Configurator | Full command surface; MDM cannot be removed by user |
| Supervised + DEP + non-removable | ADE with `is_mdm_removable: false` | Permanent management |

When unsure what a command requires, the YAML schema in `apple/device-management` has a `SupervisionRequired` and per-OS availability block. Check there.

## Glossary (for quick orientation)

- **MDM** — Mobile Device Management protocol. The command/check-in protocol.
- **DDM** — Declarative Device Management. The newer declarative layer on top of MDM.
- **ADE** — Automated Device Enrollment. The new name for DEP.
- **DEP** — Device Enrollment Program. Old name for ADE; still seen in code and Apple's API.
- **ABM** — Apple Business Manager. Admin portal for orgs.
- **ASM** — Apple School Manager. Admin portal for education.
- **VPP** — Volume Purchase Program. The B2B / school app/book purchase system; now part of ABM/ASM.
- **APNs** — Apple Push Notification service. How the server wakes devices.
- **SCEP** — Simple Certificate Enrollment Protocol. Issues device identity certs at enrollment.
- **ACME / ACME-DM** — Newer device-attested ACME for MDM identity issuance.
- **Topic** — The MDM Push Certificate UID. APNs topic for MDM pushes.
- **PushMagic** — Per-device wake-up token. Goes in the APNs `mdm` body.
- **UDID** — Hardware identifier. Was the unique device key; now phased out on user-enrolled devices in favor of `EnrollmentID`.
- **EnrollmentID** — Identifier used for User Enrollment and Account-Driven flows. Replaces UDID for those tiers.
- **PayloadIdentifier** — Reverse-DNS unique key for a profile within an org. Reinstalling with same `PayloadIdentifier` updates rather than duplicates.
