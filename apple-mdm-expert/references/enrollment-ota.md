# OTA profile-based enrollment

OTA (Over-the-Air) profile-based enrollment is the **legacy** enrollment path: a user downloads a `.mobileconfig` from a web page or email, opens it, and walks through Settings prompts to install. The MDM payload inside the profile enrolls the device into MDM.

Use OTA when ABM/ASM is unavailable, when you're piloting an MDM before purchasing through ABM, or when enrolling personal devices in a "lightweight" model without a Managed Apple ID.

## When to use OTA vs other flows

| Flow | When to use |
|---|---|
| ADE | Org-owned devices purchased through ABM/ASM. Zero-touch, full control. |
| Account-Driven (ADUE / ADDE) | BYOD or org-owned without ABM. Identity comes from Managed Apple ID. |
| OTA | Pre-existing devices, no ABM, no Managed Apple ID. User trusts and installs profile manually. |
| Apple Configurator | Initial provisioning over USB, especially for converting unsupervised → supervised. |

## The flow

1. User visits an enrollment web page on the MDM (or scans a QR code, or opens a link from email).
2. Server returns a signed `.mobileconfig` containing the MDM enrollment payload.
3. Device prompts user: "This website is trying to download a configuration profile. Do you want to allow this?"
4. User taps Allow. Profile lands in Settings → General → VPN & Device Management (iOS) / System Settings → Privacy & Security → Profiles (macOS).
5. User opens the profile and taps Install. Settings shows the payloads, asks for the device passcode, prompts for any sensitive permissions.
6. Device installs the MDM payload, fetches identity cert via SCEP/ACME (which is in the same profile), starts the check-in flow.
7. `Authenticate` arrives. Enrollment complete.

There's no `AwaitingConfiguration` pause — the user is on the Home Screen by the time MDM starts. Queue your initial configuration immediately after `TokenUpdate`.

## What the enrollment page returns

A `.mobileconfig` (XML plist) containing the same payloads as the ADE flow's enrollment profile:

- SCEP or ACME payload for identity issuance
- (Optional) PEM/PKCS1 payloads for trust anchors
- The `com.apple.mdm` payload

The profile should be **signed** with a trusted code-signing cert. An unsigned profile shows a red "Unverified" warning that scares users (and sysadmins).

Best practice: serve the profile from a TLS-secured page on a domain users recognize, signed with a cert from a public CA (one that's in the device's trust store, or one you've shipped via a separate profile beforehand).

## Pre-enrollment user authentication

You probably don't want a public URL where anyone can download a profile that enrolls them into your MDM. Common patterns:

- **Web auth page** — user signs into your SSO portal, then a downloads-page generates a per-user single-use profile URL.
- **Token in URL** — single-use signed token (JWT or similar) in the profile URL, validated server-side, expires after first use or a short window.
- **Per-user identity in the profile** — embed user ID into a custom field in the MDM payload's `Topic` payload identifier or in a side-channel database keyed by the device's pre-issued cert.

There's no protocol-level user identity in OTA — you must implement it at the web layer.

## The MDM payload — key fields

```
PayloadType:        "com.apple.mdm"
PayloadIdentifier:  "com.example.fleet.mdm"
PayloadUUID:        ...
ServerURL:          "https://mdm.example.com/cmd"
CheckInURL:         "https://mdm.example.com/checkin"   // optional, defaults to ServerURL
Topic:              "com.apple.mgmt.External.UUID"      // your MDM push cert UID
IdentityCertificateUUID: <UUID of the SCEP/ACME payload in same profile>
SignMessage:        true                                 // device signs all check-in bodies
CheckOutWhenRemoved: true                                // send CheckOut if user removes
ServerCapabilities: ["com.apple.mdm.per-user-connections"]
AccessRights:       <bitmask>
UseDevelopmentAPNS: false
```

Full schema: `https://github.com/apple/device-management/blob/release/mdm/profiles/com.apple.mdm.yaml`

`AccessRights` is a bitmask defining what the MDM can do. Setting it to `8191` grants all rights (sometimes shortcut to `(1 << 32) - 1`). In practice, request only what you need:

| Bit | Right |
|---|---|
| 1 | Inspect installed profiles |
| 2 | Install/remove profiles |
| 4 | Inspect provisioning profiles |
| 8 | Install/remove provisioning profiles |
| 16 | Inspect installed apps |
| 32 | Restrictions information |
| 64 | Security info (FileVault, etc.) |
| 128 | Device lock and passcode reset |
| 256 | Device erase |
| 512 | Network info |
| 1024 | (reserved) |
| 2048 | Add/remove managed apps |
| 4096 | Settings command |

(Bit values from older docs; verify against the schema for current OS.)

## Apple Configurator path

For converting an unsupervised device to supervised, or initial provisioning of a wiped device that's not in ABM, Apple Configurator on Mac talks USB to the device, can wipe and supervise, and can install a profile-based enrollment as part of the process. Useful for one-off shop deployments.

## Gotchas

- **Unsigned profiles work but scare users.** Sign with a real cert.
- **Order of payloads matters.** SCEP/ACME first (provides identity), then trust anchors, then MDM (which references the identity cert UUID). Out-of-order profiles can fail to install.
- **`IdentityCertificateUUID` must match** the `PayloadUUID` of the SCEP/ACME payload in the same profile. Typo here → enrollment fails silently.
- **The profile download page** must respect the `application/x-apple-aspen-config` content type (or fall back to `Content-Disposition: attachment; filename=*.mobileconfig`). Some browsers / mail clients mis-handle without the right hints.
- **iOS 16+ Stolen Device Protection delay**: on a device with Stolen Device Protection enabled, profile installs from unfamiliar locations may be delayed up to an hour for security. There's no MDM-side workaround; this is intentional protection.
- **macOS user-approved MDM**: post-Catalina, the user must click "Approve" on the profile install for certain payloads to take effect. Apple Configurator or PPPC supervised contexts bypass this.

## Quick links

- MDM payload schema: `https://github.com/apple/device-management/blob/release/mdm/profiles/com.apple.mdm.yaml`
- Apple Configurator: `https://support.apple.com/apple-configurator`
- Configuring devices via profiles overview: `https://developer.apple.com/documentation/devicemanagement/configuring-multiple-devices-using-profiles`
