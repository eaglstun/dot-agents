---
topic_id: "v2:DCNO"
topic_path: "model-runners/native-models"
semantic_id: "rIhTfPrWc21V1GxgIHNNM3V6NhDoYAAG"
related_ids:
  - "Ta1DHNqTc2xUSMjxCCHPE_N61jD48AAM"
  - "KYwCepZAf28YaPB6KZPI8nH6JgJoYAAJ"
---
# Custom procedural meshes — MeshDescriptor + MeshResource.Contents

> ⚠️ **ALTERNATIVE PATH — NOT THE FRAMEWORK THIS APP USES.** The native iOS app (`ios/KaraokeVR/`)
> is a **hand-rolled Metal** renderer (`StereoARRenderer.swift`, `Passthrough.metal`); its
> procedural models live in `ios/KaraokeVR/Models/` as interleaved vertex arrays
> (`MeshBuilder.swift`), built by the **`ios-model-maker`** agent. RealityKit is **not** linked or
> used. Keep this only as a reference for _if_ the native path ever moved to RealityKit.

**Sources** (fetched 2026-06-27, DocC JSON data endpoint):

- https://developer.apple.com/documentation/realitykit/meshdescriptor
- https://developer.apple.com/documentation/realitykit/meshdescriptor/primitives-swift.enum
- https://developer.apple.com/documentation/realitykit/meshresource/generate(from:)-1yx1
- https://developer.apple.com/documentation/realitykit/meshresource/generate(from:)-4aahn
- https://developer.apple.com/documentation/realitykit/meshresource/contents-swift.struct
- https://developer.apple.com/documentation/realitykit/meshresource/model
- https://developer.apple.com/documentation/realitykit/meshresource/part

**Availability:** the whole custom-geometry path is **iOS 15.0+** (iPadOS 15.0+, macOS 12.0+) —
**newer than the iOS 13 primitive generators.** `MeshDescriptor`, `MeshResource.Contents`,
`MeshResource.Model`, `MeshResource.Part`, and both `generate(from:)` overloads are iOS 15.0+.

> **What it's for:** the real "build geometry from scratch" path — you supply positions,
> normals, texture coordinates, and an index/primitive list, exactly like filling a Three.js
> `BufferGeometry` with `position`/`normal`/`uv` attributes and an index. Use this when the
> primitive generators don't cover the shape.

---

## Two layers

1. **`MeshDescriptor`** — the ergonomic, per-submesh builder. You set buffers and a primitive
   topology, then hand an array of descriptors to `MeshResource.generate(from:)`.
2. **`MeshResource.Contents`** — the lower-level container (`models` → `Model` → `[Part]`)
   that `generate(from:)` also accepts directly, for full control over instancing/skeletons.

Most procedural work uses layer 1.

---

## MeshDescriptor

```swift
struct MeshDescriptor                                   // iOS 15.0+
init(name: String = "")
var positions: MeshBuffers.Positions                    // vertex positions  (SIMD3<Float>)
var normals: MeshBuffers.Normals?                       // optional
var textureCoordinates: MeshBuffers.TextureCoordinates? // optional UVs
var primitives: MeshDescriptor.Primitives               // the index/topology
// (additional buffers: tangents, bitangents, etc.)
```

### MeshDescriptor.Primitives (the topology enum)

```swift
enum MeshDescriptor.Primitives                          // iOS 15.0+
case triangles([UInt32])                                // flat triangle index list
case trianglesAndQuads(triangles: [UInt32], quads: [UInt32])
case polygons(_ : [UInt8], _ : [UInt32])                // per-face vertex counts + indices
```

- `.triangles([0,1,2, ...])` — the common case, like a Three.js index buffer (3 indices/face).
- `.trianglesAndQuads(...)` — mix tris and quads (quads use 4 indices each).
- `.polygons(counts, indices)` — arbitrary n-gons: `counts` gives each face's vertex count.

---

## Generating the resource (throwing)

```swift
static func generate(from descriptors: [MeshDescriptor]) throws -> MeshResource   // iOS 15.0+
static func generate(from content: MeshResource.Contents) throws -> MeshResource  // iOS 15.0+
```

**Both overloads `throw`** (invalid buffer sizes / mismatched indices fail at generation). Call
them with `try`.

> **Async note:** there was a `MeshResource.generateAsync(from:)` returning a Combine
> `LoadRequest<MeshResource>` (iOS 15.0+), but it is **deprecated as of iOS 18.0** (macOS 15.0,
> visionOS 1.0). Prefer the synchronous throwing `generate(from:)` above; if you need it off the
> main thread, wrap the call yourself. The current `generate(from:)` is synchronous + throwing,
> **not** `async`.

---

## Minimal triangle (verbatim from Apple's MeshDescriptor discussion)

```swift
var descriptor = MeshDescriptor(name: "triangle")
descriptor.positions = MeshBuffers.Positions([
    [-1, -1, 0], [1, -1, 0], [0, 1, 0]
])
descriptor.primitives = .triangles([0, 1, 2])

let mesh = try MeshResource.generate(from: [descriptor])
let entity = ModelEntity(mesh: mesh,
                         materials: [SimpleMaterial(color: .green, isMetallic: false)])
```

## A flat quad (two triangles)

```swift
var d = MeshDescriptor(name: "quad")
d.positions = MeshBuffers.Positions([
    [-0.5, 0, -0.5], [0.5, 0, -0.5], [0.5, 0, 0.5], [-0.5, 0, 0.5]   // xz-plane, y=0
])
d.normals = MeshBuffers.Normals([[0,1,0],[0,1,0],[0,1,0],[0,1,0]])
d.textureCoordinates = MeshBuffers.TextureCoordinates([[0,0],[1,0],[1,1],[0,1]])
d.primitives = .triangles([0, 1, 2,  0, 2, 3])
let mesh = try MeshResource.generate(from: [d])
```

---

## MeshResource.Contents / Model / Part (the lower layer)

```swift
struct MeshResource.Contents {                          // iOS 15.0+
    init()
    var models: MeshModelCollection                     // the geometry, grouped into Models
    var instances: MeshInstanceCollection               // placed instances of those models
    var skeletons: MeshSkeletonCollection               // for skinned meshes
}
struct MeshResource.Model { ... }                       // "a model consists of a list of parts"
struct MeshResource.Part { ... }                        // "a part of a model consisting of a single material"
```

A **`Part`** maps 1:1 to a material slot (Three.js "material groups" within one geometry); a
**`Model`** is a list of parts; **`Contents`** holds models + instances + skeletons. You only
reach for this layer when `[MeshDescriptor]` isn't enough (e.g. multiple instances of shared
geometry, or skeletal data).

---

## Confirmed vs. inferred

- **Confirmed from live pages:** `MeshDescriptor` (iOS 15.0+), its `primitives` property, the
  `Primitives` enum cases (`.triangles`, `.trianglesAndQuads`, `.polygons`), both
  `generate(from:)` overloads + their throwing signatures, the deprecation of
  `generateAsync(from:)`, and `Contents`/`Model`/`Part` with `Contents`'s members
  (`models`/`instances`/`skeletons`/`init()`). The triangle sketch is **verbatim from Apple's
  `MeshDescriptor` discussion**, including `descriptor.positions = MeshBuffers.Positions([...])`.
- **Inferred (not separately fetched):** the exact property names/types `normals`
  (`MeshBuffers.Normals?`) and `textureCoordinates` (`MeshBuffers.TextureCoordinates?`). These
  match the established RealityKit API but were not individually confirmed off their own pages —
  verify against Xcode Quick Help before relying on them in a port.

## Do this / not that

- **Do** gate this whole path behind `if #available(iOS 15.0, *)` if a port must run on
  iOS 13/14 — `MeshDescriptor` simply doesn't exist there.
- **Do** `try` every `generate(from:)` call and surface the error — bad index/buffer pairings
  throw rather than crash.
- **Not that:** don't use the deprecated `generateAsync(from:)`; it's removed-track since
  iOS 18.
- **Not that:** don't forget normals if you light the mesh with `SimpleMaterial`/
  `PhysicallyBasedMaterial` — without normals, lit shading is wrong (set them, or use
  `UnlitMaterial`).
