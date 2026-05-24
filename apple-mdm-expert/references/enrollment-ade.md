# Automated Device Enrollment (ADE / DEP)

ADE — formerly DEP (Device Enrollment Program) — is the zero-touch enrollment flow for organization-owned devices purchased through Apple or an authorized reseller. The device, on first boot, contacts Apple, learns it belongs to your org, and enrolls into your MDM automatically before the user reaches the Home Screen.

Apple doc — Device Assignment service: `https://developer.apple.com/documentation/devicemanagement/device-assignment`
Apple doc — migrating managed devices: `https://developer.apple.com/documentation/devicemanagement/migrating-managed-devices`
Schemas: relevant pieces under `https://github.com/apple/device-management/tree/release/other` and `https://github.com/apple/device-management/tree/release/mdm/profiles` (`com.apple.mdm`)

## The full ADE flow

1. **Device purchased** through Apple Business Manager / Apple School Manager (ABM/ASM) account or via an authorized reseller that ships device serials to ABM/ASM.
2. **Admin assigns device to an MDM server token in ABM/ASM.** The MDM server token is the OAuth-equivalent credential your MDM uses to talk to the Device Assignment service.
3. **Device first-boots** and during Setup Assistant queries Apple for its assignment.
4. **Apple returns the URL of the assigned MDM**'s enrollment endpoint to the device.
5. **Device fetches the EnrollmentProfile** from your MDM (an unsigned plist with `com.apple.mdm` payload + supporting payloads).
6. **Device installs the MDM payload**, sets up identity certificate (SCEP/ACME), starts the check-in flow.
7. **`Authenticate` and then `TokenUpdate` arrive** with `AwaitingConfiguration: true`.
8. **Server queues setup commands** (initial profiles, apps, DDM activation).
9. **Server sends `DeviceConfigured`** to release Setup Assistant.

## The Device Assignment service

This is a REST API your MDM server calls to read device records from Apple's side. It's where you learn what devices have been assigned to you, fetch their hardware metadata, and acknowledge them.

Base URL: `https://mdmenrollment.apple.com/` (production)

Auth: OAuth-like dance using your **MDM server token** (downloaded from ABM/ASM as a `.p7m` file containing an encrypted blob). Steps:

1. Decrypt the server token with your MDM's private key (the token was encrypted to the public key you uploaded to ABM/ASM).
2. POST `/session` with the decrypted token to receive a short-lived `auth_session_token` (valid ~24h).
3. Use that token in `X-ADM-Auth-Session` header for all subsequent calls.

Key endpoints:

| Endpoint | Purpose |
|---|---|
| `POST /session` | Exchange server token for session token |
| `GET /server/devices` | Fetch device list (paginated, with a cursor) |
| `POST /devices/sync` | Subsequent fetches with cursor — only new/changed devices |
| `POST /devices/disown` | Remove device from ABM/ASM (use carefully) |
| `POST /profile` | Define an enrollment profile (Setup Assistant skip panes, supervision, naming, etc.) |
| `PUT /profile/devices` | Assign defined profile to specific devices |

Device records returned include serial, model, color, asset tag, profile assignment status, and last activity. **Persist them** — that's your inventory.

## The EnrollmentProfile (Device Assignment one)

The profile uploaded to Apple via `POST /profile` is JSON, **not** a plist. It contains:

```json
{
  "profile_name": "FleetLock Default",
  "url": "https://mdm.example.com/enroll",
  "is_supervised": true,
  "is_multi_user": false,
  "allow_pairing": false,
  "is_mandatory": true,
  "is_mdm_removable": false,
  "support_phone_number": "+1-555-...",
  "support_email_address": "it@example.com",
  "anchor_certs": ["BASE64-encoded-PEM-cert"],
  "skip_setup_items": ["AppleID", "Siri", "Privacy", "Diagnostics", "Biometric"],
  "configuration_web_url": "https://...",
  "language": "en",
  "region": "US",
  "auto_advance_setup": false,
  "devices": ["SERIAL1", "SERIAL2"]
}
```

`url` is where the device fetches the **MDM enrollment profile** (a plist) after deciding to enroll.

`anchor_certs` lets you pin a custom CA — useful if your enrollment endpoint uses an internal cert.

`is_mdm_removable: false` is the supervised "MDM cannot be unenrolled by user" lock. Use sparingly — locking yourself out of a device with no recourse is a real risk.

`is_mandatory: true` removes the "Skip" option from the Remote Management step. User must enroll.

## The MDM enrollment profile (the plist)

What the device actually fetches from your `url`. A plist containing:

| Payload | Purpose |
|---|---|
| `com.apple.security.scep` (or `com.apple.security.acme`) | Device identity certificate issuance |
| `com.apple.security.pem` / `pkcs1` | Trust anchors if your MDM uses an internal CA |
| `com.apple.mdm` | The MDM enrollment itself — `ServerURL`, `CheckInURL`, `Topic`, `IdentityCertificateUUID`, `SignMessage`, `CheckOutWhenRemoved`, etc. |
| Optional initial profiles | Anything you want installed before Setup Assistant proceeds |

The order matters: SCEP/ACME payload first (so the device has identity), then trust anchors, then MDM. The `com.apple.mdm` payload references the SCEP/ACME identity by `IdentityCertificateUUID`.

## Skip keys (Setup Assistant pane skipping)

`skip_setup_items` in the Device Assignment profile suppresses Setup Assistant panes. The complete list is in Apple Support's deployment guide (and the YAML schema's `Skip Keys` doc). Common ones:

| Key | Skips |
|---|---|
| `AppleID` | Apple ID sign-in |
| `Biometric` | Touch ID / Face ID setup |
| `Diagnostics` | Diagnostics opt-in |
| `Location` | Location services |
| `Passcode` | Passcode creation (use only if you'll enforce one via DDM/profile) |
| `Privacy` | Privacy notice |
| `Siri` | Siri setup |
| `TOS` | iOS Terms of Service |
| `WatchMigration` | Apple Watch migration |
| `Restore` | Restore from backup |
| `Welcome` | Hello screen |

Skip aggressively on shared/kiosk devices, lightly on user-assigned devices.

## Migration

Two real flows:

1. **Within ABM/ASM**: admin reassigns the device to a different MDM server token. The device, on next sync (or on `RefreshCellularPlans` / reboot), will pick up the new assignment.
2. **`DeviceManagementService` migration**: protocol-level migration via `migrate` command + new MDM accepting the migrated device. See Apple's "Migrating managed devices" doc — this is more complex and used when changing MDM vendors mid-life.

Migration retains device state (apps, data) — it's not a re-enroll. But identity certs typically rotate.

## Gotchas

- **Time skew**: ABM/ASM session tokens are time-sensitive (TLS plus the 24h validity). Keep server clock NTP-synced.
- **Device must hit Apple to learn its assignment.** A device taken to a no-internet location and first-booted there will not enroll automatically. It can be enrolled later by joining a network.
- **`is_mdm_removable: false` is irreversible from the device side.** Even erasing the device leaves it locked to your MDM (that's the point). Only the org's ABM/ASM admin can release the serial via `POST /devices/disown`.
- **Serial mismatch / "Unknown device"** errors from `/devices/sync` mean the device wasn't sold through your ABM/ASM channel. Check the reseller chain.
- **The MDM server token's encryption key must be the same one you uploaded to ABM/ASM.** Lose it and you need to generate a new key pair, re-upload, and re-download a new token. The old token won't decrypt without the old key.
- **Anchor certs in the Device Assignment profile** are PEM strings, base64-encoded. Apple is strict about format.
- **Token rotation**: ABM/ASM-issued MDM server tokens expire (currently 1 year). Rotate proactively.

## Quick links

- Device Assignment endpoints: `https://developer.apple.com/documentation/devicemanagement/device-assignment` and `https://github.com/apple/device-management/tree/release/other/device_assignment`
- MDM payload schema: `https://github.com/apple/device-management/blob/release/mdm/profiles/com.apple.mdm.yaml`
- Setup Assistant skip keys (admin guide): `https://support.apple.com/guide/deployment/skip-setup-assistant-mdm0f9c1c1c2/web`
