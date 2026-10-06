# Protecting the metadata: a signed manifest (proposal)

**Status: decided and built.** A key per installation, trusted on first use, told by the user. The format is in [`FORMAT.md`](FORMAT.md) section 12 (1.1). The viewer reads it, and PageKeep (the extension that writes the files)
signs every `.wsnp` it writes, from the release that follows this design: a file written before it is *unsigned*, and the viewer says so, quietly, in the metadata view and the status bar. The viewer itself does not sign yet
(that comes with conversion).

## The problem

A `.wsnp` is a ZIP, so anyone can unzip it, edit `manifest.json` and zip it again. The manifest lists the SHA-256 and the size of every other file, which protects the files. Nothing protects the manifest itself.

| What someone edits | Caught today? |
| --- | --- |
| The bytes of a file (a page, a stylesheet, a picture) | Yes: the SHA-256 in the manifest no longer matches. The snapshot is **not valid**. |
| The size or the SHA-256 of a file in the manifest | Yes: it no longer matches the file. The snapshot is not valid, or (a wrong size) does not open. |
| The title, the address the page came from, the date, the description, the viewport | **No.** Nothing in the file contradicts the edit. |
| Everything, with the hashes recomputed to match | **No.** Nothing in the file is secret, so whoever edits can redo all of it. |

Only a **signature** closes this: a proof that can be made only by someone who holds a secret that is not in the file.

## The key: who controls it

- **Each writer has a key pair of its own** (Ed25519, or ECDSA P-256 where Ed25519 is missing; the file says which). PageKeep makes it the first time it is used, as a **non-extractable** Web Crypto
  key kept in the extension's storage: the extension can ask it to sign, but its bytes cannot be read out or copied. The viewer has its own key (kept with Electron's `safeStorage`, in the operating system's
  keychain) for the files it makes: a converted ZIP, a protected copy.
- **The public key goes in the file**, with its fingerprint (the SHA-256 of the public key, shown as groups of hex: `3F2A-91C0-…`). Anyone can check the signature with it.
- **There is no central authority** and nothing to revoke. A signature says: *a key with this fingerprint signed exactly this manifest*. It does **not** say who the person is, nor that the page was true when it
  was captured.
- **The viewer trusts on first use**, as SSH does, but **the user says when**. It keeps the fingerprints the user chose to trust, in `trusted-signers.json` in the user's own profile (with an optional name, for example "PageKeep on this
  computer"). A file signed by a key on the list shows "signed by" that name; a key that is not shows "signed by a key this viewer does not know yet", with the fingerprint and a button **Trust this signer**. Nothing is trusted by itself.
- **Reinstalling the extension makes a new key.** Old files still verify (their public key is inside); their signer just looks new, and the user can tell the viewer that both keys are theirs.
- **Forging.** Someone can unzip, edit, and sign again with *their own* key: the file verifies, with another fingerprint. That is why the viewer shows the signer and warns when it is not one the user knows. A file
  received from another person is only as trustworthy as the fingerprint the user can confirm with them, out of band. The protection is **against editing**, not against a dishonest writer.
- **What it does not cover:** malware that can drive the extension to sign anything, or a person who signs a page they faked before capturing it.

## The format (a 1.1 of FORMAT.md; the rule is in section 12 there)

`signature.json` at the root, beside `manifest.json`, **not listed in `files`** (it cannot be: it signs the manifest that would list it):

```json
{
  "signature_version": "1.0",
  "algorithm": "Ed25519",
  "public_key": "<32 bytes, base64>",
  "signed": "manifest.json",
  "manifest_sha256": "<hex>",
  "signature": "<64 bytes, base64>"
}
```

The signature is over the exact bytes of `manifest.json` as stored. Because the manifest has the SHA-256 of every other file, the signature covers the whole archive. In a protected file (`FORMAT.md` section 9) the
signature is inside the encrypted content, part of the open file.

Step 7 of the checklist changes from "every entry but `mimetype` and `manifest.json` is listed" to "…but `mimetype`, `manifest.json` and `signature.json`". A reader that follows 1.0 to the letter would refuse a signed
file, which is why it is a new minor version, and why this viewer must read it before PageKeep writes it.

## What the viewer does (as built)

| The file | The viewer says |
| --- | --- |
| No `signature.json` (every file made so far) | Opens, with a quiet notice: **not signed, the metadata is not protected.** |
| Signed, the signature checks, a key the user trusts | "Signed by" the name the user gave it, with the fingerprint. "Stop trusting" is one click. |
| Signed, the signature checks, a key the user has not said to trust | "Signed by a key this viewer does not know yet", with the fingerprint, the method and a button **Trust this signer**. |
| Signed, the signature does **not** check (the manifest or the signature was edited) | **Not valid**, held back like a file whose contents changed, with "Show Anyway". |

## What has to change

- **PageKeep** (`page-snapshot-extension`): make and keep the key pair; sign `manifest.json` after writing it and add `signature.json`; write `format_version` `"1.1"`; show the fingerprint (popup or options) so the user can
  tell the viewer which key is theirs; tests that a signed file verifies and an edited one does not.
- **The viewer** (done, except the last item): `core/validate/signature.ts` (verify), the step-7 change, the list of trusted signers (`core/signers.ts`), the signature row of the metadata view, the status bar item, the held-back page, tests with signed,
  unsigned and edited files (`fixtures/sign.ts` is the writer's side). **Not yet:** signing what the viewer writes (a converted ZIP, a protected copy), which comes with conversion (phase 2).
- **FORMAT.md** (1.1, done), **PRIVACY.md** (a fingerprint list is kept in the profile) and **SECURITY.md** (the model above, including what it does not cover).

## Alternatives that were rejected

- **A hash of the manifest kept in the file.** Whoever edits the manifest recomputes it. It catches accidents only.
- **A shared secret (HMAC) built into the writer and the viewer.** The secret can be taken out of either program, so it stops only casual edits and gives a false sense of safety.
- **One key for all of PageKeep's users.** It would have to ship in every copy of the extension, where anyone can read it.
- **Signing by an authority** (certificates). Needs an organisation to run it and every user to hold a certificate: far more than the format needs.

## Open questions

- Ed25519 in Web Crypto needs a recent Chrome; is ECDSA P-256 as the fallback acceptable, or should it be the only algorithm?
- Should the viewer offer to sign a file the user opens (taking ownership of a `.wsnp` from someone else)?
