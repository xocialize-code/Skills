# Index: which reference for which question

This is a router. Find the row that matches what you're trying to do, then go to the file (and Apple URL) listed.

## Mental model questions

| Question | File |
|---|---|
| "How does MDM actually work end-to-end?" | `mdm-protocol-overview.md` |
| "What's the difference between MDM, DDM, ABM/ASM, ADE?" | `mdm-protocol-overview.md` (§ Glossary) |
| "Where do I find the canonical schema for X?" | `apple-docs-url-map.md` |
| "Fetch the Apple doc for [topic]" | `apple-docs-url-map.md` (URL patterns + how to surface for web_fetch) |

## Server↔device control plane (the daily MDM)

| Question | File |
|---|---|
| "Device just enrolled, what check-in messages will I receive?" | `checkin.md` |
| "What's in an Authenticate body?" | `checkin.md` |
| "TokenUpdate — when does it fire, what do I store?" | `checkin.md` |
| "Difference between PushMagic, Topic, and the MDM Push Certificate UID?" | `checkin.md` (§ Push identity triad) |
| "CheckOut — how do I clean up?" | `checkin.md` |
| "Device sent UserAuthenticate, what now?" | `checkin.md` |
| "How do I send a command to a device?" | `commands-and-queries.md` |
| "What's the structure of a command response?" | `commands-and-queries.md` |
| "Find me the schema for the [X] command" | `commands-and-queries.md` (table → GitHub path) |
| "What does NotNow / Idle / Acknowledged / Error / CommandFormatError mean?" | `commands-and-queries.md` (§ Status values) |
| "Async command — how do I correlate the response?" | `commands-and-queries.md` (§ Async patterns) |
| "DeviceInformation / SecurityInfo / CertificateList — what do they return?" | `commands-and-queries.md` (Queries table) |

## Configuration profiles (`.mobileconfig`)

| Question | File |
|---|---|
| "Structure of a profile?" | `configuration-profiles.md` |
| "Top-level keys — PayloadIdentifier, PayloadUUID, PayloadContent, etc." | `configuration-profiles.md` (§ Top-level keys) |
| "How do I sign a profile?" | `configuration-profiles.md` (§ Signing & encryption) |
| "Specific payload type [com.apple.X]?" | `configuration-profiles.md` (§ Payload catalog) + GitHub path |
| "PayloadScope — System vs User?" | `configuration-profiles.md` |
| "Differences between supervised, user-approved, and DEP-enrolled?" | `configuration-profiles.md` (§ Trust tiers) |
| "Profile install via command vs via UI?" | `configuration-profiles.md` (§ Delivery modes) |

## Declarative Device Management

| Question | File |
|---|---|
| "What is DDM and how is it different from classic MDM?" | `ddm.md` |
| "Synchronization tokens, declaration items, status reports" | `ddm.md` (§ Protocol) |
| "Activation, Configuration, Asset, Management — what are these?" | `ddm.md` (§ Declaration types) |
| "How do I deploy a managed app via DDM?" | `ddm.md` (§ App declarations) + `apps-vpp.md` |
| "DDM status report channel — what's on it?" | `ddm.md` (§ Status reports) |
| "Find the schema for declaration `com.apple.configuration.X`" | `ddm.md` (table → GitHub path) |
| "When should I prefer DDM over a classic profile?" | `ddm.md` (§ When to use DDM) |

## Enrollment

| Question | File |
|---|---|
| "ADE / DEP / Automated Device Enrollment from scratch" | `enrollment-ade.md` |
| "Device Assignment service API (ABM/ASM)" | `enrollment-ade.md` + `deployment-services.md` |
| "EnrollmentProfile payload" | `enrollment-ade.md` |
| "Account-Driven User Enrollment (BYOD with Managed Apple ID)" | `enrollment-account-driven.md` |
| "Account-Driven Device Enrollment" | `enrollment-account-driven.md` |
| "OAuth2 user enrollment flow" | `enrollment-account-driven.md` |
| "OTA profile-based enrollment (no ABM)" | `enrollment-ota.md` |
| "Service discovery / `.well-known/com.apple.remotemanagement`" | `enrollment-account-driven.md` |
| "SCEP identity certificate issuance" | `enrollment-scep-acme.md` |
| "ACME-DM (newer device-attested ACME for MDM)" | `enrollment-scep-acme.md` |
| "Migrating a device between MDM servers" | `enrollment-ade.md` (§ Migration) |

## Apps and content

| Question | File |
|---|---|
| "InstallApplication command" | `apps-vpp.md` |
| "VPP license assignment / association to device or user" | `apps-vpp.md` |
| "Managed app configuration (`com.apple.app.configuration`)" | `apps-vpp.md` |
| "ManagedApplicationList / ManagedApplicationFeedback" | `apps-vpp.md` |
| "App and Book Management API (ABM/ASM)" | `deployment-services.md` |
| "How do I distribute a custom B2B / in-house app?" | `apps-vpp.md` (§ Distribution modes) |

## Deployment services (Apple Business/School Manager API)

| Question | File |
|---|---|
| "How do I authenticate to the Device Assignment service?" | `deployment-services.md` (§ Auth & tokens) |
| "Roster Management — classes, students, teachers" | `deployment-services.md` |
| "App and Book purchases via the deployment API" | `deployment-services.md` |
| "Service token rotation / expiration" | `deployment-services.md` (§ Token lifecycle) |

## Secure Token, Bootstrap Token, Volume Owner (macOS Apple Silicon)

| Question | File |
|---|---|
| "What is Secure Token vs Volume Owner vs Bootstrap Token?" | `secure-token-bootstrap-apple-silicon.md` (§ The three things often conflated) |
| "How does a user get Secure Token?" | `secure-token-bootstrap-apple-silicon.md` (§ How users get Secure Token) |
| "Why is my self-grant via `sysadminctl -secureTokenOn` failing silently?" | `secure-token-bootstrap-apple-silicon.md` (§ Fabien Conus loophole closed) |
| "When does Bootstrap Token escrow automatically? Can I trigger it?" | `secure-token-bootstrap-apple-silicon.md` (§ Bootstrap Token escrow) |
| "What does `profiles install -type bootstraptoken` actually do?" | `secure-token-bootstrap-apple-silicon.md` (§ profiles install cookbook) |
| "Is `mdmclient AquireBootstrapToken` real?" | `secure-token-bootstrap-apple-silicon.md` (§ folklore) |
| "OD error 5101 — what does it mean and how do I fix it?" | `secure-token-bootstrap-apple-silicon.md` (§ OD 5101) + `errors.md` (§ OD 5101) |
| "sysadminctl is returning 0 but nothing happened" | `secure-token-bootstrap-apple-silicon.md` (§ sysadminctl footguns) |
| "How do I list APFS cryptographic users / volume owners?" | `secure-token-bootstrap-apple-silicon.md` (§ CLI recipes) |
| "How do I rotate a user's password without breaking Secure Token?" | `secure-token-bootstrap-apple-silicon.md` (§ Password rotation) |

## AccountConfiguration — creating users during enrollment

| Question | File |
|---|---|
| "How do I make the end-user appear as a tile at first loginwindow?" | `account-configuration-user-patterns.md` (§ Two MDM-blessed patterns) |
| "End-user-as-admin (backend password) vs end-user-as-Standard (user types password)?" | `account-configuration-user-patterns.md` (Patterns A and B) |
| "Why can't I just provision the end-user post-enrollment via my agent?" | `account-configuration-user-patterns.md` (§ Anti-pattern) + `secure-token-bootstrap-apple-silicon.md` |
| "AccountConfiguration key reference (LockPrimaryAccountInfo, PrimaryAccountUserName, etc.)" | `account-configuration-user-patterns.md` (§ Key matrix) |
| "What Setup Assistant panes can I skip?" | `account-configuration-user-patterns.md` (§ skip_setup_items) |
| "What does Mosyle / Jamf / Kandji actually do for zero-touch user provisioning?" | `account-configuration-user-patterns.md` (§ Peer MDM behavior) |
| "macOS 26 Tahoe auto-FileVault-enable — how do I block it?" | `account-configuration-user-patterns.md` (§ Tahoe caveats) + `secure-token-bootstrap-apple-silicon.md` |

## Errors and debugging

| Question | File |
|---|---|
| "What does MDM error code [N] mean?" | `errors.md` |
| "Device returned `Error` Status — how do I diagnose?" | `errors.md` (§ Status: Error) |
| "Push not reaching device" | `errors.md` (§ Push diagnostics) |
| "CommandFormatError" | `errors.md` |
| "DDM declaration rejected (status report)" | `errors.md` (§ DDM errors) |
| "TokenUpdate stopped firing" | `errors.md` (§ Lost-device states) |
| "OD error 5101 from `sysadminctl`" | `errors.md` (§ OD 5101) + `secure-token-bootstrap-apple-silicon.md` |

## FleetLock-specific

| Question | File |
|---|---|
| "How do I represent a profile or command in TypeScript with `@plist/plist`?" | `fleetlock-stack.md` |
| "Durable Object design for a per-device command queue" | `fleetlock-stack.md` (§ DO patterns) |
| "Where does the SCEP container fit in?" | `fleetlock-stack.md` (§ SCEP via Containers) |
| "What does NanoMDM do that I need to do differently on Workers?" | `fleetlock-stack.md` (§ NanoMDM divergences) |
| "D1 schema for enrolled devices, push tokens, certificates" | `fleetlock-stack.md` (§ D1 schema sketch) |
| "How do I sign profiles in a Worker (no OpenSSL CLI)?" | `fleetlock-stack.md` (§ Profile signing in Workers) |
| "APNs HTTP/2 push from a Worker" | `fleetlock-stack.md` (§ APNs from Workers) |

## When the answer isn't here

If your question doesn't match any row, the workflow is:

1. Search Apple docs: `web_search` with site filter `site:developer.apple.com/documentation/devicemanagement` plus a specific term.
2. If you found a topic URL, fetch its `.md` form: replace `developer.apple.com/documentation/` with `developer.apple.com/tutorials/data/documentation/` and append `.md`.
3. If you need the exact schema, check `https://github.com/apple/device-management/tree/release/` — files are named after the symbol (e.g., `mdm/commands/install_application.yaml`).
4. If the question involves admin-facing semantics rather than the developer protocol, try `support.apple.com/guide/deployment/` instead.
