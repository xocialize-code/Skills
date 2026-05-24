# Commands and Queries

Commands are server-to-device requests delivered via the **command URL** in the MDM payload. The device polls this URL (server URL) after an APNs wake, the server hands back the next queued command, and the device returns to report status.

Apple doc: `https://developer.apple.com/documentation/devicemanagement/commands-and-queries`
Schemas: `https://github.com/apple/device-management/tree/release/mdm/commands`

## Anatomy of a command

A command is a property list:

```xml
<dict>
    <key>CommandUUID</key>
    <string>00112233-4455-6677-8899-AABBCCDDEEFF</string>
    <key>Command</key>
    <dict>
        <key>RequestType</key>
        <string>DeviceInformation</string>
        <key>Queries</key>
        <array>
            <string>DeviceName</string>
            <string>OSVersion</string>
        </array>
    </dict>
</dict>
```

Two top-level keys: `CommandUUID` (you generate; the response will echo it back) and `Command` (the actual request, with `RequestType` discriminating).

## Anatomy of a command response

The device posts back to the command URL:

```xml
<dict>
    <key>CommandUUID</key>
    <string>00112233-4455-6677-8899-AABBCCDDEEFF</string>
    <key>Status</key>
    <string>Acknowledged</string>
    <key>UDID</key>
    <string>...</string>
    <!-- response-specific keys depending on Status and original RequestType -->
</dict>
```

Match the response back to the original by `CommandUUID`.

## Status values — what they mean

| Status | Meaning | What to do |
|---|---|---|
| `Idle` | Device is polling but has no pending status to report. Send next queued command or empty 200. | Pop next command from queue or close. |
| `Acknowledged` | Command completed successfully. Response may include data (e.g., DeviceInformation queries). | Record completion; persist any returned data. |
| `Error` | Command failed. Body includes `ErrorChain` array with one or more error dicts. | See `errors.md`. Don't retry blindly — many errors are permanent. |
| `CommandFormatError` | The plist you sent was structurally invalid. | Bug in your code. Inspect the command body for missing keys, wrong types. |
| `NotNow` | Device is busy / user dialog up / unable to execute right now. **Not a failure.** | Device will poll again. Leave command queued or re-queue. |

`NotNow` is the easiest trap. Treat it as deferred, not failed. The device decides when to retry; you usually don't need to do anything.

## Top-level command categories

Apple groups commands roughly into:

| Category | Examples | Schema dir |
|---|---|---|
| Device queries | `DeviceInformation`, `SecurityInfo`, `CertificateList`, `InstalledApplicationList`, `ProfileList`, `RestrictionsList`, `ProvisioningProfileList` | `mdm/commands/` |
| Profile lifecycle | `InstallProfile`, `RemoveProfile`, `ProfileList` | `mdm/commands/` |
| App lifecycle | `InstallApplication`, `RemoveApplication`, `ManagedApplicationList`, `ManagedApplicationFeedback`, `ManagedApplicationAttributes` | `mdm/commands/` |
| Lock / wipe / restart | `DeviceLock`, `EraseDevice`, `RestartDevice`, `ShutDownDevice` | `mdm/commands/` |
| Setup Assistant control | `DeviceConfigured`, `Settings` (with `MDMOptions` for skipping panes) | `mdm/commands/` |
| OS updates | `ScheduleOSUpdate`, `ScheduleOSUpdateScan`, `AvailableOSUpdates`, `OSUpdateStatus` | `mdm/commands/` |
| Identity / keys | `RotateFileVaultKey`, `RefreshCellularPlans` | `mdm/commands/` |
| Diagnostics | `ManagedMediaList`, `LogOutUser` | `mdm/commands/` |
| Education (iPad) | `Settings`, `ClassroomList`, `ShareAppleTV` | `mdm/commands/` |
| Lights-Out Management (Mac mini, Mac Studio, Mac Pro) | `LOMSetupRequest`, `LOMDeviceRequest` | `mdm/commands/` |

For an exhaustive command list, the GitHub directory is the source. File names are snake_case of the RequestType — `install_application.yaml`, `device_lock.yaml`, etc.

## Common queries — DeviceInformation

`DeviceInformation` is the swiss-army knife: one command, takes a `Queries` array, returns a dict of the requested keys. Common queries:

| Query | Returns |
|---|---|
| `UDID` | Device UDID (or EnrollmentID equivalent) |
| `DeviceName`, `Model`, `ModelName`, `SerialNumber`, `IMEI`, `MEID` | Hardware identity |
| `OSVersion`, `BuildVersion`, `ProductName` | OS identity |
| `BatteryLevel` | iOS only |
| `WiFiMAC`, `BluetoothMAC`, `EthernetMACs`, `IsNetworkTethered` | Network info |
| `AvailableDeviceCapacity`, `DeviceCapacity`, `IsMultiUser`, `IsSupervised`, `IsDEPEnrollment` | Storage / posture |
| `JailbreakDetected` | iOS only |
| `ActiveManagedUsers` (macOS) | Logged-in managed users |

Full list: `https://github.com/apple/device-management/blob/release/mdm/commands/device_information.yaml`

## Async patterns

Most commands are async by nature: you queue, you push, the device polls, you respond with the command, the device executes, the device polls again with the result, you respond with the next command or an empty 200.

Two patterns work well for tracking:

1. **Per-command record**: insert a row keyed on `CommandUUID` with status `pending`. When the device returns with that UUID, update to `acknowledged | error | not_now | format_error`. Easy to query, easy to retry.
2. **Per-device queue**: maintain an ordered queue of commands per device. When the device polls, pop and send the head. When the device reports back, the head is what's being reported on. This is simpler but harder to handle out-of-order or commands that interleave with status reports.

For FleetLock's Durable Object design, see `fleetlock-stack.md` § DO patterns.

## DeviceLock and EraseDevice — please read carefully

`DeviceLock` and `EraseDevice` are irreversible from the user's perspective. Always confirm with the user before sending. `EraseDevice` on a supervised device cannot be undone; on a personally-owned device with User Enrollment, only the work data is wiped (which is desirable).

Both take a `PIN` (six-digit numeric on iOS) and `Message` (display on lock screen). EraseDevice on macOS Apple Silicon has additional fields for `ReturnToService` (restore to ADE-pending state instead of erasing identity).

## DeviceConfigured — releasing Setup Assistant

When a device enrolls via ADE, it pauses Setup Assistant on the "Remote Management" screen and sets `AwaitingConfiguration: true` in TokenUpdate. The server has a window to queue profile installs, app installs, and DDM activation while the device is paused. When done, send `DeviceConfigured` with no extra parameters — the device proceeds through Setup Assistant.

Forgetting this leaves devices stuck. Confirm explicit success before assuming the device has moved past Setup Assistant.

## Settings command (with MDMOptions)

`Settings` is a multipurpose command that takes a `Settings` array of dicts; each dict has an `Item` discriminator plus item-specific fields. Items include `Wallpaper`, `ApplicationAttributes`, `Bluetooth`, `OrganizationInfo`, `MDMOptions`, etc.

`MDMOptions` is especially useful — it skips Setup Assistant panes during ADE:

```
{ "Item": "MDMOptions",
  "MDMOptions": { "BootstrapTokenAllowedForAuthentication": "always",
                  "BootstrapTokenRequired": true } }
```

Schema: `https://github.com/apple/device-management/blob/release/mdm/commands/settings.yaml`

## Gotchas

- **Don't send `EraseDevice` to test a device** unless you're prepared to lose it. There's no "are you sure" prompt.
- **`CommandUUID` is server-generated and should be unique** across all commands ever sent to the device. Don't reuse.
- **A device with no `TokenUpdate` yet is unreachable.** Don't queue commands until you've received `TokenUpdate`.
- **Some commands require supervision.** The YAML schema's `SupervisionRequired: true` is the source of truth.
- **Older OS versions reject newer commands silently** — `CommandFormatError` is sometimes "this OS doesn't know that command". Check `SupportedOS` in the schema before sending.
- **`InstalledApplicationList` can be huge.** On a Mac with hundreds of apps, the response can exceed expected sizes. Page if needed (`OnlyAppUpdates`, `Identifiers` filters).
