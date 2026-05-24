---
name: apple-mdm-expert
description: Apple MDM development reference covering the MDM protocol, Declarative Device Management (DDM), configuration profiles and payloads, enrollment flows (ADE/DEP, Account-Driven, OTA, SCEP/ACME), VPP and managed apps, and Apple Business/School Manager APIs. Routes any question to the right Apple developer doc page or the apple/device-management GitHub schema, and includes implementation patterns for the FleetLock stack (NanoMDM as architectural reference, TypeScript with @plist/plist, Cloudflare Workers/Durable Objects/D1/R2). Use this whenever building, debugging, or designing anything that talks to Apple MDM, even when the user does not say MDM explicitly. Relevant terms include MDM check-in, TokenUpdate, Authenticate, InstallApplication, configuration profile, mobileconfig, payload, DDM, declarative management, ADE/DEP, ABM/ASM token, SCEP, ACME, NanoMDM, MDM error codes, BootstrapToken, PushMagic, APNs MDM topic, managed apps, VPP license, User Enrollment, status report.
---

# Apple MDM Expert

You're helping a developer building and debugging Apple device management infrastructure. The user is the author of FleetLock — a Cloudflare-Workers-based MDM service for a Mac rental fleet, with Durable Objects for per-device command queues, D1 for enrollment state, R2 for profiles/binaries, and Cloudflare Containers running `micromdm/scep` for identity issuance. NanoMDM is the architectural reference, but the actual implementation is TypeScript using `@plist/plist`. Keep that context in mind when discussing trade-offs and code samples.

This skill is an **index and router**, not a re-host of Apple's documentation. Apple changes things every OS release, and reproducing their docs here would be both stale and a copyright problem. The job here is: route a question to the right authoritative source fast, surface the conceptual model that ties pieces together, and capture the gotchas that don't live anywhere in a single Apple page.

## The three canonical sources

Always prefer these over secondary references (blog posts, Stack Overflow, AI-summarized content).

1. **Apple developer docs** — `https://developer.apple.com/documentation/devicemanagement`. This is the human-facing conceptual layer plus the symbol reference. The page renders via JavaScript in a browser, so a direct `web_fetch` of the HTML URL returns an empty placeholder. The underlying markdown is fetchable at `https://developer.apple.com/tutorials/data/documentation/devicemanagement/[topic].md` (lowercase, hyphenated). Important: `web_fetch` only allows URLs that have appeared in a prior search or user message. To fetch a specific topic, run `web_search` first to surface the URL, then fetch. See `references/apple-docs-url-map.md` for the full URL atlas.

2. **`apple/device-management` on GitHub** — `https://github.com/apple/device-management`, `release` branch. This is the **authoritative machine-readable schema** for every MDM command, check-in request, profile, error code, declaration, and status item. It's versioned per OS release (current tag tracks iOS/macOS/tvOS/visionOS/watchOS 26.2). When you need exact field types, enums, or platform availability, this repo is the source of truth — not the human docs, which sometimes lag. Browseable layout:
   - `mdm/commands/` — every command's YAML schema
   - `mdm/checkin/` — check-in request schemas
   - `mdm/profiles/` — every payload type's schema
   - `mdm/errors/` — error code catalog
   - `declarative/declarations/` — DDM declaration schemas
   - `declarative/status/` — DDM status item schemas
   - `declarative/protocol/` — DDM protocol schemas
   - `other/` — misc data formats (e.g., Device Assignment service)

3. **Apple Support deployment guide** — `https://support.apple.com/guide/deployment/`. Admin-facing conceptual explanations that often clarify intent behind a feature when the developer docs are too terse. Useful for explaining behavior to non-MDM-engineer collaborators.

## How to navigate this skill

Start at **`references/index.md`** — that's the question-to-file router. Each entry maps a likely question or symptom to the right reference file (or directly to an Apple URL when the file would just be a thin wrapper).

The cheat sheets in `references/` are intentionally short. Each one:

- Names the concept and its real-world purpose (not just a definition)
- Lists the key symbols/commands/keys with one-line summaries
- Calls out the gotchas that bite people (and which payload version they appeared in)
- Links to the Apple doc URL and the GitHub schema path
- For FleetLock-relevant topics, includes a "FleetLock notes" section

When a question is too deep for a cheat sheet, the answer is "fetch the Apple `.md` form or read the GitHub YAML" — and the cheat sheet tells you exactly which one.

## Mental model: what MDM actually is, in three sentences

MDM is **a server-pushed JSON/XML control plane over APNs**. The server doesn't talk to a device directly — it sends a "wake up" push via Apple's APNs, the device then polls the server, the server hands it the next command in its queue, the device executes and reports back. Everything else (profiles, DDM, ADE, ABM) is a layer on top of that core polling loop plus the initial enrollment that gives the server permission to push to that device at all.

## When to consult this skill

Consult eagerly — undertriggering is the bigger risk than overtriggering. Apple MDM has dense, version-dependent semantics that are easy to get subtly wrong:

- `PushMagic` vs `Topic` vs `MDM Push Certificate` (three different identifiers, all needed)
- `UnlockToken` vs `BootstrapToken` vs `EscrowToken` (three different recovery mechanisms)
- Synchronous vs asynchronous commands (most are async; the device polls, you can't block on a response)
- `Status` reports (DDM) vs `CommandReport` (classic MDM) — entirely separate channels
- ADE vs User Enrollment vs Account-Driven Device Enrollment vs Account-Driven User Enrollment (four distinct flows, easy to confuse)
- SCEP vs ACME identity issuance (ACME-DM since iOS/macOS 16; SCEP is legacy but everywhere)
- `com.apple.configuration.*` (DDM declarations) vs `PayloadType` values (classic profile payloads) — they overlap in capability but are configured differently

If you see any of the following in a user's question, this skill is in play: a `.mobileconfig`, an `MDM` payload, a check-in request body, a command UUID/Status pair, a `com.apple.configuration.*` declaration, a SCEP/ACME challenge URL, an ABM/ASM service token, the `mdm.push.apple.com` topic, an error code like `12009` or `MCMDMTopicMismatch`, a `BootstrapToken`, an `EnrollmentProfile`, a `PushMagic`, a `TokenUpdate`, a managed app config, a VPP license assignment.

## When to stop and ask

This is enterprise software running on real devices. Ask the user before:

- Suggesting an irreversible operation (wipe, erase, lock, removing the MDM profile from a supervised-only configuration)
- Proposing a change to identity certificate issuance (you can lock devices out)
- Asserting platform availability for a command/declaration without checking the GitHub schema's `SupportedOS` block
- Writing production code that touches APNs without confirming the topic matches the MDM push certificate's `UID`

## FleetLock context

When the user is working in FleetLock code, also read `references/fleetlock-stack.md`. That file translates MDM concepts into the actual TypeScript / Workers / Durable Objects patterns the codebase uses and notes where NanoMDM's Go patterns diverge from what's idiomatic on Cloudflare.
