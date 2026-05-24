# Apple docs URL map

This is the URL atlas. When you need to fetch Apple documentation or look up a schema, this is where you start.

## The three URL bases

| Source | URL base | Format | When to use |
|---|---|---|---|
| Apple developer docs (HTML) | `https://developer.apple.com/documentation/devicemanagement/` | JS-rendered HTML | Linking to a doc page for the user to read in a browser |
| Apple developer docs (Markdown) | `https://developer.apple.com/tutorials/data/documentation/devicemanagement/` + `.md` | Raw markdown | Programmatic fetch via `web_fetch` |
| Apple schema (YAML) | `https://github.com/apple/device-management/blob/release/` | YAML files | Authoritative type info, platform availability, enums |

## Fetching the Markdown form

The browser-rendered HTML URL returns an empty placeholder when fetched directly because content is loaded by JavaScript. To get the actual content via `web_fetch`, transform the URL:

```
HTML  : https://developer.apple.com/documentation/devicemanagement/check-in
MD    : https://developer.apple.com/tutorials/data/documentation/devicemanagement/check-in.md
```

Rule: insert `tutorials/data/` after the domain and append `.md` to the path. Path segments stay lowercase and hyphenated.

**Important constraint**: `web_fetch` only allows URLs that have appeared in a prior `web_search` result or that the user typed. If you've never seen the specific `.md` URL before in this conversation, you must first run a `web_search` that surfaces it (search for something like `"developer.apple.com" "devicemanagement/[topic]"`), then fetch.

## Top-level topic URLs

| Topic | HTML | MD path (append `.md`) |
|---|---|---|
| Root | `/documentation/devicemanagement` | `/tutorials/data/documentation/devicemanagement` |
| Configuring Multiple Devices Using Profiles | `/configuring-multiple-devices-using-profiles` | same |
| Profile-Specific Payload Keys | `/profile-specific-payload-keys` | same |
| Implementing Device Management | `/implementing-device-management` | same |
| Commands and Queries | `/commands-and-queries` | same |
| Check-in | `/check-in` | same |
| Account-driven enrollment | `/account-driven-enrollment` | same |
| Migrating managed devices | `/migrating-managed-devices` | same |
| Leveraging declarative management at scale | `/leveraging-the-declarative-management-data-model-to-scale-devices` | same |
| Integrating Declarative Management | `/integrating-declarative-management` | same |
| Deploying apps with declarative management | `/deploying-apps-with-declarative-management` | same |
| Declarations | `/devicemanagement-declarations` | same |
| Status Reports | `/status-reports` | same |
| Device Assignment | `/device-assignment` | same |
| Roster Management | `/roster-management` | same |
| App and Book Management | `/app-and-book-management` | same |

For individual symbols (commands, payloads, etc.), the URL pattern continues with the lowercase symbol name, e.g. `/devicemanagement/installapplicationcommand` for the `InstallApplication` command.

## GitHub schema layout

The repo is `https://github.com/apple/device-management`, branch `release`. Current release tracks iOS / macOS / tvOS / visionOS / watchOS 26.2.

```
device-management/
├── mdm/
│   ├── commands/        # one YAML per command
│   ├── checkin/         # check-in request schemas (Authenticate, TokenUpdate, ...)
│   ├── profiles/        # one YAML per profile payload type
│   └── errors/          # error code catalog
├── declarative/
│   ├── declarations/    # one YAML per declaration type
│   ├── status/          # status item schemas
│   └── protocol/        # DDM transport schemas
├── other/               # Device Assignment service, app/book service, etc.
└── docs/
    └── schema.md        # YAML meta-schema (what fields each YAML file uses)
```

To find a specific schema, use GitHub's file finder (press `t` on the repo page) and type the symbol name. File names use snake_case versions of the symbol — e.g. `InstallApplication` → `mdm/commands/install_application.yaml`. The `apple-docs-url-map` section "Symbol → schema lookup" below gives the rules.

### Raw URL pattern for direct fetch

```
https://raw.githubusercontent.com/apple/device-management/release/[path]
```

Example: `https://raw.githubusercontent.com/apple/device-management/release/mdm/commands/install_application.yaml`

This is the fastest way to grab the authoritative type/enum/availability data for a specific symbol.

## Symbol → schema lookup rules

| Symbol form | YAML path |
|---|---|
| `InstallApplicationCommand` | `mdm/commands/install_application.yaml` |
| `DeviceInformationCommand` | `mdm/commands/device_information.yaml` |
| `TokenUpdate` (check-in) | `mdm/checkin/token_update.yaml` |
| `Authenticate` (check-in) | `mdm/checkin/authenticate.yaml` |
| Payload `com.apple.wifi.managed` | `mdm/profiles/com.apple.wifi.managed.yaml` |
| Declaration `com.apple.configuration.account.mail` | `declarative/declarations/configurations/account.mail.yaml` |

The two reliable patterns: commands and check-ins use `snake_case`; profiles and declarations use the exact reverse-DNS PayloadType / declaration identifier as the filename.

## Apple Support deployment guide (admin-facing)

`https://support.apple.com/guide/deployment/` — concept-level explanations, not protocol details. Use for:

- Understanding the difference between supervised, user-approved, etc.
- ABM / ASM admin workflows (creating MDM server tokens, assigning devices)
- High-level descriptions of what a payload accomplishes

This site is regular HTML and fetchable directly, though pages are long and sometimes spread thin.

## Apple platform release notes

When a feature's availability is uncertain, the most reliable answer is the platform release notes:

- iOS / iPadOS: `https://developer.apple.com/documentation/ios-ipados-release-notes`
- macOS: `https://developer.apple.com/documentation/macos-release-notes`

Search for "MDM", "declarative", or the specific command/payload name. New MDM features get a line in the "Device Management" or "Enterprise" subsection.

## When Apple's docs are wrong or incomplete

It happens. In order of trust:

1. The YAML schema in `apple/device-management` — closest to ground truth, regenerated each release.
2. The actual behavior observed on a current OS — definitive but expensive to determine.
3. The Apple developer doc page — usually right, occasionally lags.
4. Apple Support deployment guide — high-level, can be subtly out of date.

Never trust a blog post or third-party doc over (1) or (2). If a doc and the schema disagree, the schema wins.
