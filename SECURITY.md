# Security policy

## What protects users

The registry is untrusted infrastructure by design (ARD 05 §5, §9). Integrity does **not** rely on
GitHub or TLS:

- **Signed manifests.** Each `channels/<ch>/manifest.json` is signed with an offline Ed25519 key
  (detached `manifest.json.sig`, base64 over the exact manifest bytes). The app pins the public key
  and rejects anything that does not verify. Bundle integrity comes from the `sha256` recorded in the
  signed manifest, so individual files need no separate signatures.
- **Host pinning.** Independently of signatures, the APK pins the allowed **host suffixes per
  retailer id**. Even a validly signed malicious definition cannot send requests — or user locations —
  to a host outside them.
- **A constrained engine.** Definitions are declarative. The engine forbids payment/checkout/order
  paths, never lets a definition set `Authorization`/`Cookie`, and injects the user session itself.
- **Rollback & freeze protection.** `sequence` is monotonic per channel (a lower sequence is
  rejected); `revoked` / `revokedKeyIds` deactivate bad bundles or a compromised key; `expiresAt`
  surfaces a stale manifest without disabling the app.

## The signing key

- **Key id:** `reg-2026-a`. Public key (raw 32-byte Ed25519, base64) is in `keys/reg-2026-a.pub.b64`
  and pinned in the app.
- The **private key is never committed** and never placed in GitHub Actions secrets for the `stable`
  channel (`keys/private/` is git-ignored). Maintainers hold it offline / in a hardware token.
- **Signing procedure (stable):** done in the source monorepo, not here. A maintainer rebuilds the
  manifest reproducibly from the definitions, confirms the `sha256` values, reviews the diff of every
  `retailer.json` `hosts`/`auth`, signs with the offline key, and publishes the clean signed subset to
  this repo. A compromised CI cannot get a manifest signed, because CI never holds the stable key.
- **Rotation:** publish a manifest signed with the standby key, ship an app version that pins the next
  standby, then revoke the old key via `revokedKeyIds` in a manifest signed by the standby.

> **Bootstrap note.** `reg-2026-a` was generated to make this repo verifiable from day one. Before any
> public launch, rotate to a key generated and stored entirely offline (hardware token), using the
> rotation steps above, and add it to the APK's pinned set.

## Reporting a vulnerability

Email **security@superyar.ir** (or open a private security advisory on GitHub). Please do not open a
public issue for anything exploitable. We aim to acknowledge within a few days. Include the affected
file(s), the manifest `sequence`/`keyId`, and a minimal reproduction.

## Fixture hygiene

Fixtures must be trimmed to the minimum fields and carry **no** tokens, cookies, session ids, phone
numbers, or other personal data. CI scans for token/secret-shaped content and literal
`Authorization`/`Cookie` headers and fails the PR.
