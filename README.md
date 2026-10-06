# WSNP — Web SNaPshot file format

The description of the `.wsnp` format: one web page, with every file it needs, kept in a single ZIP
to be read offline. The `.wsnpx` sibling carries the scripts of an application.

| File | What it is |
| --- | --- |
| [`FORMAT.md`](FORMAT.md) | The format, versions 1.0 and 1.1: container, manifest, paths, scripts, password protection, `.wsnpx`, validation checklist, signed manifest |
| [`MANIFEST-SIGNING.md`](MANIFEST-SIGNING.md) | Why and how the manifest is signed, and who controls the keys (the decision record behind section 12 of `FORMAT.md`) |

## Who uses it

| Program | Role |
| --- | --- |
| [PageKeep](https://github.com/asantos43/webpage-snapshot) | Browser extension that **writes** `.wsnp` files. It also holds the reference validator and the password-protection code (`tests/wsnp-check.mjs`, `tests/wsnp-crypt.mjs`) |
| [WSNP Viewer](https://github.com/asantos43/wsnp-viewer) | Desktop application that **reads** them |

Both link to this repository and keep no copy of the description. A change to the format is made
here first; the programs follow.

## License

[Mozilla Public License 2.0](LICENSE) (MPL-2.0).
