---
topic_id: "v2:DMKO"
topic_path: "model-runners"
semantic_id: "3Y1bPE4V_085QOjyDLHEMlsaI0J4wAAC"
related_ids:
  - "KYwCepZAf28YaPB6KZPI8nH6JgJoYAAJ"
  - "Ta1DHNqTc2xUSMjxCCHPE_N61jD48AAM"
---
# Loading authored assets — .usdz / .reality

> ⚠️ **ALTERNATIVE PATH — NOT THE FRAMEWORK THIS APP USES.** The native iOS app (`ios/KaraokeVR/`)
> is a **hand-rolled Metal** renderer (`StereoARRenderer.swift`, `Passthrough.metal`); its
> procedural models live in `ios/KaraokeVR/Models/` as interleaved vertex arrays
> (`MeshBuilder.swift`), built by the **`ios-model-maker`** agent. RealityKit is **not** linked or
> used. Keep this only as a reference for _if_ the native path ever moved to RealityKit.

**Sources** (fetched 2026-06-27, DocC JSON data endpoint):

- https://developer.apple.com/documentation/realitykit/entity/load(named:in:)
- https://developer.apple.com/documentation/realitykit/entity/loadmodel(named:in:)
- https://developer.apple.com/documentation/realitykit/entity/loadasync(named:in:)
- https://developer.apple.com/documentation/realitykit/entity/loadmodelasync(named:in:)
- https://developer.apple.com/documentation/realitykit/entity/init(named:in:)

**Availability:** the synchronous loaders are **iOS 13.0+**; the Combine `LoadRequest` async
loaders are iOS 13.0+ but **deprecated in iOS 18.0**; the modern async/await initializer is
**iOS 18.0+**.

> **What it's for:** the alternative to building geometry in code — load a pre-authored
> `.usdz` or `.reality` file (made in Reality Composer Pro, exported from a DCC tool, etc.) as
> a ready-made entity tree. The rough analogue of loading a glTF with a Three.js `GLTFLoader`,
> except RealityKit reads `.usdz`/`.reality` natively and there's no separate loader object.

---

## Synchronous loaders (iOS 13.0+, throwing)

```swift
static func load(named name: String, in bundle: Bundle? = nil) throws -> Entity        // iOS 13.0+
static func loadModel(named name: String, in bundle: Bundle? = nil) throws -> ModelEntity  // iOS 13.0+
```

- `load(named:)` returns a generic **`Entity`** tree (a whole authored scene/hierarchy).
- `loadModel(named:)` returns a **`ModelEntity`** (a single model). Use it when you want the
  convenience subclass.
- `name` is the resource name **without extension** (`"robot"` loads `robot.usdz` /
  `robot.reality`); `bundle: nil` means the main bundle.
- Both **`throw`** and both **block** — fine for tiny assets, but they stall the main thread for
  anything substantial. Prefer async loading for real assets.

```swift
let robot = try Entity.load(named: "robot")            // robot.usdz in the app bundle
```

## Modern async/await (iOS 18.0+)

```swift
convenience init(named name: String, in bundle: Bundle? = nil) async throws   // iOS 18.0+
```

`ModelEntity` inherits this from `Entity`, so `try await ModelEntity(named:)` also works and
returns a `ModelEntity`. **This is the recommended way to load on iOS 18+** — it replaces the
deprecated `LoadRequest` async loaders below.

```swift
Task { @MainActor in
    let robot = try await Entity(named: "robot")
    anchor.addChild(robot)
}
```

> **There is no synchronous `ModelEntity(named:)` initializer.** To "make a ModelEntity from a
> file," use `Entity.loadModel(named:in:)` (sync) or `try await ModelEntity(named:in:)` (async,
> iOS 18+).

## Deprecated Combine loaders (avoid)

```swift
static func loadAsync(named name: String, in bundle: Bundle? = nil)
    -> LoadRequest<Entity>          // iOS 13.0+, DEPRECATED iOS 18.0
static func loadModelAsync(named name: String, in bundle: Bundle? = nil)
    -> LoadRequest<ModelEntity>     // iOS 13.0+, DEPRECATED iOS 18.0
```

These return a Combine `LoadRequest` publisher you `.sink` on. **Deprecated as of iOS 18.0**
(macOS 15.0, visionOS 1.0) — use the async/await `Entity(named:in:)` instead. Only relevant if
a port must support pre-iOS-18 _and_ wants off-main-thread loading without `async`.

---

## Author externally vs build in code

| Build in code (`MeshResource` + materials)                                    | Load an authored file (`.usdz`/`.reality`)        |
| ----------------------------------------------------------------------------- | ------------------------------------------------- |
| Simple parametric props (boxes/cylinders/spheres) — the `assets/` house style | Complex models, sculpted geometry, baked textures |
| Size/color driven by options at runtime                                       | Fixed art made in a DCC / Reality Composer Pro    |
| No asset files to ship or version                                             | Ships binary assets in the app bundle             |
| Matches the web procedural-factory pattern                                    | Matches "designer hands you a model" workflows    |

**Do this / not that**

- **Do** prefer `try await Entity(named:)` on iOS 18+; fall back to the throwing synchronous
  `load`/`loadModel(named:)` only for trivially small assets or older deployment targets.
- **Do** add `.usdz`/`.reality` files to the app **bundle** (the `in:` parameter) — these
  loaders read from a bundle, not arbitrary file paths.
- **Not that:** don't adopt `loadAsync`/`loadModelAsync` in new code — they're on the
  deprecation track since iOS 18.
- **Not that:** don't assume a sync `ModelEntity(named:)` exists — it doesn't.

---

## Could not confirm

- Whether these loaders accept an arbitrary on-disk `URL` (vs. bundle resource name) was not
  checked here — only the `(named:in:)` bundle-resource forms were fetched. RealityKit does have
  URL-based loading APIs; confirm the exact signature before relying on it.
