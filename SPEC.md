# Adriatic Extensions — Specification v1

This document defines the manifest format, wire protocol, and contribution rules for the Adriatic extensions registry.

---

## 1. What an extension is

An **Adriatic extension** is a macOS app that:

1. Links the [`AdriaticCore`](https://github.com/stelath/Meriggiare-Apps/tree/main/Adriatic/AdriaticCore) Swift package.
2. Connects to the AdriaticHost Unix Domain Socket at `~/Library/Application Support/Adriatic/host.sock`.
3. Registers an `ExtensionManifest` declaring its id, display name, and capabilities.
4. Emits a server-driven SwiftUI tree (RVL) in response to `rvl.open` messages from the iOS client.

Extensions are full Mac apps, not in-process plugins. The UDS boundary gives crash isolation, language flexibility, and respects the iOS App Store's prohibition on downloading executable code (clients only ever receive a JSON UI tree, never a binary).

## 2. The wire protocol

Newline-delimited JSON `ExtensionFrame`s flow over the UDS in both directions. The host wraps outgoing frames in an `ExtensionEnvelope` and forwards them through the iOS client's data channel. Inside that envelope, RVL messages use these `innerType`s:

| Direction | `innerType` | Payload | Purpose |
|---|---|---|---|
| client → ext | `rvl.open` | `{ viewId }` | User opened a view. |
| ext → client | `rvl.set` | `{ viewId, tree, rvlVersion, title? }` | Replace the tree for a view. |
| client → ext | `rvl.event` | `{ viewId, action, state?, payload? }` | User triggered an action. |
| ext → client | `rvl.navigate` | `{ operation, destination? }` | Push / pop / replace navigation. |
| client → ext | `rvl.close` | `{ viewId }` | View was dismissed. |

The `tree` is a `RemoteView` value: a tagged-union JSON tree of SwiftUI-shaped components. See `RemoteView.swift` in `AdriaticCore` for the full v1 vocabulary (~21 components covering layout, display, interactive controls, lists, and navigation).

**Forward compatibility:** unknown component names decode to `unknown(type:)` on the client and render as a placeholder. Add new components freely in v2; older clients won't crash.

## 3. The manifest (`extension.json`)

Each entry in `extensions/<id>/` carries an `extension.json` validated against [`schema/extension-v1.schema.json`](./schema/extension-v1.schema.json).

```json
{
  "$schema": "https://raw.githubusercontent.com/stelath/adriatic-extensions/main/schema/extension-v1.schema.json",
  "id": "com.example.myextension",
  "displayName": "My Extension",
  "tagline": "One-line elevator pitch.",
  "summary": "Longer 1–2 paragraph description shown on the detail screen.",
  "iconSymbol": "puzzlepiece.extension",
  "messagePrefix": "myext.",
  "author": { "name": "Your Name", "url": "https://example.com" },
  "repository": "https://github.com/you/your-repo",
  "homepage": "https://example.com/myextension",
  "license": "MIT",
  "minHostVersion": "1.0.0",
  "rvlVersion": 1,
  "capabilities": [],
  "releases": {
    "macos": {
      "version": "1.0.0",
      "url": "https://github.com/you/your-repo/releases/download/v1.0.0/MyExtension.dmg",
      "sha256": "abc123…",
      "minOS": "13.0",
      "notarized": true,
      "teamId": "XXXXXXXXXX"
    }
  }
}
```

**Required fields:** `id`, `displayName`, `iconSymbol`, `messagePrefix`, `author`, `license`, `minHostVersion`, `rvlVersion`.

**Optional `releases`:** an entry without `releases` is treated as **source-only** — AdriaticHost will surface a "Build from source" link to your repository instead of an Install button. This is the right shape for early-stage extensions or for projects that prefer not to distribute prebuilt binaries.

### Field reference

| Field | Type | Notes |
|---|---|---|
| `id` | string | Reverse-DNS, must match what your app registers on the UDS. Becomes the folder name under `extensions/`. |
| `displayName` | string | Shown in the iOS client's tile and as the default nav title. |
| `tagline` | string | One-liner shown under the displayName in the catalog. |
| `summary` | string | 1–2 paragraph description for the detail screen. Markdown not supported in v1. |
| `iconSymbol` | string | An [SF Symbol](https://developer.apple.com/sf-symbols/) name. Required because the iOS client renders icons from system symbols (App Store §2.5.2 — no network image fetching in v1). |
| `messagePrefix` | string | Reserved namespace for the extension's wire messages. Must end with `.`. |
| `author` | object | `{ name, url }`. URL is optional. |
| `repository` | string | URL to the source repo. Mandatory if `releases` is omitted (so users can build from source). |
| `homepage` | string | Optional product page. |
| `license` | string | SPDX identifier (e.g. `MIT`, `Apache-2.0`, `GPL-3.0`). |
| `minHostVersion` | string | Semver string. AdriaticHost refuses to install if its version is lower. |
| `rvlVersion` | int | Required RVL version. v1 today. |
| `capabilities` | string[] | Optional forward-compat feature flags (e.g. `"fullscreen-view"`). Unknown values are ignored. |
| `releases.macos` | object | If present, must include `version`, `url`, `sha256`. `notarized: true` and `teamId` are required for the Verified tier (see below). |

## 4. Trust model

Two implicit tiers:

**Verified** — the manifest declares `notarized: true` and a `teamId`. AdriaticHost downloads the binary, verifies SHA-256, runs `spctl --assess`, and confirms the codesigning Team ID matches the manifest. One-click install with no warnings.

**Unverified** — manifest omits `notarized` / `teamId`, or `spctl` rejects. AdriaticHost shows a "this extension is not notarized" warning and requires Dev Mode to be on before installing.

**Dev mode (sideload)** — running any UDS-conforming Mac app outside of any registry. Always works; not subject to registry rules. This is how extension authors iterate.

The registry maintainer reviews each PR for ownership: the `id`'s reverse-DNS prefix should plausibly belong to the submitter (a domain they own, or their GitHub org). Repeat edits to an entry must come from the same author.

## 5. Contribution paths

### Path A — Standalone extension app

Build a new Mac app that links `AdriaticCore`. The minimal entry point:

```swift
import AdriaticCore

let manifest = ExtensionManifest(
    id: "com.example.myext",
    displayName: "My Extension",
    iconSymbol: "puzzlepiece.extension",
    messagePrefix: "myext.",
    rvlVersion: 1
)

let client = ExtensionClient()
client.onMessage = { message in
    // Handle rvl.open / rvl.event / rvl.close
    // Reply with rvl.set carrying a RemoteView tree.
}
client.connect(manifest: manifest)
```

Notarize via Xcode → Organizer, publish a GitHub release, open a PR to this repo with your manifest folder.

### Path B — Add extension support to an existing Mac app

Same code, just add `AdriaticCore` as an SPM dependency to your existing project and instantiate `ExtensionClient` alongside your normal app logic. Your app stays usable on its own; when AdriaticHost is running, it also becomes an iOS-accessible extension.

### Path C — Non-Swift extension

Anything that can speak newline-delimited JSON over a Unix Domain Socket can be an extension. Connect to `~/Library/Application Support/Adriatic/host.sock`, send a `register` frame with your manifest, then exchange `toClient` / `fromClient` frames carrying RVL JSON directly. No Swift required.

## 6. Submission checklist

- [ ] Manifest validates against `schema/extension-v1.schema.json`.
- [ ] `id`'s reverse-DNS prefix is one you own (domain or GitHub org).
- [ ] If you provided a `releases.macos.url`, the file is reachable and its SHA-256 matches.
- [ ] If you marked `notarized: true`, `spctl --assess --type install <App>.app` passes.
- [ ] `iconSymbol` is a real SF Symbol name (`SF Symbols.app` to verify).
- [ ] `summary` is honest about scope, capabilities, and any data the extension sends off-device.
- [ ] License is declared.

CI runs the first four checks automatically on PR.
