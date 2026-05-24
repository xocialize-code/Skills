# Configuration profiles

A configuration profile is an XML property list containing one or more **payloads** — each payload configures a specific area of the device (Wi-Fi, VPN, certificate trust, app restrictions, etc.). Profiles are the classic delivery mechanism for device configuration; DDM is gradually replacing some of these, but profiles remain the dominant tool in practice.

Apple doc — overview: `https://developer.apple.com/documentation/devicemanagement/configuring-multiple-devices-using-profiles`
Apple doc — payload key reference: `https://developer.apple.com/documentation/devicemanagement/profile-specific-payload-keys`
Schemas: `https://github.com/apple/device-management/tree/release/mdm/profiles`

## Top-level structure

```xml
<plist>
<dict>
    <key>PayloadType</key>          <string>Configuration</string>
    <key>PayloadVersion</key>       <integer>1</integer>
    <key>PayloadIdentifier</key>    <string>com.example.fleet.wifi-corp</string>
    <key>PayloadUUID</key>          <string>00112233-...</string>
    <key>PayloadDisplayName</key>   <string>Corp Wi-Fi</string>
    <key>PayloadDescription</key>   <string>Connects to corp Wi-Fi</string>
    <key>PayloadOrganization</key>  <string>MVS Collective</string>
    <key>PayloadScope</key>         <string>System</string>     <!-- or User -->
    <key>PayloadRemovalDisallowed</key> <true/>
    <key>PayloadContent</key>
    <array>
        <dict>
            <key>PayloadType</key>       <string>com.apple.wifi.managed</string>
            <key>PayloadVersion</key>    <integer>1</integer>
            <key>PayloadIdentifier</key> <string>com.example.fleet.wifi-corp.wifi</string>
            <key>PayloadUUID</key>       <string>...</string>
            <!-- payload-specific keys -->
        </dict>
    </array>
</dict>
</plist>
```

The outer dict is a "profile" (`PayloadType: Configuration`); inner dicts in `PayloadContent` are the individual payloads.

## Top-level keys (the wrapper)

| Key | Required | Notes |
|---|---|---|
| `PayloadType` | yes | Always `"Configuration"` for the wrapper |
| `PayloadVersion` | yes | `1` for the wrapper |
| `PayloadIdentifier` | yes | Reverse-DNS; **reinstall semantics**: same `PayloadIdentifier` = update, not duplicate |
| `PayloadUUID` | yes | Fresh UUID per generation; doesn't need to be stable across versions |
| `PayloadDisplayName` | recommended | Shown in Settings → General → VPN & Device Management |
| `PayloadDescription` | recommended | Longer description shown to user |
| `PayloadOrganization` | recommended | Organization name |
| `PayloadScope` | macOS only | `System` (whole machine) or `User` (specific user). iOS profiles are always system. |
| `PayloadRemovalDisallowed` | optional | Profile can't be removed by user. Only honored on supervised devices for most types. |
| `RemovalDate` | optional | Profile self-removes at this date |
| `DurationUntilRemoval` | optional | Profile self-removes after N seconds |
| `ConsentText` | optional | Per-language consent text shown at install |

## Inner payload keys (each payload in PayloadContent)

Each payload dict has the same `PayloadType` / `PayloadVersion` / `PayloadIdentifier` / `PayloadUUID` quartet, then payload-specific keys.

`PayloadIdentifier` for inner payloads is conventionally `[outer.PayloadIdentifier].[role]` — e.g., outer `com.example.fleet.wifi-corp`, inner Wi-Fi `com.example.fleet.wifi-corp.wifi`, inner cert `com.example.fleet.wifi-corp.cert`.

## The payload catalog — finding the right type

The catalog is large (100+ types) and grows each OS release. Use the GitHub schema directory as the authoritative index:

`https://github.com/apple/device-management/tree/release/mdm/profiles`

Files are named with the exact `PayloadType` string — e.g., `com.apple.wifi.managed.yaml`, `com.apple.vpn.managed.yaml`, `com.apple.security.scep.yaml`.

A few high-frequency payload types worth memorizing:

| PayloadType | Configures |
|---|---|
| `com.apple.wifi.managed` | Wi-Fi networks |
| `com.apple.vpn.managed` | VPN config (per-app VPN, IKEv2, etc.) |
| `com.apple.security.scep` | SCEP-issued identity certificate |
| `com.apple.security.pem` | Trusted root cert (PEM) |
| `com.apple.security.pkcs1` | Trusted root cert (DER) |
| `com.apple.security.pkcs12` | Identity cert with private key (.p12) |
| `com.apple.security.acme` | ACME-DM identity (iOS/macOS 16+) |
| `com.apple.MDM` | MDM enrollment payload |
| `com.apple.applicationaccess` | Restrictions / app access |
| `com.apple.dock` (macOS) | Dock items and behavior |
| `com.apple.notificationsettings` | Per-app notification restrictions |
| `com.apple.systemextensions` (macOS) | Permitted system extensions |
| `com.apple.TCC.configuration-profile-policy` (macOS) | TCC / privacy preferences (PPPC) |
| `com.apple.app.lock` | Single App Mode |
| `com.apple.mail.managed` | Mail account |
| `com.apple.eas.account` | Exchange ActiveSync account |
| `com.apple.calendar.managed` | CalDAV account |
| `com.apple.contacts.managed` | CardDAV account |
| `com.apple.subscribedcalendar.managed` | Subscribed calendar |
| `com.apple.webContentFilter` | Web content filter |

For specifics on any of these, fetch the YAML schema; the field types and per-OS availability are right there.

## Signing & encryption

Profiles can be sent unsigned, signed, or signed+encrypted.

| Form | Format | When |
|---|---|---|
| Unsigned | Plain XML plist | Development; UI-installed where user trusts source |
| Signed | CMS / PKCS#7 SignedData wrapping the plist | Production; user sees "Verified" badge |
| Signed + encrypted | CMS EnvelopedData wrapping SignedData | When payload contains secrets you don't want stored cleartext on the wire to a known device |

For MDM-delivered profiles, signing is usually optional (the MDM channel is already authenticated), but signing avoids problems if the profile is ever exported or re-installed. **For OTA-installed profiles** (user manually downloads `.mobileconfig`), signing is strongly recommended — otherwise the device shows a "Not Verified" warning.

To sign: CMS SignedData with detached or attached signature, signer = a code-signing or org cert. To encrypt: per-recipient EnvelopedData using the device's identity cert as recipient (so only that device can decrypt).

## Delivery modes

| Mode | How |
|---|---|
| Via MDM command | `InstallProfile` command with `Payload` = the profile data (base64 in plist) |
| Via OTA / direct download | Device downloads `.mobileconfig`, user taps Install, walks through prompts |
| Via Apple Configurator | Mac admin tool, USB tether, push profile to iOS device |
| As part of ADE | Profiles queued during `AwaitingConfiguration: true` window get auto-installed before user sees Home Screen |

## Trust tiers (what a profile can do)

| Tier | What's possible |
|---|---|
| User-installed (unsupervised) | Wi-Fi, VPN, mail, calendar, certs, app config. **Not** managed apps, **not** TCC (most), **not** lock screen control. User can remove the profile any time. |
| Standard MDM (unsupervised) | All of above + managed apps + most restrictions. User can still remove MDM and all profiles drop. |
| Supervised (ADE or Apple Configurator) | Full surface — single app mode, TCC, hidden apps, Activation Lock bypass, etc. MDM can be made non-removable. |
| User Enrollment | Limited — only configures the work partition. No device-wide control. No UDID. |

For each payload, the YAML schema's availability block lists `RequiresSupervision: true/false` per OS.

## Reinstall semantics

Reinstalling a profile with the **same `PayloadIdentifier`** updates that profile in place. The device replaces the previous version. Inner payloads also match by `PayloadIdentifier` — adding/removing/modifying inner payloads on the same outer ID works as expected.

Reinstall with a **different `PayloadIdentifier`** results in two parallel profiles. Conflicts (e.g., two Wi-Fi profiles for the same SSID) are resolved by Apple's documented payload-merge rules; usually "most restrictive wins" or "last-installed wins" depending on the payload type.

`PayloadUUID` does **not** affect reinstall semantics. Generate fresh per export.

## Gotchas

- **PPPC / TCC profiles must be signed** with a developer ID-issued cert or they're ignored. macOS only.
- **System extension allowlists** require both the signed profile and supervision; an unsupervised Mac will not honor the allowlist.
- **`PayloadDisplayName` is shown to users**; don't put internal IDs there. Make it human-readable.
- **Encrypted profiles need the device's cert at the time of generation.** If you regenerate the device cert (re-enroll), encrypted profiles encrypted to the old cert become un-installable.
- **`com.apple.MDM` payload** has special handling: it cannot be inside a profile delivered by a command unless the device is in an enrollment flow. It's primarily delivered via OTA / ADE.
- **iOS 16+ uses a `RestrictionsProfile` model** for some restrictions that no longer take effect via `com.apple.applicationaccess`. Check current restrictions schema.

## Quick links

- All profile schemas: `https://github.com/apple/device-management/tree/release/mdm/profiles`
- Apple's payload key reference (HTML): `https://developer.apple.com/documentation/devicemanagement/profile-specific-payload-keys`
- MDM payload (the enrollment payload itself): `https://github.com/apple/device-management/blob/release/mdm/profiles/com.apple.mdm.yaml`
- `InstallProfile` command: `https://github.com/apple/device-management/blob/release/mdm/commands/install_profile.yaml`
