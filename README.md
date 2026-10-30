# SuperYar registry

Public, versioned, **signed distribution** for [SuperYar](https://superyar.ir)'s retailer API
definitions. The Android app reads from here, so a retailer's API change — or a whole new retailer —
ships **without a new app release**: the app fetches the latest signed manifest and updates its
on-device definitions.

This repo contains **only what the app downloads** — nothing else. It is never contacted per retailer
request, never proxies traffic, **holds no secrets**, and has no per-user data (ARD 05 §1).

> **Generated distribution.** The source of truth — definitions, fixtures, JSON Schemas, the engine,
> tests, and the build/sign tooling — lives in the private SuperYar monorepo. Maintainers validate
> there and publish the clean, signed subset here. Don't open content PRs against this repo; it is
> regenerated on each release.

> **Status: pre-launch.** Every retailer is `compliance.basis = pending-review` — no legal clearance
> (CG-1) and no permission yet. These definitions are not activated in a shipping app until that
> clears. See [SECURITY.md](SECURITY.md) and each `retailer.json`.

## What's here (all of it is downloaded or pinned)

```
channels/
  stable/manifest.json(.sig)   what all users get        (Ed25519-signed)
  beta/manifest.json(.sig)     opt-in from Diagnostics    (Ed25519-signed)
retailers/<id>/
  retailer.json                hosts, budgets, errors, capabilities, compliance, auth
  operations/*.json            one capability per file (file name = operation id)
tables/*.json                  mapping tables (unit aliases, …) referenced by the manifest
keys/<keyId>.pub.*             Ed25519 public key (raw 32 bytes, base64) — pinned in the APK
```

Fixtures, schemas and tooling are intentionally **not** here — they are CI/audit/build artifacts that
the app never downloads, and they live in the source monorepo.

## How the app consumes it (ARD 05 §6)

1. `GET channels/<channel>/manifest.json` (conditional, `If-None-Match`) + `manifest.json.sig`.
2. Verify the Ed25519 signature against the pinned public key; reject on failure.
3. Check `sequence` (monotonic per channel; lower = rejected = rollback protection), `expiresAt`,
   and `revokedKeyIds`. Apply `killSwitches`, `revoked` and `params` immediately.
4. For each changed bundle whose `requiresEngine ≤ ENGINE_LEVEL`, fetch the listed files **raw at the
   release tag** (immutable), verify each `sha256`, stage, validate, then activate atomically.

Integrity comes from the signed manifest, not from TLS. Files are fetched per-file at an immutable tag
rather than as release zips — a documented deviation from ARD 05 §2, chosen for simplicity.

## Versioning (ARD 05 §4)

| Thing | Scheme |
|---|---|
| Definition bundle | `YYYY.MM.DD-n` per retailer (`retailer.json → version`) |
| Manifest | integer `sequence`, monotonic **per channel**, bumped only when content changes |
| SDL language | `sdl` major.minor in each document |
| Engine | integer `requiresEngine`; the app compares against its compiled `ENGINE_LEVEL` |

Each release is a git tag `r<sequence>` (e.g. `r2`); the manifest's files are read at that tag.

## Verify a manifest yourself

```bash
# raw 32-byte Ed25519 public key is in keys/<keyId>.pub.b64 (keyId is in the manifest)
node -e 'const {createPublicKey,verify}=require("crypto"),fs=require("fs");
 const m=fs.readFileSync("channels/stable/manifest.json");
 const raw=Buffer.from(fs.readFileSync("keys/"+JSON.parse(m).keyId+".pub.b64","utf8").trim(),"base64");
 const k=createPublicKey({key:{kty:"OKP",crv:"Ed25519",x:raw.toString("base64url")},format:"jwk"});
 console.log(verify(null,m,k,Buffer.from(fs.readFileSync("channels/stable/manifest.json.sig","utf8").trim(),"base64")));'
```

## License

Apache-2.0 for the schemas and tooling (in the source repo); mapping tables (`tables/`) are dedicated
to the public domain under CC0-1.0. See [LICENSE](LICENSE).
