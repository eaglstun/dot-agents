---
topic_id: "v2:DINF"
topic_path: "model-runners/realitykit-authoring"
semantic_id: "Ta1DHNqTc2xUSMjxCCHPE_N61jD48AAM"
related_ids:
  - "rIhTfPrWc21V1GxgIHNNM3V6NhDoYAAG"
  - "3Y1bPE4V_085QOjyDLHEMlsaI0J4wAAC"
---
# MeshResource primitive generators

> ⚠️ **ALTERNATIVE PATH — NOT THE FRAMEWORK THIS APP USES.** The native iOS app (`ios/KaraokeVR/`)
> is a **hand-rolled Metal** renderer (`StereoARRenderer.swift`, `Passthrough.metal`); its
> procedural models live in `ios/KaraokeVR/Models/` as interleaved vertex arrays
> (`MeshBuilder.swift`), built by the **`ios-model-maker`** agent. RealityKit is **not** linked or
> used. Keep this only as a reference for _if_ the native path ever moved to RealityKit.

**Source** (fetched 2026-06-27, DocC JSON data endpoint):
https://developer.apple.com/documentation/realitykit/meshresource and its per-method pages.

**Availability:** `MeshResource` itself is **iOS 13.0+**. Individual generators vary — the box/
sphere/plane/text generators are iOS 13.0+, but **`generateCylinder`/`generateCone` are iOS
18.0+**. All generators are `@MainActor static func` and return a `MeshResource` by value. The
primitive generators below are **non-throwing** (unlike the custom-geometry `generate(from:)` —
see `procedural-meshes.md`).

> **What it's for:** the direct analogue of the project's boxes/cylinders/spheres house style.
> A `MeshResource` is just the geometry (Three.js's `BufferGeometry`); pair it with materials
> in a `ModelComponent`/`ModelEntity` to get something visible.

---

## Box

```swift
static func generateBox(size: Float, cornerRadius: Float = 0) -> MeshResource          // iOS 13.0+
static func generateBox(size: SIMD3<Float>, cornerRadius: Float = 0) -> MeshResource    // iOS 13.0+
static func generateBox(width: Float, height: Float, depth: Float,
                        cornerRadius: Float = 0,
                        splitFaces: Bool = false) -> MeshResource                        // iOS 13.0+
static func generateBox(size: SIMD3<Float>, majorCornerRadius: Float,
                        minorCornerRadius: Float) -> MeshResource                        // exists; availability not separately confirmed
```

- `size: Float` → a cube of that edge length. `size: SIMD3<Float>` → width/height/depth.
- `splitFaces: true` lets you assign a **different material per face** (the materials array is
  consumed one-per-face) — handy for a "six-sided" look.
- **Centered on the origin.** To ground at the base, offset the entity up by `height/2`.

```swift
let cube = ModelEntity(mesh: .generateBox(size: 0.2, cornerRadius: 0.01),
                       materials: [SimpleMaterial(color: .orange, isMetallic: false)])
```

## Sphere

```swift
static func generateSphere(radius: Float) -> MeshResource                               // iOS 13.0+
```

Centered on the origin. (Use for floating props; ground others by offsetting up by `radius`.)

## Plane

```swift
static func generatePlane(width: Float, depth: Float, cornerRadius: Float = 0) -> MeshResource   // iOS 13.0+
static func generatePlane(width: Float, height: Float, cornerRadius: Float = 0) -> MeshResource  // iOS 13.0+
```

Two overloads with **different orientations** — pick by the second label:

- `width:depth:` → lies in the **xz-plane** (horizontal, like a floor).
- `width:height:` → stands in the **xy-plane** (vertical, like a wall/billboard).

```swift
let floor = ModelEntity(mesh: .generatePlane(width: 2, depth: 2),
                        materials: [SimpleMaterial(color: .gray, isMetallic: false)])
```

## Cylinder & Cone — **iOS 18.0+ only**

```swift
static func generateCylinder(height: Float, radius: Float) -> MeshResource              // iOS 18.0+
static func generateCone(height: Float, radius: Float) -> MeshResource                  // iOS 18.0+
```

> **Do this / not that:** these are the **newest** primitives. If a native port must run on
> iOS 13–17, you **cannot** call `generateCylinder`/`generateCone` — build the shape from a
> `MeshDescriptor` (see `procedural-meshes.md`) or approximate with a box, and gate the call
> behind `if #available(iOS 18.0, *)`.

## Text (extruded 3D text)

```swift
static func generateText(_ string: String,
                         extrusionDepth: Float = 0.25,
                         font: MeshResource.Font = .systemFont(ofSize: 16),
                         containerFrame: CGRect = .zero,
                         alignment: CTTextAlignment = .left,
                         lineBreakMode: CTLineBreakMode = .byTruncatingTail) -> MeshResource   // iOS 13.0+
```

- Produces an extruded 3D mesh for **static** text (re-generate the resource to change it).
- **Availability quirk:** the platform list is iOS / iPadOS / Mac Catalyst / tvOS / visionOS —
  **no native macOS** entry. `MeshResource.Font` is a typealias for the platform font type
  (`UIFont` on iOS).

```swift
let label = ModelEntity(mesh: .generateText("SING", extrusionDepth: 0.02,
                                            font: .systemFont(ofSize: 0.1)),
                        materials: [UnlitMaterial(color: .white)])
```

---

## Notes / gotchas

- There is **no `generateConvexPolygon`** on `MeshResource` — verified against the live symbol
  list. For arbitrary convex/concave geometry use `MeshDescriptor`.
- Primitive generators are cheap one-liners but give you no control over segment counts /
  vertex layout. When you need that (or a shape these don't cover), drop to `MeshDescriptor`.
- A `MeshResource` is geometry only — it has no color. Color/lighting lives in the material.
