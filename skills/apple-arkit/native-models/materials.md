---
topic_id: "v2:DCKK"
topic_path: "model-runners/native-models"
semantic_id: "rKm0OM6EyjUb5GhQCOFP41n4f45I0AAG"
related_ids:
  - "Zik2asZFTCSP7_neJMQO9XFwuz7ikAAN"
  - "vuSYYOfQOrEb5t4BuqFT05nalb7KMAAB"
---
# Materials — SimpleMaterial / PhysicallyBasedMaterial / UnlitMaterial

> ⚠️ **ALTERNATIVE PATH — NOT THE FRAMEWORK THIS APP USES.** The native iOS app (`ios/KaraokeVR/`)
> is a **hand-rolled Metal** renderer (`StereoARRenderer.swift`, `Passthrough.metal`); its
> procedural models live in `ios/KaraokeVR/Models/` as interleaved vertex arrays
> (`MeshBuilder.swift`), built by the **`ios-model-maker`** agent. RealityKit is **not** linked or
> used. Keep this only as a reference for _if_ the native path ever moved to RealityKit.

**Sources** (fetched 2026-06-27, DocC JSON data endpoint):

- https://developer.apple.com/documentation/realitykit/material
- https://developer.apple.com/documentation/realitykit/simplematerial
- https://developer.apple.com/documentation/realitykit/physicallybasedmaterial
- https://developer.apple.com/documentation/realitykit/unlitmaterial

**Availability:** `Material` (protocol), `SimpleMaterial`, `UnlitMaterial` are **iOS 13.0+**.
`PhysicallyBasedMaterial` is **iOS 15.0+** (iPadOS 15.0+, macOS 12.0+).

> **What it's for:** a `MeshResource` carries no color — materials do. They're **value types**
> (structs conforming to the `Material` protocol) attached as the `materials: [any Material]`
> array of a `ModelComponent`/`ModelEntity`. One material per mesh "part" (face/submesh); a
> single-material model just passes a one-element array.

---

## Material (protocol)

```swift
protocol Material                                       // iOS 13.0+
```

"A type that describes the material aspects of a mesh, like color and texture." The concrete
conformers below are all value types; `materials` is typed `[any Material]`.

---

## SimpleMaterial — lit, the default workhorse (iOS 13.0+)

"A basic material that responds to lights in the scene." This is the **closest analogue to the
project's `MeshStandardMaterial`** (the lit path that needs a light in the scene).

```swift
struct SimpleMaterial                                   // iOS 13.0+
init()
init(color: SimpleMaterial.Color,
     roughness: MaterialScalarParameter,
     isMetallic: Bool)

var color: SimpleMaterial.BaseColor                     // base color (+ optional texture)
var roughness: MaterialScalarParameter                  // 0 = glossy, 1 = matte
var metallic: MaterialScalarParameter
```

- `SimpleMaterial.Color` is the platform color type (`UIColor` on iOS).
- `roughness`/`metallic` are `MaterialScalarParameter`, which is `ExpressibleByFloatLiteral` —
  you can pass a bare `Float` (`roughness: 0.5`).
- There is also a common convenience form `SimpleMaterial(color:isMetallic:)` used throughout
  Apple's samples; the fully-specified initializer is `init(color:roughness:isMetallic:)` above.

```swift
let lit = SimpleMaterial(color: .systemTeal, roughness: 0.4, isMetallic: false)
```

## PhysicallyBasedMaterial — full PBR (iOS 15.0+)

"A material that simulates the appearance of real-world objects." Richer than `SimpleMaterial`
(separate maps for base color, roughness, metallic, emissive, normal, clearcoat, etc.). The
near-equivalent of a fully-configured `MeshStandardMaterial`/`MeshPhysicalMaterial`.

```swift
struct PhysicallyBasedMaterial                          // iOS 15.0+
init()

var baseColor: PhysicallyBasedMaterial.BaseColor
var roughness: PhysicallyBasedMaterial.Roughness
var metallic: PhysicallyBasedMaterial.Metallic
var emissiveColor: PhysicallyBasedMaterial.EmissiveColor
var emissiveIntensity: Float
var blending: PhysicallyBasedMaterial.Blending
var faceCulling: PhysicallyBasedMaterial.FaceCulling
```

Set fields after `init()`:

```swift
var pbr = PhysicallyBasedMaterial()
pbr.baseColor = .init(tint: .orange)
pbr.roughness = 0.2
pbr.metallic  = 1.0
pbr.emissiveColor = .init(color: .orange)   // glow source for bloom-style looks
pbr.emissiveIntensity = 2.0
```

## UnlitMaterial — ignores lighting (iOS 13.0+)

"A material that doesn't respond to lights in the scene." This is the **analogue of the
project's additive/unlit glow idiom** (`MeshBasicMaterial`, `toneMapped:false`, etc.) — flat
color that doesn't depend on scene lights. Good for neon/HUD/reticle-style geometry.

```swift
struct UnlitMaterial                                    // iOS 13.0+
init()
init(color: UnlitMaterial.Color)                                       // UIColor on iOS
init(color: UnlitMaterial.Color, applyPostProcessToneMap: Bool)
init(applyPostProcessToneMap: Bool)
init(texture: TextureResource)

var color: UnlitMaterial.BaseColor
```

```swift
let neon = UnlitMaterial(color: .magenta)                  // flat, light-independent
```

> Note: `UnlitMaterial` is the _additive-glow-adjacent_ primitive but isn't literally additive
> blending. For true additive/transparency control reach for `PhysicallyBasedMaterial.blending`
> or `UnlitMaterial`'s `applyPostProcessToneMap` flag; a 1:1 of the Three.js
> `AdditiveBlending + depthWrite:false` idiom wasn't confirmed on a single page here.

---

## Mapping to the project's two-material split

| `assets/` (Three.js r132)                                                         | RealityKit                                                      |
| --------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| `MeshStandardMaterial` (solid/lit, needs a light)                                 | `SimpleMaterial` (iOS 13) or `PhysicallyBasedMaterial` (iOS 15) |
| additive-unlit glow (`MeshBasicMaterial`, `toneMapped:false`, `AdditiveBlending`) | `UnlitMaterial` (light-independent flat color)                  |

## Do this / not that

- **Do** remember materials are **values** — build them inline and pass them in the array;
  reassigning `entity.model?.materials` replaces them.
- **Do** keep the per-part contract in mind: with `generateBox(..., splitFaces: true)` or a
  multi-`Part` mesh, the `materials` array is consumed one entry per part/face.
- **Not that:** don't reach for `PhysicallyBasedMaterial` if the port must run on iOS 13/14 —
  it's iOS 15.0+. Use `SimpleMaterial` there.
- **Not that:** don't expect a lit material to show shading without a light/environment in the
  scene — same caveat as `MeshStandardMaterial` in the web assets.
