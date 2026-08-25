# crate-bundles

Distribution point for the third-party command-line binaries that the **Crate**
macOS app relies on (`yt-dlp`, `ffmpeg`), so they can be updated over the air
without shipping a new build of the app.

Two things live here, and nothing else:

| What | Where |
| --- | --- |
| `manifest.json` — the signed index of available binaries | this repo, on `main` |
| the binaries themselves | GitHub **Release assets**, tag `binaries-<YYYY-MM-DD>` |

This repository is public on purpose: installed copies of Crate fetch the
manifest and the release assets unauthenticated. No source, credentials, or
build tooling belong here.

---

## `manifest.json` schema

`schema_version` is currently `1`.

```jsonc
{
  "binaries": {
    "<name>": {                 // e.g. "yt-dlp", "ffmpeg"
      "version": "2026.03.17",  // string; compared against the copy the app already has
      "url":     "https://github.com/.../releases/download/<tag>/<name>",
      "sha256":  "<64 hex chars>"  // of the file at `url`
    }
  },
  "schema_version": 1,
  "updated_at": "2026-04-28T08:50:00Z",  // ISO 8601, UTC
  "signature": "<base64 Ed25519 signature>"
}
```

What the fields are for: the app verifies `signature`, then for each binary
compares `version` against the copy it already has; if the manifest is newer it
downloads `url` and refuses to install unless the bytes hash to `sha256`. The
app's binary updater is the authoritative consumer of this file — if the two
ever disagree, the app wins and this document is wrong.

Keep `main` in a valid, correctly signed state at all times. There is no
staging branch; whatever is on `main` is what installed copies read.

---

## Signature scheme

**Ed25519**, detached, base64-encoded, stored in the manifest's own `signature`
field.

The signed message is the manifest **with the `signature` key removed**,
serialized as **compact JSON** — `,` and `:` separators, no whitespace — with
**keys sorted lexicographically**, encoded UTF-8. No trailing newline.

The public key is embedded in the Crate app; it is not stored in this repo, so
the app remains the single source of truth for what is trusted.

### Verifying a manifest

```python
import base64, json

manifest = json.load(open("manifest.json"))
sig = base64.b64decode(manifest.pop("signature"))
signed_bytes = json.dumps(manifest, sort_keys=True, separators=(",", ":")).encode()
# then: Ed25519 verify (sig, signed_bytes, <public key embedded in the Crate app>)
```

Two consequences worth knowing:

- **Reformatting the file is safe.** The signature covers the canonical
  re-serialization, not the bytes on disk, so indentation and key order *in the
  file* are cosmetic.
- **Any value change invalidates the signature** — including bumping
  `updated_at` or correcting a typo in a URL. Every edit needs a re-sign.

---

## Releases

Binaries are uploaded as assets on a dated tag, one tag per snapshot:

- `binaries-2026-04-28` — `ffmpeg` 6.0, `yt-dlp` 2026.03.17

Publish a new tag rather than replacing assets on an existing one. Old tags must
keep working: copies of Crate that have not yet fetched a newer manifest are
still pointing at them.

---

## Updating a binary

The signing key and the script that wraps these steps live in the **private
Crate repository**, not here. In outline:

1. Upload the new binary as an asset on a **new** `binaries-<YYYY-MM-DD>` tag.
2. Record its `sha256` and version in `manifest.json`, and set `updated_at`.
3. Re-sign the manifest with the private signing key and write the new
   `signature` value.
4. Verify before pushing — recompute the canonical bytes and check the signature
   against the public key the shipped app carries. An unverifiable manifest on
   `main` breaks updates for every installed copy.
5. Commit and push to `main`.

The private signing key must never be committed to this repository or pasted
into an issue, PR, or commit message. Rotating it means shipping a new build of
the app, because the public key travels inside the app.
