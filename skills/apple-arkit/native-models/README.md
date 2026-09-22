---
topic_id: "v2:DIKM"
topic_path: "model-runners/realitykit-authoring"
semantic_id: "Ja0DfbxW_00pWPNIGJrJK2NgNxB4MAAH"
related_ids:
  - "Ve0Du1xyVkd02VFAGBpMJ295NBa8sAAO"
  - "KYwCepZAf28YaPB6KZPI8nH6JgJoYAAJ"
---
# RealityKit model-authoring references (native counterpart to `assets/`)

These are trimmed digests of Apple's RealityKit documentation, focused on **building 3D
models in code** (and loading authored ones). They are the **native/RealityKit counterpart**
to this project's web `assets/` procedural-model pipeline — the same "function takes size/color
options, builds from primitives, grounded at the base, returns a composed object" mental model,
expressed in Swift entities/components instead of Three.js groups/meshes.

> ⚠️ **ALTERNATIVE PATH — NOT THE FRAMEWORK THIS APP USES.** The native iOS app
> (`ios/KaraokeVR/`) is a **hand-rolled Metal** renderer (`StereoARRenderer.swift`,
> `Passthrough.metal`), and its native procedural models already live in `ios/KaraokeVR/Models/`
> as interleaved vertex arrays (`MeshBuilder.swift`, `DiscoBall.swift`) — built by the
> **`ios-model-maker`** agent, not RealityKit. RealityKit is **not** linked or used anywhere.

**RealityKit is a possible future expansion, NOT the current pipeline.** Today's live experiences
render as WebGL (Three.js r132) inside an iOS WebView, and the native shell renders with Metal.
Nothing here is wired into either; these docs exist so that _if_ a native RealityKit render path
is ever added, the API surface is already mapped and verified against real Apple pages. For the
**current** native model pipeline, see `ios/KaraokeVR/Models/README.md` instead.

Fetched **2026-06-27** from `developer.apple.com/documentation/realitykit` (via the DocC JSON
data endpoint, which carries the real signatures + per-platform availability that the rendered
HTML pages hide behind JavaScript). Every signature and `iOS n.0+` marker below was read off a
live page on that date.

## Files

- **`realitykit-overview.md`** — `Entity` / `ModelEntity` / `ModelComponent`: the
  entity-component model, how a "model" is assembled and added to a scene, framed against the
  Three.js `Group`-of-`Mesh`es model.
- **`meshresource-primitives.md`** — `MeshResource.generateBox/Sphere/Plane/Cylinder/Cone/Text`
  primitive generators with exact signatures + availability. Closest analogue to the
  boxes/cylinders/spheres house style.
- **`procedural-meshes.md`** — `MeshDescriptor` + `MeshResource.Contents/Model/Part` and
  `generate(from:)`: the real build-geometry-from-scratch path. Newer (iOS 15) and
  throwing/async.
- **`materials.md`** — `SimpleMaterial` / `PhysicallyBasedMaterial` / `UnlitMaterial` /
  `Material`, their params, value-type attachment, and the mapping to the project's
  `MeshStandardMaterial`-vs-additive-unlit split.
- **`loading-usdz.md`** — loading `.usdz` / `.reality` via `Entity.load`/`loadModel` and the
  async `Entity(named:)`; sync vs async; when to author externally vs build in code.

## Verified-availability cheat sheet (iOS)

| API                                                          | iOS                        | notes                       |
| ------------------------------------------------------------ | -------------------------- | --------------------------- |
| `Entity`, `ModelEntity`, `ModelComponent`                    | 13.0+                      | core entity-component types |
| `SimpleMaterial`, `UnlitMaterial`, `Material`                | 13.0+                      |                             |
| `Transform`                                                  | 13.0+                      | `@frozen`                   |
| `MeshResource.generateBox/Sphere/Plane`                      | 13.0+                      | non-throwing                |
| `MeshResource.generateText`                                  | 13.0+                      | no macOS; Mac Catalyst yes  |
| `MeshResource.generateCylinder/generateCone`                 | **18.0+**                  | newest primitives           |
| `MeshDescriptor`, `MeshResource.Contents/Model/Part`         | 15.0+                      | custom geometry             |
| `MeshResource.generate(from:)` (descriptors **or** contents) | 15.0+                      | **throws**                  |
| `Entity.load/loadModel(named:in:)`                           | 13.0+                      | synchronous, throwing       |
| `Entity.loadAsync/loadModelAsync(named:in:)`                 | 13.0+, **deprecated 18.0** | `LoadRequest` (Combine)     |
| `Entity.init(named:in:) async throws`                        | **18.0+**                  | modern async/await loader   |

## Could not confirm (follow-ups)

- `MeshResource.generateBox(size:majorCornerRadius:minorCornerRadius:)` exists in the symbol
  list but its individual availability was not separately fetched.
- No `ModelEntity(named:)` _synchronous_ initializer exists; see `loading-usdz.md`.
- No `generateConvexPolygon` exists on `MeshResource` (an earlier model-memory guess; the live
  symbol list does not contain it).
