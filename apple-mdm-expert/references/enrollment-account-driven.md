# Account-Driven Enrollment

Account-driven enrollment is the **user-initiated** enrollment flow that uses a Managed Apple ID for identity. Two variants:

- **Account-Driven User Enrollment (ADUE)** — BYOD. Personal device, work data partitioned. The user signs into a Managed Apple ID under Settings → Sign in to Work or School Account.
- **Account-Driven Device Enrollment (ADDE)** — Organization-owned but enrolled via Managed Apple ID rather than ADE. Full device management. iOS 15+ / macOS 14+.

Apple doc: `https://developer.apple.com/documentation/devicemanagement/account-driven-enrollment`

## The user-facing flow

1. User opens Settings → General → VPN & Device Management (iOS) / System Settings → Privacy & Security → Profiles (macOS) and chooses "Sign in to Work or School Account" / equivalent.
2. User enters their Managed Apple ID (e.g., `dustin@example.com`).
3. iOS/macOS extracts the domain (`example.com`) and queries `https://example.com/.well-known/com.apple.remotemanagement` for service discovery.
4. Server returns a JSON document indicating which enrollment types it supports and the URL to start each.
5. Device contacts the start URL, gets an OAuth2 auth request.
6. User authenticates via OAuth2 (likely your IdP).
7. After successful auth, device fetches the MDM enrollment profile and proceeds with identity cert issuance + enrollment.

## Service discovery: `.well-known/com.apple.remotemanagement`

This is the entry point for account-driven enrollment. The domain in the Managed Apple ID dictates where the device looks; you must host this endpoint on the domain you use for Managed Apple IDs.

Response shape (JSON):

```json
{
  "Servers": [
    {
      "Version": "mdm-byod",
      "BaseURL": "https://mdm.example.com/enroll/byod"
    },
    {
      "Version": "mdm-adde",
      "BaseURL": "https://mdm.example.com/enroll/adde"
    }
  ]
}
```

`Version` values:
- `mdm-byod` — Account-Driven User Enrollment
- `mdm-adde` — Account-Driven Device Enrollment

You can advertise one or both. The device picks the flow.

## The OAuth2 user-enrollment flow

After service discovery, the device hits `BaseURL` and the server returns an OAuth2 authorization challenge. The relevant Apple doc is "Implementing the OAuth2 Authentication User Enrollment Flow" (linked from the account-driven-enrollment overview).

Key points:

- Standard OAuth2 authorization code flow with PKCE.
- The device opens a browser tab to your authorization endpoint, with `redirect_uri` set to `apple-remotemanagement-user-login://authentication-results` (a custom scheme the device intercepts).
- After successful auth, server redirects with `?code=...`.
- Device exchanges code at your token endpoint for an access token.
- Device sends a follow-up request to the enrollment service with `Authorization: Bearer [token]`.
- Server returns the MDM enrollment profile.

The OAuth2 piece is **your problem to implement**, typically by proxying to your existing IdP (Okta, Azure AD, Google, custom). The user's Managed Apple ID is the audience but the actual IdP handshake is opaque to Apple.

## ADUE: what's different from full MDM

User Enrollment is intentionally limited. It can:

- Install managed apps (separately from personal apps)
- Set up a managed Mail / Calendar / VPN
- Apply per-app VPN
- Set a passcode policy on the work partition
- Be remotely removed by the org, taking only work data with it

It cannot:

- Read or modify personal apps or data
- Access UDID, IMEI, serial number (privacy)
- Wipe the entire device — only `EraseDevice` with `PreserveDataPlan: true` semantics, or selective work-data wipe
- Configure system-level features (Wi-Fi for the whole device, etc.)
- Use commands marked `RequiresSupervision: true`

The device sends `EnrollmentID` (a UUID-shaped opaque string) instead of `UDID`. All check-in and command identity is by `EnrollmentID`.

## ADDE: what's different from ADE

ADDE feels like ADE without being purchased through ABM/ASM. The device is fully managed (supervised, even), but the user initiated enrollment by signing in.

- Device is fully managed and (typically) supervised.
- `UDID` is present and usable.
- No `AwaitingConfiguration` window because there's no Setup Assistant — the user is already past it. You configure on the side.
- Activation Lock and other supervision features apply.

ADDE is the modern path for "I want full management without buying through ABM" — useful for non-Apple-channel hardware (e.g., used devices), or for orgs that don't use ABM/ASM at all.

## Managed Apple ID

The identifier itself. Provisioned in ABM/ASM (or federated with Microsoft Entra ID / Google Workspace). Users can use it to:

- Sign in for work-account enrollment
- Access org-purchased apps and books (instead of consuming a personal Apple ID's purchases)
- Use iCloud Drive in a managed work space (with org policy)

The federation flow lets you skip provisioning Managed Apple IDs manually; Microsoft / Google identities are reused.

## Quick links

- Service discovery format: `https://developer.apple.com/documentation/devicemanagement/account-driven-enrollment`
- OAuth2 flow detail: `https://developer.apple.com/documentation/devicemanagement/implementing-the-oauth2-authentication-user-enrollment-flow`
- MDM payload schema (for the eventual enrollment profile): `https://github.com/apple/device-management/blob/release/mdm/profiles/com.apple.mdm.yaml`

## Gotchas

- **`.well-known` must be on the exact domain in the Managed Apple ID.** If the user is `dustin@example.com`, you need `https://example.com/.well-known/com.apple.remotemanagement` — not `mdm.example.com`. Add the route to the root domain.
- **HTTPS only**, modern TLS, valid CA-issued cert (no self-signed). The device validates aggressively at this stage.
- **The custom-scheme redirect `apple-remotemanagement-user-login://`** is intercepted by iOS/macOS automatically. Don't try to register it yourself.
- **ADUE and ADDE share `.well-known`** but the flows diverge after `BaseURL` is hit. Tag the flow on your side based on which `Version` value the device announced (the device echoes it in the request).
- **For ADDE, you cannot rely on `AwaitingConfiguration`** to pause for setup. Push initial config commands immediately after `TokenUpdate` arrives; the user is already at the Home Screen.
- **Managed Apple ID federation requires DNS verification** of the user-facing domain on the ABM/ASM side. Plan for the ABM/ASM DNS TXT challenge.
- **Removing User Enrollment** doesn't trigger a `CheckOut` even with `CheckOutWhenRemoved: true` in some OS versions. Verify the behavior on your target OS.
