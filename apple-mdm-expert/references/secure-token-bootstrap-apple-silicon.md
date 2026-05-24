# Secure Token, Bootstrap Token, and Volume Ownership on Apple Silicon

This page is the practical reference for the cryptographic-user
mechanics that gate account creation, FileVault, and any operation
that needs Apple Silicon's APFS data-volume key wrap. It's where
people get stuck when "the command returned 0 but nothing actually
happened."

Apple doc — *Use secure token, bootstrap token, and volume ownership
in deployments*: `https://support.apple.com/guide/deployment/dep24dbdcf9e`
Apple doc — *Managing FileVault in macOS*:
`https://support.apple.com/guide/security/sec8447f5049`
Schema (BootstrapToken CheckIns):
`https://github.com/apple/device-management/tree/release/mdm/checkin`

## The three things often conflated

These are correlated but not the same thing — and getting them
confused is the most common source of "why doesn't this work":

1. **Secure Token** — per-user attribute in OD's `AuthenticationAuthority`
   (`;SecureToken;` tag). A user who holds Secure Token can author
   future Secure Token grants and can unlock a FileVault-encrypted
   data volume. Check with `sysadminctl -secureTokenStatus <user>`.

2. **APFS Volume Owner / Cryptographic User** — APFS-volume-level
   record (UUID + type) that owns a key in the volume's keybag.
   Listed by `diskutil apfs listUsers /`. Types seen in practice:
   - `Local Open Directory User` (a real user with Secure Token)
   - `MDM Bootstrap Token External Key` (the BT itself)
   - `Personal Recovery User` (FileVault PRK material — present
     even when FV is off, on macOS 15+)
   - `Local Recovery User`
   - `iCloud Recovery User`

3. **Bootstrap Token (BT)** — per-device key minted by the SEP and
   escrowed to the MDM via `SetBootstrapToken` Check-In. Lets macOS
   silently grant Secure Token to a new user on first GUI login,
   without needing an existing ST-admin authorizer.

A user can have an OD record with no ST. The MDM BT exists as its
own cryptographic user, independent of any OD user. ST is a per-user
property; volume ownership is a per-volume record; BT is a per-device
recovery key. They cooperate but don't imply each other.

## How users get Secure Token (mechanisms in order)

1. **First user typed at Setup Assistant** — automatic. The user
   the operator (or end user) creates in the "Create your account"
   pane becomes the first ST holder. The interactive password entry
   is what triggers it.

2. **`AutoSetupAdminAccounts` admins** — created during Setup Assistant
   with backend-supplied passwords (or `passwordHash`). On older
   macOS these were documented as NOT auto-getting ST. On macOS 15+
   with a BT-supporting MDM, they get ST on their first GUI login
   via the BT-mediated cascade (mechanism #3 below).

3. **First GUI login of any user on a BT-escrowed device** — Apple's
   documented automatic path. macOS uses the escrowed BT to grant
   ST to that user silently. Works on macOS 10.15.4+. The "user"
   here can be any local OD user, hidden or visible, admin or
   Standard.

4. **Explicit grant via existing ST admin** —
   `sysadminctl -secureTokenOn <newuser> -password <newpw>
    -adminUser <stadmin> -adminPassword <stadminpw>`.
   The authorizer MUST be in the `admin` group AND hold Secure
   Token. Otherwise: **OD error 5101** ("authentication server
   refused operation"). On Standard primaries: transient-promote
   via `dseditgroup -o edit -a <user> admin` → grant → demote.

5. **Fabien Conus user-less-environment self-grant** (legacy):
   `sysadminctl -adminUser <ma> -adminPassword X -secureTokenOn <ma>
    -password X` works ONLY when the volume has ZERO Secure Token
   holders. **CLOSED on macOS 15 Apple Silicon under MDM** — see
   next section.

## The Fabien Conus loophole is CLOSED on macOS 15 + MDM

Fabien Conus (April 2023, fabien.iconus.ch, *Secure token and
bootstrap token in a user-less environment*) documented a pattern
where a hash-created admin could grant itself Secure Token via the
`sysadminctl` "no-ST-holders" bootstrap loophole. The pattern is
widely cited in MacAdmins circles.

**It does not work on macOS 15 Apple Silicon under MDM enrollment.**
Empirically verified on real hardware (FleetLock M9 Phase C,
2026-05-22):

When `AccountConfiguration` completes on a device whose enrollment
profile declares `ServerCapabilities ⊇ com.apple.mdm.bootstraptoken`
and `AccessRights` bit 13 (8192), **TWO things auto-register as APFS
cryptographic users immediately**, before any user GUI-logs-in:

1. `MDM Bootstrap Token External Key` — the BT itself, minted in
   the SEP, escrowed to the MDM. Counted as a volume owner.
2. `Personal Recovery User` — FileVault PRK material. Present even
   when FileVault is OFF on macOS 15.

The OD authorizer rule for `sysadminctl -secureTokenOn` treats both
as "existing Secure Token holders" for the user-less-bootstrap gate.
The "zero ST holders" precondition Fabien's pattern requires is
therefore **never satisfied** on a properly-MDM-enrolled device.

Symptom of attempting the self-grant anyway:
```
$ sudo sysadminctl -adminUser mvsadmin -adminPassword X \
                   -secureTokenOn mvsadmin -password X
2026-05-22 10:23:38 sysadminctl[...]: Done!     ← LIES
$ sudo sysadminctl -secureTokenStatus mvsadmin
2026-05-22 10:23:42 sysadminctl[...]: Secure token is DISABLED for user mvsadmin
```
`sysadminctl` returns exit 0 ("Done!") even when the grant was
silently rejected by OD. **Always verify state independently** —
this is a long-standing Apple footgun.

What works instead on macOS 15 + MDM: mechanism #3 above. Trigger
a GUI login of the target user (or any user) and macOS grants ST
via the escrowed BT silently. See "AccountConfiguration user patterns"
reference for how to make this happen automatically at first boot.

## Bootstrap Token escrow

### When BT escrows automatically (macOS 15 + MDM)

`SetBootstrapToken` Check-In fires automatically when:

- Device is fully MDM-enrolled (`Authenticate` + `TokenUpdate` done)
- Enrollment profile advertised `ServerCapabilities ⊇
  com.apple.mdm.bootstraptoken` and `AccessRights` bit 13 set
- `AccountConfiguration` command is acknowledged by the device

**Confirmed empirically:** on KJYF0XFYYJ (M5 MacBook Pro, macOS 15)
the BT escrowed within seconds of `AccountConfiguration` ack, with
NO GUI login by any user. This is more permissive than some older
Apple guidance suggested ("first login by an ST-enabled user")—
modern macOS appears to escrow BT eagerly once the MDM declares
support.

Schema for the Check-In:
`https://github.com/apple/device-management/blob/release/mdm/checkin/bootstrap_token.yaml`

### How to manually trigger BT escrow

If automatic escrow didn't fire (older macOS, or
`AccountConfiguration` wasn't sent, or the MDM didn't declare the
capability), the device-side trigger is `profiles(1)`:

```bash
# Documented interactive form (prompts for admin):
sudo profiles install -type bootstraptoken

# Undocumented but production-proven flag form (mostlymac.blog
# 2021-2024). Note: flags are NOT in the man page; Apple has never
# committed to keeping them. Add fallback below.
sudo profiles install -type bootstraptoken -user mvsadmin -pass "$pw"

# expect(1) fallback when the flag form silently breaks
expect <<'EOF'
spawn /usr/bin/profiles install -type bootstraptoken
expect "user name:"      { send "mvsadmin\r" }
expect "password for user" { send "secret\r" }
expect eof
EOF
```

Verify with:
```bash
profiles status -type bootstraptoken
# Want:
#   profiles: Bootstrap Token supported on server: YES
#   profiles: Bootstrap Token escrowed to server:  YES
```

### `mdmclient AquireBootstrapToken` is folklore

`/usr/libexec/mdmclient` deliberately documents nothing. The Mosen
profiledocs catalog (most complete reverse-engineered list) does
NOT contain `AquireBootstrapToken`, `AcquireBootstrapToken`,
`EscrowBootstrapToken`, or `escrowToken`. The strings exist in the
binary but as MDM-protocol Check-In messages, not CLI subcommands.

Use `profiles(1)`. To confirm on a target device:
`strings /usr/libexec/mdmclient | grep -i bootstrap`.

## OD error 5101 — sysadminctl authorizer rejected

The most common ST grant failure on macOS Apple Silicon. Message:
"authentication server refused operation" (sometimes truncated).

Means the `-adminUser` is one of:

- Not in the `admin` group (e.g., Standard primary that you forgot
  to promote)
- Not a Secure Token holder (e.g., `AutoSetupAdminAccounts` admin
  before its first GUI login)
- Both

Diagnose with:
```bash
dseditgroup -o checkmember -m <user> admin     # rc=0 = member
sysadminctl -secureTokenStatus <user>          # must show ENABLED
```

Common recoveries:
- Transient promote/demote a Standard primary:
  `dseditgroup -o edit -a primary admin && sysadminctl … &&
   dseditgroup -o edit -d primary admin`
- Use mechanism #3 (BT-mediated first-login grant) to give the
  intended authorizer ST first.

Apple references the rule (without naming the error code) in
*Use secure token, bootstrap token, and volume ownership in
deployments*:
> "Changing the secure token status of a user using sysadminctl
> always requires the user name and password of an existing secure
> token–enabled administrator, either interactively or through the
> appropriate flags on the command."

## sysadminctl footguns

### `sysadminctl` returns 0 even on auth failure
**Always verify state independently after a `sysadminctl` call.**
`-secureTokenOn`, `-addUser`, `-deleteUser`, `-resetPasswordFor`
can all print "Done!" with exit 0 when the operation was rejected
by OD. Apple bug, present since Catalina, still in macOS 15.

### `-UID` is broken since macOS 13
The `-addUser -UID <n>` flag is silently ignored. UIDs auto-assign
501+. If you must pin a UID, do `dscl . change /Users/<name>
UniqueID <old> <new>` + `chown -R` the home directory after creation.

### No `-hidden` flag on `-addUser`
Apple never shipped one. Hide a user post-creation:
```bash
dscl . create /Users/<name> IsHidden 1
```
Also set in `com.apple.loginwindow` `HiddenUsersList` array if you
want loginwindow-level hiding (sometimes desired even if the user
appears in System Settings).

### Password rotation breaks Secure Token via the wrong tool
- `sysadminctl -resetPasswordFor <user> -newPassword <new>
   -adminUser <stadmin> -adminPassword <stadminpw>` — **preserves
  Secure Token**. SEP re-wraps the KEK.
- `passwd <user>` — **BREAKS Secure Token**. Writes raw crypt hash
  bypassing OD's SecureToken machinery.
- Root-only `dscl . -passwd /Users/<u> <new>` (without supplying old
  password) — also breaks ST.

For backend-driven rotation, always use `sysadminctl -resetPasswordFor`
with an ST admin as authorizer. Note: `sysadminctl -resetPasswordFor`
does NOT re-key login.keychain; for users who've GUI-logged-in, the
old keychain becomes orphaned and you need to either supply old
password to a `dscl . -passwd <user> <old> <new>` (which does re-key)
or delete the keychain and let `securityd` recreate it.

## Useful CLI recipes (macOS 15 Apple Silicon)

```bash
# Who has Secure Token?
for u in $(dscl . list /Users UniqueID | awk '$2 >= 500 {print $1}'); do
  sysadminctl -secureTokenStatus "$u" 2>&1
done

# Who can unlock the data volume?
diskutil apfs listUsers /
# Look for "Type: Local Open Directory User" with Volume Owner: Yes
# Plus the MDM Bootstrap Token External Key + Personal Recovery User

# Lightest possible OD password check (no keychain materialization)
dscl . -authonly <user> "<password>"
# rc=0 = password authenticates against OD; rc!=0 = doesn't.

# Has a user ever GUI-logged-in?
last <user>                                      # console login history
ls /Users/<user>/Library/Keychains 2>&1          # absent = never GUI'd

# Show the SEP-managed cryptographic state
sudo fdesetup list                               # FileVault-enabled users
sudo bputil -d                                   # LocalPolicy on Apple Silicon
```

## macOS 26 Tahoe caveats

Two changes in macOS 26 Tahoe that affect this surface (per Apple
*What's new for enterprise in macOS Tahoe 26*,
`https://support.apple.com/en-us/124963`):

1. **FileVault auto-enables on Apple Account sign-in at Setup
   Assistant.** If the user signs in with an Apple Account during
   the SA flow, FileVault turns on automatically AND escrows the
   recovery key to that Apple Account (not your MDM). Belt-and-braces:
   set `LockPrimaryAccountInfo: true` +
   `DontAutoPopulatePrimaryAccountInfo: true` in
   `AccountConfiguration` to keep Apple Account out of the path.

2. **FileVault recovery key auto-rotates on MDM-installed escrow
   config when BT is present.** If your MDM installs a FileVault
   escrow config on a device that already has a recovery key
   present AND a Bootstrap Token escrowed, macOS auto-rotates the
   key before re-escrowing. Useful but means your stored PRK can
   become stale after MDM config changes.

The BT escrow trigger itself is unchanged.

## FleetLock notes

See `Docs/m9_implementation_plan.md` for the full project context
including the M9 v1 architectural pivot.

- M7-A creates `mvsadmin` (hidden admin) via
  `AutoSetupAdminAccounts` at enrollment.
- M7-B grants `mvsadmin` Secure Token via the transient-promote-
  primary pattern + a Self-Service password capture for the primary
  user's authorizer credentials.
- M8 uses `mvsadmin` (now ST-enabled) as the silent authorizer for
  `sysadminctl -addUser` + `-secureTokenOn` of additional backend-
  defined users. Password injected at dispatch from the
  `M7_ADMIN_PASSWORD` Worker secret.
- M9 v1 attempted to replace M7-B's interactive grant with an
  agent-driven Fabien Conus self-grant. **This failed empirically
  on KJYF0XFYYJ — see this page's "loophole closed" section.**
- M9 v2 (Task #28) pivots to creating the end-user IN
  `AccountConfiguration` (Patterns A or B in the
  `account-configuration-user-patterns.md` reference) so Apple's
  BT-mediated first-login grant does the work.

### When Phase C of M9 erase-and-reverify happens

The first agent self-grant attempt always fails because the loophole
is closed. The Worker's `/m9/init` retry also fails. Verifying that
the rest of the M9 chain (loginwindow profile, useradd_local on ST'd
mvsadmin, end-user login) works requires triggering mvsadmin's ST
grant via mechanism #3 — currently a manual "log in as mvsadmin once
at loginwindow" step. M9 v2 will automate this.

## Quick links

- `https://support.apple.com/guide/deployment/dep24dbdcf9e` — Apple's
  ST/BT/volume-ownership reference
- `https://support.apple.com/guide/security/sec8447f5049` — Managing
  FileVault in macOS
- `https://github.com/apple/device-management/tree/release/mdm/checkin`
  — `SetBootstrapToken` / `GetBootstrapToken` schemas
- `https://keith.github.io/xcode-man-pages/sysadminctl.1.html` —
  current `sysadminctl(1)` man page
- `https://fabien.iconus.ch/secure-token-and-bootstrap-token-in-a-user-less-environment/`
  — Fabien Conus's original writeup (now closed on macOS 15 + MDM)
- `https://mostlymac.blog/2021/06/09/using-a-self-service-policy-to-grant-end-users-a-secure-token/`
  — undocumented `profiles install -user/-pass` flags
