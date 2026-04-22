# Adriatic Extensions Registry

The official catalog of [Adriatic](https://github.com/stelath/Meriggiare-Apps) extensions.

[Adriatic](https://github.com/stelath/Meriggiare-Apps) is a remote-desktop application purpose-built for iPhone-to-Mac connections. **Extensions** are macOS apps that register with AdriaticHost over a Unix Domain Socket and pipe rich, server-driven UIs to the iOS client. The protocol is open: any macOS app that speaks the wire format and emits an `RVL` (Remote View Language) tree can be an extension.

This repo is the discovery + distribution catalog for those extensions. AdriaticHost reads `index.json` to populate its in-app browser; users one-click install from there.

## What's here

```
adriatic-extensions/
├── SPEC.md                          # wire protocol + manifest format + contribution rules
├── schema/extension-v1.schema.json  # JSON Schema for `extension.json` validation
├── extensions/
│   └── <reverse-dns-id>/
│       ├── extension.json           # source of truth (PR-edited)
│       ├── icon.png                 # optional, 256×256 or 512×512 (iOS uses iconSymbol)
│       └── screenshots/*.png        # optional, rendered in catalog
├── index.json                       # CI-generated flat catalog (what AdriaticHost reads)
└── .github/workflows/
    ├── validate.yml                 # schema + URL reachability + checksum on PR
    └── publish-index.yml            # regenerate index.json on merge to main
```

## Submit an extension

1. Build a macOS app that links `AdriaticCore` and registers as an extension. See [SPEC.md](./SPEC.md) for the three contribution paths (standalone app, library-mode add-on, non-Swift).
2. Sign + notarize a release and publish it on GitHub Releases.
3. Open a PR to this repo with a folder under `extensions/<your-reverse-dns-id>/` containing `extension.json`, `icon.png`, and any screenshots.
4. CI validates the manifest against the schema; a maintainer reviews ownership of the `id` prefix.

The full submission process and trust model live in [SPEC.md](./SPEC.md).

## License

Registry contents (this repo's source) are MIT-licensed. Each individual extension carries its own license declared in its `extension.json`.
