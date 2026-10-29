# Proposal: faster dataset upload

Proposed changes to the TANUH dataset upload, kept here for review and future reference. They are **not merged** into `ui-dx` or `files-connect-api`.

They come from the October 2026 upload benchmark. On a 100 Mbit/s link, TANUH took 24.9 min to upload 1,000 small files with encryption, against 2.0 min for Google Drive. Most of the time went to requests made for every file, not to sending data.

## What changes

| # | Change | Where |
|---|---|---|
| 1 | The file list is no longer refreshed after every file (its response was never used). | `ui-dx`: `upload.component.ts`, `manage-files-modal.component.ts` |
| 2 | 6 files upload at the same time instead of one. | `ui-dx`: new `dataset-upload.service.ts` |
| 3 | Encryption: one random key per upload session, wrapped once by KMS. **One manifest per session**, written after the last file, replaces the per-file `<file>.manifest.json`. | `ui-dx`: `lib/encrypted-dataset-upload.ts`, `encrypted-upload.service.ts` |
| 4 | **No plaintext hash**: AES-GCM tags already detect any change. The base IV is in a 36-byte header at the start of each file, and chunk *i* uses base IV + *i*. | `ui-dx`: `lib/encrypted-dataset-upload.ts` |
| 5 | Large files: 3 parts of a file upload at once, and the next part is encrypted meanwhile. | `ui-dx`: `lib/encrypted-dataset-upload.ts` |
| 6 | Small files: initiated 20 per request, as direct uploads (one presigned PUT to the final object, no complete call). | `files-connect-api`: `POST /databanks/:id/uploads/batch` and `direct` on initiate. `ui-dx`: `dataset-upload.service.ts`, `file-api.service.ts` |
| 7 | Preview of an unknown file type no longer reads the whole object into memory. It used to OOM-kill the pod on 2 GiB `.bin` files. | `files-connect-api`: `databanks-routes.ts` |

The encrypted format (`tanuh-enc-dataset-v2`) is described at the top of `lib/encrypted-dataset-upload.ts` in `ui-dx.diff`:

```
object  = header | chunk 0 ciphertext+tag | chunk 1 ciphertext+tag | ...
header  = "TNHE" | version 2 | 3 zero bytes | session id (16) | base IV (12)     (36 bytes)
IV(i)   = base IV + i                       (96-bit big-endian)
AAD(i)  = header | uint32be(i) | uint32be(total chunks)
manifest: _tanuh/manifest-<session id>.json = wrapped key, chunk size, and per file: key, base IV, size, chunk count
```

## Files

| File | Diff of | Base |
|---|---|---|
| [`files-connect-api.diff`](files-connect-api.diff) | `files-connect-api` | `tanuh-gcp/v2.3` at `18f22b0b238b313b9d85fbe90558c3013ad50dfe` |
| [`ui-dx.diff`](ui-dx.diff) | `ui-dx` (TANUH app and the shared `FileApiService`) | `development` at `64b4576b9fdef1a617dcf430ae055135de03b38c`, after running Prettier on the touched files |

The original branches are `feat/faster-dataset-upload` on `prathmeshj1729/files-connect-api` and `prathmeshj1729/ui-dx`.

To apply one, check out its base commit and run `git apply <file>.diff`. For `ui-dx`, run `npx prettier --write` on the files the diff touches first, because the diff was taken after formatting them.

## Before using it

- **Deploy order:** the UI needs the file-server change. Against today's server, batch initiate returns 404.
- **The Processing TEE is not updated.** It still expects the old format (per-file manifest with a SHA-256), so it cannot decrypt datasets uploaded with this change until it learns `tanuh-enc-dataset-v2`.
- **Files without a manifest can't be decrypted.** If the tab closes before the session manifest is written, that session's files cannot be decrypted. This is the same "not finalized" rule as before.
- **Not tested end to end.** Both branches type-check and the UI builds. The encryption was round-tripped against an independent Node.js decrypter, including tampering cases, but the whole flow has not run against a deployed server.
