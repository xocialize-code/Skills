# AccountConfiguration: user-creation patterns at enrollment

`AccountConfiguration` is the single highest-leverage MDM command
for user provisioning, because **it's the only command that creates
users during Setup Assistant — when those users can get Secure Token
automatically via Apple's documented BT-mediated first-login path.**

Every alternative ("provision users post-enrollment via an agent")
runs into the Apple Silicon volume-ownership gate
(see `secure-token-bootstrap-apple-silicon.md`). For zero-touch
end-user provisioning at fleet scale, `AccountConfiguration` is
where you make it happen.

Apple doc — *AccountConfigurationCommand.Command*:
`https://developer.apple.com/documentation/devicemanagement/accountconfigurationcommand`
Schema (machine-readable):
`https://github.com/apple/device-management/blob/release/mdm/commands/account_configuration.yaml`

## Key matrix

```xml
<dict>
  <key>RequestType</key><string>AccountConfiguration</string>

  <!-- PRIMARY user pane controls (the "Create your account" screen) -->
  <key>SkipPrimarySetupAccountCreation</key><true/>
  <!-- true = NO "Create your account" pane in Setup Assistant.
       Combine with AutoSetupAdminAccounts to make sure SOME user
       exists before loginwindow renders. -->

  <key>PrimaryAccountUserName</key><string>alice</string>
  <key>PrimaryAccountFullName</key><string>Alice Smith</string>
  <!-- Pre-populate the primary creation pane's name fields. -->

  <key>LockPrimaryAccountInfo</key><true/>
  <!-- Make the name fields uneditable. User can only type a
       password. Pair with the PrimaryAccount* keys above. -->

  <key>SetPrimarySetupAccountAsRegularUser</key><true/>
  <!-- true = Standard role; false (default) = admin role.
       Only applies to the primary created via the pane. -->

  <key>DontAutoPopulatePrimaryAccountInfo</key><true/>
  <!-- Belt-and-braces against Apple-Account auto-populate paths,
       relevant on macOS 26 Tahoe (auto-FV enable on Apple Account
       sign-in). -->

  <!-- ADMIN accounts created hash-only at Setup Assistant time. -->
  <key>AutoSetupAdminAccounts</key>
  <array>
    <dict>
      <key>shortName</key><string>mvsadmin</string>
      <key>fullName</key><string>Managed Admin</string>
      <key>hidden</key><true/>
      <key>password</key><string>...</string>
      <!-- OR pre-hashed (more secure than shipping plaintext): -->
      <!-- <key>passwordHash</key><data>base64-of-ShadowHashData</data> -->
      <!-- <key>hashType</key><string>SALTED-SHA512-PBKDF2</string> -->
    </dict>
  </array>
</dict>
```

**CRITICAL — `AutoSetupAdminAccounts` is structurally an array but
Setup Assistant ONLY USES THE FIRST ELEMENT.** Quoting the
authoritative schema
(`apple/device-management/mdm/commands/account.configuration.yaml`,
release branch, 2026 OS-version):

> A dictionary that describes the administrator account to create
> with Setup Assistant, which uses the first element and ignores
> additional elements.

So you can configure exactly ONE admin via this mechanism — either
a hidden management admin OR a visible end-user, not both. This
constraint shapes everything below.

## The two MDM-blessed patterns for "end-user visible at first loginwindow"

The single biggest UX observation: **a true drop-ship enrollment
should land the device at a loginwindow with the END USER'S tile
already visible.** No "Other..." prompt, no blank name+password
fields, no IT touch between unboxing and end-user login.

There are two mechanisms to achieve this. Choose based on whether
the end-user needs admin role and whether the backend or the user
supplies the password.

### Pattern A — End-user is the sole admin (no mvsadmin)

```xml
<key>SkipPrimarySetupAccountCreation</key><true/>
<key>AutoSetupAdminAccounts</key>
<array>
  <dict>
    <key>shortName</key><string>alice</string>
    <key>fullName</key><string>Alice Smith</string>
    <key>hidden</key><false/>
    <key>passwordHash</key>
    <data>... base64-encoded salted PBKDF2 SHA512 hash ...</data>
  </dict>
</array>
```

**Schema constraint:** because `AutoSetupAdminAccounts` only uses the
first element, this pattern means you give up a hidden management
admin (`mvsadmin`). Whatever you put here IS the only admin created
during Setup Assistant.

What happens:
- Setup Assistant SKIPS the "Create your account" pane entirely.
- ONE user created during enrollment: alice (visible, admin).
- Device boots to loginwindow showing **alice's tile** (no other
  users to show).
- alice types her backend-supplied password.
- macOS auto-grants Secure Token to alice via the escrowed BT on
  this first GUI login.
- alice also becomes an APFS Local Open Directory User / volume
  owner via the same grant.

**Best when:** single-user devices where end-user admin rights are
acceptable AND you can do without a hidden management admin (e.g.,
ABM-supervised devices where you have other admin paths via Apple
Configurator, recovery mode, FileVault PRK, or a post-enrollment
agent-created hidden admin).

**Worst when:** drop-ship rentals or shared devices where you NEED
a break-glass hidden admin for IT support. For that case, see
Pattern B (which preserves mvsadmin at the cost of one human action
at Setup Assistant) or Pattern C (ConfigurationURL/ADDE — full
zero-touch but needs an IdP).

### Pattern B — Locked primary + hidden mvsadmin, user types password

The pattern that preserves a hidden management admin AND lands the
end-user as a loginwindow tile, at the cost of one human action
(end-user types a password during Setup Assistant).

```xml
<key>SkipPrimarySetupAccountCreation</key><false/>   <!-- SHOW the pane -->
<key>PrimaryAccountUserName</key><string>alice</string>
<key>PrimaryAccountFullName</key><string>Alice Smith</string>
<key>LockPrimaryAccountInfo</key><true/>
<key>SetPrimarySetupAccountAsRegularUser</key><true/>
<key>DontAutoPopulatePrimaryAccountInfo</key><true/>
<key>AutoSetupAdminAccounts</key>
<array>
  <dict>
    <key>shortName</key><string>mvsadmin</string>
    <key>fullName</key><string>Managed Admin</string>
    <key>hidden</key><true/>
    <key>password</key><string>... management secret ...</string>
  </dict>
</array>
```

What happens:
- Setup Assistant DOES show the "Create your account" pane.
- Name fields are pre-filled with `Alice Smith` / `alice` and
  uneditable (`LockPrimaryAccountInfo: true`).
- End-user types a password of their choice and clicks Continue.
- Setup Assistant creates alice as a Standard user. (Despite being
  Standard, she gets Secure Token because she's the first user
  created via the interactive pane — this is mechanism #1 in the
  ST reference.)
- mvsadmin is also created in the background (hidden, admin).
- Device boots to loginwindow with alice's tile.
- alice types her own password (the one she just set).
- alice is already ST + volume owner from the pane creation.

**Best when:** end-users should have Standard role (most fleets;
matches enterprise least-privilege norm), you want to keep a hidden
break-glass admin (`mvsadmin`), AND it's acceptable that the end-user
types a password during Setup Assistant. Not "pure zero-touch" but
the human action is the end user setting their own password —
minimal friction, no IT-issued temporary credential to remember.

### Pattern C — Account-Driven Device Enrollment (ConfigurationURL)

True zero-touch, including no password entry at Setup Assistant.
What Mosyle ships as "Mosyle Auth." Requires building an IdP that
speaks Apple's Account-Driven Authentication protocol.

The mechanism is encoded in `LockPrimaryAccountInfo`'s undocumented
behavior, quoted verbatim from the schema:

> If the user's password is also available from authentication
> through ConfigurationURL, Setup Assistant automatically creates
> the primary account with that information and skips showing the
> user interface to view or edit these fields.

The DEP profile includes a `configuration_web_url` pointing at your
IdP. During Setup Assistant, the device opens that URL in an
embedded WebView; the user authenticates with your IdP; the IdP
returns user credentials in a structured response; Apple's Setup
Assistant uses those credentials to silently create the primary
account.

Apple references:
- `https://developer.apple.com/documentation/devicemanagement/account-driven-device-enrollment`
- DEP profile `configuration_web_url` field
- See also `enrollment-account-driven.md` reference in this skill

**Build cost:** an IdP endpoint + identity store + Apple-Account-Driven-
Authentication protocol implementation. Significant new infrastructure
unless you already have an IdP that speaks the protocol.

**Returns:** the most polished UX — end-user signs in once at the IdP
screen during Setup Assistant (which they're going to do anyway for
identity reasons), and lands at the loginwindow with their tile and
a password already set.

## The anti-pattern: skip primary + agent self-grant

This is the architectural mistake FleetLock M9 v1 made (and walked
back via the M9 v2 pivot, Task #28). DO NOT do this:

```xml
<key>SkipPrimarySetupAccountCreation</key><true/>
<key>AutoSetupAdminAccounts</key>
<array>
  <dict>
    <key>shortName</key><string>mvsadmin</string>
    <key>hidden</key><true/>
    <key>password</key><string>...</string>
  </dict>
</array>
```

Followed by an agent post-enrollment trying to grant mvsadmin
Secure Token via Fabien Conus's user-less self-grant:
```
sysadminctl -adminUser mvsadmin -adminPassword X \
            -secureTokenOn mvsadmin -password X
```

**This doesn't work on macOS 15 Apple Silicon under MDM.** The MDM
Bootstrap Token External Key + Personal Recovery User auto-register
as APFS cryptographic users when `AccountConfiguration` completes,
which closes the "zero ST holders" loophole the self-grant requires.
See `secure-token-bootstrap-apple-silicon.md` for the empirical
diagnosis.

The only way to grant mvsadmin Secure Token on this configuration
is to trigger a GUI login of mvsadmin (which materializes the
BT-mediated grant). That requires either a human at the device or
an automated auto-login + reboot hack — neither of which beats
just putting the end-user in `AccountConfiguration` (Patterns A or
B above).

## Setup Assistant pane skips (`skip_setup_items`)

`AccountConfiguration` skips the account-creation pane. Other Setup
Assistant panes (Apple ID, Privacy, Touch ID, TimeZone, …) are
skipped via the DEP **profile's** `skip_setup_items` array, not via
`AccountConfiguration`.

Common skip items (case-sensitive):
- `AppleID` — Apple Account sign-in pane (also avoids the macOS 26
  Tahoe FileVault auto-enable trap)
- `Privacy` — privacy settings explanation
- `TOS` — Terms of Service
- `Diagnostics` — diagnostics sharing
- `Siri` — Siri setup
- `TouchID` — Touch ID setup
- `Payment` — Apple Pay
- `Zoom` — display zoom
- `iCloudStorage` — iCloud storage
- `iCloudDiagnostics` — iCloud analytics
- `Restore` — Migration Assistant
- `Android` — Android migration
- `Passcode` — passcode setup
- `Biometric` — biometric setup
- `FileVault` — FileVault enable
- `iCloudPaymentService` — iCloud Payment Service
- `ScreenTime` — Screen Time
- `DeviceToDeviceMigration` — quick start
- `Welcome` — welcome screen
- `Appearance` — light/dark mode
- `Accessibility` — accessibility shortcut
- `Wallpaper` — wallpaper picker

Full enum:
`https://github.com/apple/device-management/blob/release/other/profileservice/profile_response.yaml`
under `skip_setup_items`.

For a truly minimal Setup Assistant experience under either Pattern
A or B, skip the bulk of these. Some panes always render regardless
(language, country, Wi-Fi, Remote Management) — accept those.

## Peer MDM behavior (cross-reference)

What do other MDMs actually ship for zero-touch user provisioning?
(Public docs + reverse-engineered observation.)

### Mosyle
Achieves "end-user as loginwindow tile at first boot" via both
patterns:
- **Local User** profile UI with `Type: Administrator` and
  `hidden: false` → Pattern A
- **Mosyle Auth** flow with locked primary fields → Pattern B
- Profile has explicit warning "do not use Local User profile if
  also using Mosyle Auth + ADE" — implies the underlying mechanism
  is one of the two patterns above and double-configuring breaks it.

### Jamf Pro
- Primarily Pattern B (PrePopulated + Locked) via the **PreStage
  Enrollment** "Account Settings" tab.
- Can also create a separate management admin via PreStage's
  "Create local administrator account" → this maps to a
  `AutoSetupAdminAccounts` entry under the hood.
- The `jamf` binary (root agent) handles any post-enrollment
  user creation that's not Setup-Assistant-time — equivalent to
  what M8 does.

### Kandji
- Per their KB *Create User Accounts*: their agent creates users
  post-enrollment, BUT documents that "this does not automatically
  grant that user a secure token. The supported recovery is to have
  the user (or admin) log in once at the GUI login window."
- So Kandji's post-enrollment user creation is the M8 equivalent
  and they explicitly accept the "first GUI login is needed for ST"
  limitation. For drop-ship Standard users they use PrestageEnrollment
  account settings (= Pattern B).

### Addigy, FileWave, SOTI
- All rely on PrestageEnrollment-equivalent (= Pattern B).
- All warn that resetting passwords via their GoLive-equivalents
  breaks Secure Token (use `sysadminctl -resetPasswordFor` instead).

## macOS 26 Tahoe caveats

If your `AccountConfiguration` does NOT skip the Apple ID Setup
Assistant pane (via the DEP profile's `skip_setup_items`), macOS
26 Tahoe will:

- **Auto-enable FileVault** when the user signs in with an Apple
  Account during SA.
- **Escrow the FileVault recovery key to the Apple Account** (not
  to your MDM).

Belt-and-braces:
1. Include `AppleID` in `skip_setup_items`.
2. Set `LockPrimaryAccountInfo: true` +
   `DontAutoPopulatePrimaryAccountInfo: true` in
   `AccountConfiguration` so the device cannot back into Apple
   Account flow via the primary account pane.

Reference: Apple *What's new for enterprise in macOS Tahoe 26*,
`https://support.apple.com/en-us/124963`.

## FleetLock notes

See `Docs/m9_implementation_plan.md` for the project context.

Pre-M9 v2 (the current FleetLock production state), the M7-A
configuration is:
```ts
// src/services/accounts.ts
buildCommandPlist(commandUuid, 'AccountConfiguration', {
  SkipPrimarySetupAccountCreation: spec.skipPrimaryAccountCreation, // false
  SetPrimarySetupAccountAsRegularUser: spec.primaryAccountAsRegularUser, // false (admin)
  AutoSetupAdminAccounts: [
    { shortName: 'mvsadmin', fullName: 'Managed Admin',
      hidden: true, password: env.M7_ADMIN_PASSWORD }
  ],
})
```
Which is the **default M7-A path**: end user types their own name +
password at Setup Assistant (becomes admin + first ST holder),
mvsadmin created hidden in background.

M9 v1 attempted to change to `SkipPrimarySetupAccountCreation: true`
plus a `noPrimary: true` flag plus agent self-grant. **This is the
anti-pattern above.** M9 v2 (Task #28) will pivot to Pattern A
(end-user-as-admin via AutoSetupAdminAccounts) or Pattern B
(end-user-as-Standard via locked PrimaryAccount* keys), depending
on the per-user `role` field in the FleetLock `device_users` table.

Implementation plan for the v2 builder:

```ts
// pseudocode for src/services/accounts.ts (M9 v2 shape)
async function buildAccountConfiguration(env, udid, commandUuid) {
  const adminSpec = await getLocalAdminSpec(env);          // mvsadmin
  const preStaged = await listDeviceUsers(env, udid);      // end-user(s)

  const adminAccounts = [adminEntry(adminSpec, env.M7_ADMIN_PASSWORD)];
  let primaryKeys = {};

  for (const u of preStaged) {
    if (u.role === 'admin') {
      // Pattern A
      adminAccounts.push({
        shortName: u.shortName, fullName: u.fullName,
        hidden: u.hidden, password: u.password,
      });
    } else {
      // Pattern B — only one Standard primary per device (Apple limit)
      if (Object.keys(primaryKeys).length > 0) {
        throw new Error('only one Standard primary supported per device');
      }
      primaryKeys = {
        PrimaryAccountUserName: u.shortName,
        PrimaryAccountFullName: u.fullName,
        LockPrimaryAccountInfo: true,
        SetPrimarySetupAccountAsRegularUser: true,
        DontAutoPopulatePrimaryAccountInfo: true,
      };
    }
  }

  return buildCommandPlist(commandUuid, 'AccountConfiguration', {
    AutoSetupAdminAccounts: adminAccounts,
    SkipPrimarySetupAccountCreation:
      Object.keys(primaryKeys).length === 0,  // skip if no Standard primary
    ...primaryKeys,
  });
}
```

Key constraints/decisions:
- **One Standard primary per device max** (`PrimaryAccount*` keys
  are not pluralizable — pane creates exactly one primary). Multiple
  Standard end-users requires post-enrollment M8 useradd_local for
  the second+ Standard user, accepting they won't appear at first
  loginwindow but will after.
- **Multiple admin end-users OK** via `AutoSetupAdminAccounts` array
  with multiple entries (all visible at loginwindow).
- **Hybrid case** (one Standard primary + N admin end-users + 1
  hidden mvsadmin) is supported — `PrimaryAccountUserName` config
  doesn't conflict with the `AutoSetupAdminAccounts` array.

## Quick links

- `https://developer.apple.com/documentation/devicemanagement/accountconfigurationcommand`
  — Apple doc
- `https://github.com/apple/device-management/blob/release/mdm/commands/account_configuration.yaml`
  — schema (the authoritative source for available keys + types)
- `https://github.com/apple/device-management/blob/release/other/profileservice/profile_response.yaml`
  — DEP profile schema (including `skip_setup_items`)
- `secure-token-bootstrap-apple-silicon.md` — why this pattern matters
  (the underlying Secure Token / Bootstrap Token mechanics)
