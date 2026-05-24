# Errors and debugging

When something goes wrong in MDM, errors come from several layers and look superficially similar. This page is a triage guide: identify the layer, find the right diagnostic, recover.

Apple doc — MDM errors (in command responses): `https://developer.apple.com/documentation/devicemanagement/commands-and-queries`
Schemas: `https://github.com/apple/device-management/tree/release/mdm/errors`

## The error surfaces

| Surface | What it tells you | Where to look |
|---|---|---|
| HTTP status from MDM server | Transport-layer (auth, format) | Your server logs |
| `Status: Error` in command response | Device executed the command, it failed | `ErrorChain` in the response |
| `Status: CommandFormatError` | Device couldn't parse your command plist | Your code — invalid command structure |
| `Status: NotNow` | Device deferred — **not a failure** | Just wait |
| DDM status report error array | DDM declaration failed validation or application | Status item paths in the report |
| APNs HTTP/2 response | Push delivery problems | APNs `:status` + `apns-id` + body |
| ABM/ASM API error JSON | Deployment service issue | Code/message in JSON body |

## Status: Error — the ErrorChain

When a device returns `Status: Error`, the response includes an `ErrorChain` array:

```xml
<key>Status</key>     <string>Error</string>
<key>CommandUUID</key> <string>...</string>
<key>ErrorChain</key>
<array>
    <dict>
        <key>ErrorCode</key>          <integer>4001</integer>
        <key>ErrorDomain</key>        <string>MCMDMErrorDomain</string>
        <key>LocalizedDescription</key> <string>...</string>
        <key>USEnglishDescription</key> <string>...</string>
    </dict>
    <dict>
        <key>ErrorCode</key>          <integer>...</integer>
        <key>ErrorDomain</key>        <string>...</string>
        ...
    </dict>
</array>
```

The chain reads top-down: the topmost dict is the most general / user-facing; lower dicts are the underlying cause. When debugging, **read from the bottom up** — the deepest error is usually the most informative.

Error domains you'll see:
- `MCMDMErrorDomain` — MDM-specific
- `MCInstallationErrorDomain` — profile installation
- `MCProfileErrorDomain` — profile parsing/validity
- `NSURLErrorDomain` — network
- `NSCocoaErrorDomain` — Foundation-layer

Apple's official error catalog lives at `https://github.com/apple/device-management/tree/release/mdm/errors`. Browse by domain.

## Common error codes (the ones you'll hit)

| Code | Domain | Meaning |
|---|---|---|
| 4001 | MCInstallationErrorDomain | Profile already installed or duplicate identifier |
| 4002 | MCInstallationErrorDomain | Profile signature invalid |
| 4011 | MCMDMErrorDomain | MDM service identity (Topic) mismatch |
| 8000 | MCMDMErrorDomain | Device not authorized for command (supervision required) |
| 12009 | MCMDMErrorDomain | Command's request type not supported on this OS |
| 12022 | MCMDMErrorDomain | Required field missing |
| 13017 | (varies) | InstallApplication failed at App Store |
| `MCMDMTopicMismatch` | MCMDMErrorDomain | The Topic in your command doesn't match the device's enrollment topic |

The numeric catalog is not exhaustively documented in human form anywhere — the authoritative source is the YAML schema directory.

## CommandFormatError — code-side bug

If you see `CommandFormatError`, the device couldn't parse your command. Causes:

- Missing `RequestType` or `CommandUUID`
- Wrong type for a field (passing a string where an integer is expected, etc.)
- Invalid base64 in a `Data` field
- Plist that isn't valid XML or binary plist

Diagnostic: print the command body before sending and validate against the YAML schema for that command. The schema's `Required` and `Type` fields are the contract.

## NotNow — defer, don't despair

`NotNow` means "I can't do that right now". Causes:

- User is in a modal dialog
- Device is in a state where the command can't execute (e.g., asleep with FileVault key not unwrapped)
- The action would interrupt something user-initiated

The device will poll again later and either acknowledge or return another NotNow. **Don't re-queue** — the command is still in flight. Just wait.

If you see persistent NotNow on a command, the device may be in a stuck state. Power-cycling sometimes clears it. For long-running issues, the device may need to be unenrolled and re-enrolled.

## Push diagnostics

If your APNs push doesn't reach the device:

| Symptom | Likely cause | Fix |
|---|---|---|
| APNs returns `400 BadDeviceToken` | Token is invalid or stale | Wait for next TokenUpdate; remove old token |
| APNs returns `400 BadTopic` | Topic doesn't match cert | Verify Topic = MDM push cert UID |
| APNs returns `403 InvalidProviderToken` | JWT auth token expired / wrong key | Regenerate JWT with valid APNs Auth Key |
| APNs returns `410 Unregistered` | Device unenrolled or token retired | Mark device unenrolled; remove token |
| APNs returns `200`, but device never polls | Network issue, device asleep, or PushMagic mismatch | Verify PushMagic in body; check device connectivity |

Apple's APNs HTTP/2 docs: search "APNs Provider API" on developer.apple.com — this is general iOS push infrastructure, not MDM-specific.

## DDM declaration rejected

DDM errors arrive in the status report's `Errors` array and as `valid: invalid` on the specific declaration's status item:

```json
{
  "StatusItems": {
    "management": {
      "declarations": {
        "configurations": [{
          "identifier": "com.example.fleet.passcode",
          "valid": "invalid",
          "active": false,
          "reasons": [{
            "code": "Error.ConfigurationCannotBeApplied",
            "description": "Passcode policy conflicts with existing classic profile",
            "details": { ... }
          }]
        }]
      }
    }
  },
  "Errors": [{ ... }]
}
```

Common DDM rejection causes:
- Conflict with a classic profile setting the same thing
- Asset referenced by a configuration is missing
- Activation predicate doesn't match (declaration sent but never activated)
- Required platform feature not available (e.g., trying to manage a feature on an OS version that doesn't support it)

Status reports are the *only* feedback channel for DDM — there's no command-response loop. Subscribe to relevant status paths via `com.apple.configuration.management.status-subscriptions`.

## OD error 5101 — sysadminctl authorizer rejected

Not an MDM-protocol error — this is an **opendirectoryd** error
that surfaces on macOS when `sysadminctl -secureTokenOn` (or related
commands needing an admin+ST authorizer) is rejected.

Message variants:
- "OpenDirectory Error 5101"
- "authentication server refused operation"
- Or silent — `sysadminctl` returns 0 and prints "Done!" while the
  underlying OD operation was rejected. **Always verify state
  independently after `sysadminctl` calls** —
  `sysadminctl -secureTokenStatus <user>` is the authoritative check.

Means the `-adminUser` you supplied is one of:

- Not in the `admin` group
- A Secure Token holder but not an admin
- An admin but not a Secure Token holder (e.g.,
  `AutoSetupAdminAccounts` admin before its first GUI login)
- Both

Diagnose:
```bash
dseditgroup -o checkmember -m <user> admin     # rc=0 = is admin
sysadminctl -secureTokenStatus <user>          # must show ENABLED
```

Recoveries:
- Transient promote a Standard primary:
  `dseditgroup -o edit -a primary admin && sysadminctl … &&
   dseditgroup -o edit -d primary admin`
- Get the intended authorizer ST first via the BT-mediated
  first-login grant (see
  `secure-token-bootstrap-apple-silicon.md`).
- If you're trying the Fabien Conus user-less self-grant on a
  modern macOS Apple Silicon + MDM device: **the loophole is
  closed**. The MDM Bootstrap Token + Personal Recovery User
  auto-register as APFS cryptographic users and the "zero ST
  holders" precondition is never satisfied. Pivot to creating
  the target user IN `AccountConfiguration` instead — see
  `account-configuration-user-patterns.md`.

## TokenUpdate stopped firing

A device that's enrolled but no longer sending TokenUpdate is heading for unreachability:

| Symptom | Cause | Recovery |
|---|---|---|
| No TokenUpdate for weeks | Device offline / asleep extensively | Send a push to wake; check on-device |
| TokenUpdate stopped after OS update | Push registration glitch | Usually self-resolves on next boot |
| TokenUpdate stopped after restore | Device was wiped without CheckOut | Mark unenrolled; await re-enrollment |
| TokenUpdate with new Token but old PushMagic | Mid-rotation; transient | Use the latest TokenUpdate; previous push may fail |

If a device hasn't checked in for a long period, send a wake push and watch for any check-in within ~24h. No response → device is gone (sold, wiped, stolen, dead). Reconcile against ABM/ASM device list.

## Lost-device states

Common scenarios and recovery:

| Scenario | What happened | Recovery |
|---|---|---|
| Device wiped by user (unsupervised) | MDM profile lost without CheckOut | Device reappears as "unknown"; depends on ABM/ASM assignment to re-enroll |
| Device wiped (supervised, in ABM) | ADE re-enrolls automatically on next boot/network | Wait; verify TokenUpdate arrives with same UDID |
| User Enrollment removed | Work data wiped; no CheckOut on some OS versions | Mark unenrolled; user can re-enroll via account |
| MDM cert expired without renewal | Device's identity cert chain broken | Re-enroll device |
| Push cert rotated without server update | All devices unreachable but enrolled | Update server's push cert; existing Topic continues to work |

## Diagnostic workflow

When something's not working, walk this list:

1. **Is the device enrolled?** Check for recent TokenUpdate and a valid identity cert.
2. **Can you push to it?** Send a quiet command (DeviceInformation with one query) and watch for poll. If APNs returns 200 but no poll, the device is probably offline.
3. **Is the command well-formed?** Validate against schema before sending.
4. **Did you read the entire ErrorChain?** Bottom-up.
5. **Is this command supported on this OS?** Check `SupportedOS` block in the YAML schema.
6. **Is supervision required?** Same place.
7. **For DDM**: check `Errors` and per-declaration `reasons` in status reports.

## Gotchas

- **A successful HTTP 200 from your server in response to a device poll doesn't mean the command worked** — it means the device received the command. Wait for the next poll to see the result.
- **ErrorChain depth varies.** Some errors are one-deep; some have four-plus layers. Don't assume position.
- **The `LocalizedDescription` is sometimes hilariously vague** (e.g., "An error occurred"). Always read the deeper layers.
- **APNs `410 Unregistered` doesn't mean the device unenrolled** — it can mean the token rotated. Wait for next TokenUpdate before marking the device gone.
- **DDM errors with no `Errors` array** mean status hasn't reported yet, not that the declaration succeeded. Wait for the next status report or actively query via `tokens` endpoint.

## Quick links

- All MDM error schemas: `https://github.com/apple/device-management/tree/release/mdm/errors`
- DDM status item schemas: `https://github.com/apple/device-management/tree/release/declarative/status`
- APNs Provider API: `https://developer.apple.com/documentation/usernotifications/sending_notification_requests_to_apns`
