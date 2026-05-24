# Apps and VPP (managed apps)

Distributing apps to managed devices spans two domains: the **command** that tells a device to install something, and the **license / source** that gives you the right to install it. The two come together as "managed apps."

Apple doc — App and Book Management: `https://developer.apple.com/documentation/devicemanagement/app-and-book-management`
Apple doc — Deploying apps with DDM: `https://developer.apple.com/documentation/devicemanagement/deploying-apps-with-declarative-management`
Schemas: `https://github.com/apple/device-management/tree/release/mdm/commands` (app-related commands), `https://github.com/apple/device-management/tree/release/other/app_and_book_management` (purchase API)

## Three app distribution modes

| Mode | Source | License | Use |
|---|---|---|---|
| VPP (org-purchased) | App Store | VPP license, device- or user-assigned | Bought through ABM/ASM, mass-deploy commercial apps |
| Custom B2B / Custom App | App Store (private listing) | VPP license, distributed only to your org | App built by you/vendor, App-Store-distributed but private |
| Enterprise / In-House | Self-hosted IPA | Apple Developer Enterprise Program license | iOS only; ad-hoc enterprise apps with internal-only signing |

VPP is the dominant path. Enterprise In-House requires a $299/year Apple Developer Enterprise Program membership and is reserved for genuinely internal-only apps (Apple is strict).

## InstallApplication command

The classic command for installing an App-Store app on a device:

```xml
<key>RequestType</key>
<string>InstallApplication</string>
<key>iTunesStoreID</key>
<integer>1234567890</integer>          // App Store ID, integer
<key>InstallAsManaged</key>
<true/>                                  // makes it a managed app (removable by MDM, not user)
<key>ManagementFlags</key>
<integer>5</integer>                     // bitmask, see below
<key>ChangeManagementState</key>
<string>Managed</string>                 // convert installed unmanaged app to managed
<key>Configuration</key>
<dict> ... </dict>                       // managed app configuration delivered with install
<key>Attributes</key>
<dict> ... </dict>                       // VPN association, content filter rules, etc.
<key>iTunesStoreID</key>                 // or
<key>Identifier</key>
<string>com.example.app</string>         // bundle ID, alternative to iTunesStoreID
<key>InstallationKind</key>
<string>InstallOnly</string>             // InstallOnly | Update
```

`ManagementFlags` bitmask:
| Bit | Meaning |
|---|---|
| 1 | Remove app when MDM profile is removed |
| 4 | Prevent backup of app data to iCloud |

`Attributes` keys include `VPNUUID`, `ContentFilterUUID`, `DNSProxyUUID`, `AssociatedDomains`, `Removable`, `Hideable`, `Lockable`. Useful for per-app VPN, content filters, etc.

Schema: `https://github.com/apple/device-management/blob/release/mdm/commands/install_application.yaml`

## VPP license assignment

Before `InstallApplication` will succeed on a device for a paid app, the device (or user) needs a VPP license. License assignment is done via the **App and Book Management API** (sometimes called the Apple Business Manager Apps API).

### Two license modes

| Mode | Identifier | Notes |
|---|---|---|
| Device-based | Device's serial | Default modern choice. No Apple ID required on device. License lives with the device. |
| User-based | iTunesStore account | Requires Apple ID on device. Used for shared iPad in education. Mostly legacy. |

### License assignment flow

1. Org admin buys N copies of an app in ABM/ASM.
2. Your MDM authenticates to App and Book Management API using the ABM/ASM service token (different from the MDM server token — separately issued, often called "Apps and Books token" or "VPP token").
3. Your MDM polls for app inventory and remaining licenses.
4. Before installing on a device, your MDM associates one license to the device (or user) via a license-assignment endpoint.
5. Send `InstallApplication` with `iTunesStoreID` matching the app.
6. Device installs; the App Store sees the assigned license and bypasses purchase prompt.

License assignment / un-assignment uses long-running async jobs — submit, get a job ID, poll for completion.

### Apps and Books service auth

Distinct from the ADE Device Assignment service. Different token, different endpoints. Base URL is typically `https://vpp.itunes.apple.com/mdm/v2/` or the newer App and Book Management endpoints under `https://api.apple-business.apple.com/`. Get the current base URL from the service config endpoint.

## ManagedApplicationList and ManagedApplicationFeedback

`ManagedApplicationList` — query the device for the state of managed apps. Returns per-app:

- `Status` (Managed, ManagedButUninstalled, NeedsRedemption, UserInstalledApp, Installing, PromptingForLogin, Failed, etc.)
- `Identifier` (bundle ID)
- `ExternalVersionIdentifier` (compare to App Store's version ID to detect available updates)
- `IsValidated` (Enterprise apps must validate)
- `BundleSize`, `DynamicSize`

`ManagedApplicationFeedback` — read per-app feedback the app has written (apps that opt into the managed feedback protocol can push key-value status to MDM).

## ManagedApplicationAttributes

Modify per-app attributes (VPN association, content filter assignment, etc.) for already-installed managed apps. Lighter-weight than re-installing.

Note: apps managed by **DDM** don't show up in `ManagedApplicationList` results. They live in DDM status reports instead.

## Managed app configuration

For apps that opted into managed app config (most enterprise-friendly apps do), you can deliver a key-value config dict to the app at install time or at runtime:

- At install: `Configuration` key in `InstallApplication`.
- Later: `Settings` command with `ApplicationConfiguration` item.

The app reads its config via `UserDefaults` under the key `com.apple.configuration.managed`. Apple has full docs on the app side: `https://developer.apple.com/documentation/uikit/uiapplication/applicationiconbadgenumber` (just kidding — search "Managed App Configuration" on developer.apple.com).

## Declarative app deployment

The DDM way of deploying apps. Two declarations:

- `com.apple.configuration.app.managed` — declares the desired app state for the device
- `com.apple.asset.app.managed` — declares the source/license info, referenced by the configuration

A single asset can be referenced by many configurations across activations. The device handles installation, updates, and removal as it reconciles. Status arrives through the DDM status channel.

Apple doc: `https://developer.apple.com/documentation/devicemanagement/deploying-apps-with-declarative-management`

When to prefer DDM-managed apps:
- New deployments, modern fleet (iOS 17+ for full support)
- You want continuous reconciliation (re-install if app removed, update on new version)
- You want richer status

When to stay on `InstallApplication`:
- Older devices
- One-shot installs where you don't want continuous reconciliation
- Existing tooling around `InstallApplication`

The two can coexist within a fleet but not on the same app on the same device — pick one for a given (device, app) pair.

## Enterprise (In-House) apps

iOS only. Distributed by hosting an `.ipa` and a `manifest.plist` somewhere, then `InstallApplication` with `iTunesStoreID` replaced by `ManifestURL`:

```xml
<key>RequestType</key>
<string>InstallApplication</string>
<key>ManifestURL</key>
<string>https://apps.example.com/myapp/manifest.plist</string>
<key>InstallAsManaged</key>
<true/>
```

The manifest is itself a plist describing the `.ipa` location, asset kind, bundle ID, etc.

Apple has been **steadily restricting** Enterprise In-House — apps must validate against Apple's servers periodically, signing certs are revoked aggressively for misuse. Treat as a last resort and only when the use case is genuinely internal-only.

## Gotchas

- **`InstallAsManaged: true`** is what makes an app "managed". Without it, the app is installed but indistinguishable from a user-installed app — MDM can't remove it without the user's involvement.
- **`ChangeManagementState: "Managed"`** converts an existing unmanaged install to managed. Useful for org-bought-but-already-installed apps. Requires user consent on iOS in some versions.
- **Apps targeting iOS 16+** may require additional `Configuration` or `Attributes` to function correctly in managed contexts (per-app VPN no longer Just Works without explicit binding).
- **VPP license-assignment jobs are async** — don't assume the license is in place by the time `InstallApplication` runs. Wait for the job to complete.
- **`iTunesStoreID` is the integer App Store ID** (visible in the App Store URL). Don't confuse with the bundle identifier.
- **Custom B2B apps** require buyer-side acceptance in ABM/ASM before they're available for purchase. Not always instant.
- **Update behavior**: `InstallApplication` with `InstallationKind: Update` updates an existing managed app. Without it, the device may refuse to "install" something that's already there at an older version.
- **`ManagedApplicationFeedback` is per-app opt-in.** The app must call the relevant API. Most don't.

## Quick links

- InstallApplication: `https://github.com/apple/device-management/blob/release/mdm/commands/install_application.yaml`
- ManagedApplicationList: `https://github.com/apple/device-management/blob/release/mdm/commands/managed_application_list.yaml`
- ManagedApplicationAttributes: `https://github.com/apple/device-management/blob/release/mdm/commands/managed_application_attributes.yaml`
- App and Book Management overview: `https://developer.apple.com/documentation/devicemanagement/app-and-book-management`
- Declarative app config: `https://github.com/apple/device-management/blob/release/declarative/declarations/configurations/app.managed.yaml`
- Declarative app asset: `https://github.com/apple/device-management/blob/release/declarative/declarations/assets/app.managed.yaml`
