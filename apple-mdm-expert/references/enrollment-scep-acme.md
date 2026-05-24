# Identity cert issuance: SCEP and ACME

Every MDM-enrolled device needs an **identity certificate** that authenticates it to your MDM server. The server uses this cert to (a) verify check-in / command-poll signatures when `SignMessage: true`, and (b) optionally as the recipient for encrypted profiles. Two protocols issue these certs: SCEP (legacy, universal) and ACME (modern, device-attested, iOS/macOS 16+).

## SCEP (Simple Certificate Enrollment Protocol)

SCEP is the universal default. Every Apple platform supports it. The device contacts your SCEP server during enrollment with a challenge, receives a cert, and uses it from then on.

### The flow

1. Enrollment profile includes a `com.apple.security.scep` payload pointing at your SCEP server URL.
2. Device generates a key pair, builds a CSR (Certificate Signing Request) including the challenge.
3. Device POSTs the CSR to the SCEP URL.
4. SCEP server validates the challenge, signs the CSR, returns the certificate.
5. Device stores key + cert; MDM payload's `IdentityCertificateUUID` resolves to this cert.

### `com.apple.security.scep` payload — key fields

| Field | Purpose |
|---|---|
| `URL` | SCEP server endpoint |
| `Name` | CA identifier (NameValue in SCEP terminology). Often the issuing CA's name. |
| `Subject` | DN array of arrays — `[[ ["O", "MVS Collective"], ["CN", "$UDID"] ]]`. `$UDID` is a placeholder the device fills in. |
| `Challenge` | Pre-shared challenge string the SCEP server uses to authorize the request |
| `Keysize` | RSA modulus (2048 typical, 4096 for higher security) |
| `KeyType` | `RSA` (default) or `ECSECPrimeRandom` for ECC |
| `KeyUsage` | Bitmask: 1=signing, 4=encryption, 5=both |
| `Retries` | Retry count if SCEP server returns pending |
| `RetryDelay` | Seconds between retries |
| `CAFingerprint` | SHA-1 / SHA-256 of CA cert, for pinning. **Strongly recommended.** |

Full schema: `https://github.com/apple/device-management/blob/release/mdm/profiles/com.apple.security.scep.yaml`

### Subject DN — useful placeholders

`Subject` supports per-device interpolation:

| Placeholder | Replaced with |
|---|---|
| `$UDID` | Device UDID (or empty for User Enrollment) |
| `$SERIAL` | Device serial number |
| `$EMAIL` | User email (when known) |
| `$PROFILE_UUID` | The profile's PayloadUUID |
| Per-OS additional variables | Check the schema |

Common pattern: `CN=$UDID, O=MVS Collective, OU=FleetLock` so each device's cert is uniquely identified.

### SCEP server implementations

For FleetLock, the canonical implementation is `micromdm/scep` (Go). Run it in Cloudflare Containers, expose via Workers route, configure with your CA private key. The container is small and stateless beyond the CA key + serial counter.

Alternatives:
- `EJBCA` — full CA suite, complex to operate but enterprise-grade.
- `openxpki` — Perl-based open-source CA.
- AWS Private CA with the ACMP shim — not native SCEP, requires translation layer.

For ground truth on SCEP RFC, see RFC 8894. Apple's implementation follows RFC 8894 with some quirks around the challenge field.

## ACME for device management (iOS/macOS 16+)

ACME-DM is Apple's modernization of identity issuance: instead of a shared challenge string, the device proves its identity to the ACME server using **device attestation** (signed by Apple's hardware-rooted attestation key). This is much stronger evidence that the device is genuine Apple hardware, not a spoofed enrollment.

Apple doc: `https://developer.apple.com/documentation/devicemanagement/return_an_acme_server_directory_resource` (and surrounding pages)
Schema: `https://github.com/apple/device-management/blob/release/mdm/profiles/com.apple.security.acme.yaml`

### `com.apple.security.acme` payload

| Field | Purpose |
|---|---|
| `DirectoryURL` | ACME directory endpoint (root of your ACME service) |
| `ClientIdentifier` | Identifier scoping the cert to a specific use (e.g., "mdm-enrollment") |
| `HardwareBound` | Bind key to hardware (Secure Enclave). Highly recommended. |
| `KeySize`, `KeyType` | Like SCEP |
| `KeyUsage` | Like SCEP |
| `Subject` | Same DN array structure with same placeholders |
| `Attest` | If true, device performs hardware attestation as part of order |
| `Extensions` | X.509 extensions to include |

### When to choose ACME over SCEP

ACME wins when:
- You want hardware-attested identity (preventing simulator/spoofed enrollments)
- You're operating a modern fleet (iOS/macOS 16+ only — older devices need SCEP)
- You want standardized cert lifecycle (renewals, revocation) — ACME is better-defined

SCEP wins when:
- You need to support older devices (iOS 15 and earlier)
- You already have a working SCEP setup and have no reason to migrate

In a mixed fleet, you can support both — the enrollment profile chooses one based on device OS version (you'd serve a different profile per OS), or you can include both payloads and let the device pick (it will).

### ACME servers

ACME for device management is a profile on top of standard ACME (RFC 8555). Existing ACME server implementations (smallstep/certificates, step-ca, Boulder) can issue these certs but the device attestation extension is Apple-specific. As of writing, `smallstep/step-ca` has Apple device attestation support.

## Identity cert renewal

Identity certs eventually expire (typical 1-2 years). The device handles renewal automatically by re-running the SCEP/ACME exchange shortly before expiration, **if** the enrollment profile is still present.

If a device's identity cert expires without renewal (offline too long, etc.), the device's check-ins will fail signature verification and MDM is effectively lost. The recovery is re-enrolling the device.

Plan for renewal:
- Use a reasonable cert validity (1-2 years; not 10).
- Monitor cert expiration dates on your side as part of fleet health.
- For ADE devices, you can push a re-issued enrollment profile to refresh.

## Revocation

If you suspect a device has been compromised:

1. Mark the device record as revoked in your DB.
2. On the next command-poll signature verification, your server rejects the device — the bad actor with the stolen cert can no longer talk to MDM.
3. (Optional) Add the cert to a CRL or OCSP if you have one wired up. Apple's MDM client does not consult CRL/OCSP at runtime, so this is server-side only.
4. Re-issue an enrollment if the legitimate device returns.

## Gotchas

- **CAFingerprint matters.** Without it, the device trusts whatever cert the SCEP/ACME server returns. With it, the device validates the cert against the fingerprint. **Always set it.**
- **`HardwareBound: true` for ACME** binds the key to the Secure Enclave. Without it, the private key can be extracted from a backup. For production, set this.
- **Challenge strings in SCEP are not bearer credentials by design** but in practice they often are (the device sends the challenge and the server validates). Use single-use or short-lived challenges in dynamic enrollment — don't bake a permanent challenge into a public profile.
- **Cert request includes the public key**, not the private key. The private key never leaves the device.
- **SCEP servers should idempotent-handle re-issuance.** A device with the same `$UDID` re-running SCEP should get a fresh cert without errors. Some SCEP impls refuse duplicate DNs.
- **iOS occasionally re-runs SCEP unexpectedly** — e.g. after a system update or major iCloud event. Don't assume one cert per device for life.

## Quick links

- SCEP payload: `https://github.com/apple/device-management/blob/release/mdm/profiles/com.apple.security.scep.yaml`
- ACME payload: `https://github.com/apple/device-management/blob/release/mdm/profiles/com.apple.security.acme.yaml`
- micromdm/scep (reference implementation): `https://github.com/micromdm/scep`
- smallstep/certificates (modern CA with Apple ACME): `https://github.com/smallstep/certificates`
- RFC 8894 (SCEP): `https://www.rfc-editor.org/rfc/rfc8894`
- RFC 8555 (ACME): `https://www.rfc-editor.org/rfc/rfc8555`
