---
topic_id: "v2:DCKP"
topic_path: "model-runners/native-models"
semantic_id: "KYwCepZAf28YaPB6KZPI8nH6JgJoYAAJ"
related_ids:
  - "Ja0DfbxW_00pWPNIGJrJK2NgNxB4MAAH"
  - "3Y1bPE4V_085QOjyDLHEMlsaI0J4wAAC"
---
# RealityKit overview — Entity / ModelEntity / ModelComponent

> ⚠️ **ALTERNATIVE PATH — NOT THE FRAMEWORK THIS APP USES.** The native iOS app (`ios/KaraokeVR/`)
> is a **hand-rolled Metal** renderer (`StereoARRenderer.swift`, `Passthrough.metal`); its
> procedural models live in `ios/KaraokeVR/Models/` as interleaved vertex arrays
> (`MeshBuilder.swift`), built by the **`ios-model-maker`** agent. RealityKit is **not** linked or
> used. Keep this only as a reference for _if_ the native path ever moved to RealityKit.

**Sources** (fetched 2026-06-27, via the DocC JSON data endpoint):

- https://developer.apple.com/documentation/realitykit/entity
- https://developer.apple.com/documentation/realitykit/modelentity
- https://developer.apple.com/documentation/realitykit/modelcomponent
- https://developer.apple.com/documentation/realitykit/hastransform
- https://developer.apple.com/documentation/realitykit/hashierarchy

**Availability:** `Entity`, `ModelEntity`, `ModelComponent`, `HasTransform`, `HasHierarchy` are
all **iOS 13.0+** (iPadOS 13.0+, Mac Catalyst 13.0+, macOS 10.15+, visionOS, tvOS 26.0+). All
RealityKit rendering types are `@MainActor`.

---

## The entity-component model vs Three.js

RealityKit is **entity-component**, not a scene-graph-of-meshes. In Three.js you build a
`THREE.Group` and `.add()` `THREE.Mesh` children, where each mesh _is_ geometry + material. In
RealityKit:

- An **`Entity`** is just an identity + a transform + a bag of **components**. It carries no
  geometry by itself.
- Geometry+material is a **`ModelComponent`** (mesh + materials) you attach to an entity.
- A **`ModelEntity`** is the convenience subclass of `Entity` that already has a
  `ModelComponent` (plus collision/physics slots) — the thing you reach for to represent "a
  visible model."

| Three.js (r132)                    | RealityKit                                                |
| ---------------------------------- | --------------------------------------------------------- |
| `THREE.Group`                      | `Entity` (no mesh of its own; holds children + transform) |
| `THREE.Mesh` (geometry + material) | `ModelEntity`, or any `Entity` with a `ModelComponent`    |
| `mesh.geometry`                    | `ModelComponent.mesh` (a `MeshResource`)                  |
| `mesh.material`                    | `ModelComponent.materials` (`[any Material]`)             |
| `group.add(child)`                 | `entity.addChild(_:)`                                     |
| `mesh.position/quaternion/scale`   | `entity.position/orientation/scale` (via `HasTransform`)  |

So the `assets/` house pattern — _a factory that composes a `Group` of primitive meshes,
grounded at y=0, facing +Z_ — maps to _a factory that composes a root `Entity` with child
`ModelEntity`s, grounded at y=0_. (RealityKit is right-handed, **+Y up**; see the facing note
below.)

---

## Entity

```swift
@MainActor class Entity { init() }                       // iOS 13.0+
var components: Entity.ComponentSet { get set }          // the component bag
var name: String
var isEnabled: Bool
var scene: Scene?                                         // non-nil once added to a scene
```

**Transform** (from the `HasTransform` protocol, iOS 13.0+ — `Entity` conforms):

```swift
var transform: Transform                                 // scale + rotation + translation
var position: SIMD3<Float>                                // shorthand for transform.translation
var orientation: simd_quatf
var scale: SIMD3<Float>
func setPosition(_ position: SIMD3<Float>, relativeTo referenceEntity: Entity?)
func setOrientation(_ orientation: simd_quatf, relativeTo referenceEntity: Entity?)
func look(at target: SIMD3<Float>, from position: SIMD3<Float>,
          upVector: SIMD3<Float>, relativeTo referenceEntity: Entity?)
```

**Hierarchy** (from the `HasHierarchy` protocol, iOS 13.0+):

```swift
func addChild(_ entity: Entity, preservingWorldTransform: Bool = false)
func removeFromParent(preservingWorldTransform: Bool = false)
var children: Entity.ChildCollection
var parent: Entity?
```

> The bare `position`/`orientation`/`scale`/`addChild` accessors are protocol-extension
> members (`HasTransform`/`HasHierarchy`), not direct members of the `Entity` symbol page —
> that's why you won't find them listed _on_ the `Entity` page itself.

---

## ModelComponent

```swift
struct ModelComponent {                                  // iOS 13.0+ — value type
    init(mesh: MeshResource, materials: [any Material])
    var mesh: MeshResource
    var materials: [any Material]
    var boundsMargin: Float
}
```

`ModelComponent` is the unit that makes an entity _visible_: a mesh plus one material per mesh
"part." It is a **value type** attached via the component bag:

```swift
let entity = Entity()
entity.components.set(ModelComponent(mesh: .generateBox(size: 0.1),
                                     materials: [SimpleMaterial(color: .red, isMetallic: false)]))
```

---

## ModelEntity

The convenience subclass — an `Entity` that already owns a `ModelComponent`. This is the usual
"a model" object.

```swift
@MainActor class ModelEntity: Entity {                                   // iOS 13.0+
    init()
    convenience init(mesh: MeshResource, materials: [any Material] = [])
    convenience init(mesh: MeshResource, materials: [any Material],
                     collisionShape: ShapeResource, mass: Float)
    convenience init(mesh: MeshResource, materials: [any Material],
                     collisionShapes: [ShapeResource], mass: Float)
}
```

Loading a model _by name_ (from a `.usdz`/`.reality`) is **not** an init on `ModelEntity` —
use `Entity.loadModel(named:in:)` or the async `Entity(named:in:)`. See `loading-usdz.md`.

---

## Assembling a model and adding it to a scene (the house-style factory, RealityKit edition)

```swift
import RealityKit

/// Grounded at y = 0 (base on the floor). RealityKit is +Y up, right-handed.
@MainActor
func makeBeacon(height: Float = 0.6, color: UIColor = .systemTeal) -> Entity {
    let root = Entity()

    // post — a box, lifted so its base sits at y = 0
    let post = ModelEntity(mesh: .generateBox(width: 0.05, height: height, depth: 0.05),
                           materials: [SimpleMaterial(color: color, isMetallic: false)])
    post.position.y = height / 2          // generateBox is centered on origin → raise by half
    root.addChild(post)

    // cap — a sphere sitting on top
    let cap = ModelEntity(mesh: .generateSphere(radius: 0.06),
                          materials: [SimpleMaterial(color: .white, isMetallic: false)])
    cap.position.y = height + 0.06
    root.addChild(cap)

    return root
}

// elsewhere, with a RealityKit view:
//   let anchor = AnchorEntity(world: .zero)
//   anchor.addChild(makeBeacon())
//   arView.scene.addAnchor(anchor)
```

**Do this / not that**

- **Do** raise primitive parts by half their height — `generateBox`/`generateSphere` are
  **centered on the origin**, so "grounded at base" means offsetting up, exactly like the
  web factories' grounding convention.
- **Do** treat materials as values you build inline and hand to the `ModelComponent`.
- **Not that:** don't expect an `Entity` to render just because you created it — geometry only
  appears once a `ModelComponent` (or a `ModelEntity`) is in the bag _and_ the entity is
  parented under an anchor in a live `scene`.

---

## Transform (placement)

**Source:** https://developer.apple.com/documentation/realitykit/transform — **iOS 13.0+**,
`@frozen struct`. A `Transform` is the value behind an entity's `.transform` (RealityKit's
`TransformComponent`): scale + rotation + translation, in that order.

```swift
@frozen struct Transform                                // iOS 13.0+
init()                                                  // identity
init(scale: SIMD3<Float>, rotation: simd_quatf, translation: SIMD3<Float>)
init(pitch: Float, yaw: Float, roll: Float)             // Euler convenience
init(matrix: float4x4)

var translation: SIMD3<Float>                           // position
var rotation: simd_quatf                                // orientation
var scale: SIMD3<Float>
var matrix: float4x4                                    // composed 4x4
```

- Positions/scales are **`SIMD3<Float>`**; orientation is **`simd_quatf`** (e.g.
  `simd_quatf(angle: .pi, axis: [0,1,0])` for a 180° Y flip).
- RealityKit works in **meters**, right-handed, **+Y up**. "Grounded at base" = translate the
  entity up by half its height (primitive meshes are origin-centered).
- Set it whole (`entity.transform = Transform(scale:rotation:translation:)`) or via the
  shorthands `entity.position` / `.orientation` / `.scale` shown above.

### Facing convention (+Z vs −Z)

RealityKit's camera looks down **−Z**, so an object's "front" conventionally faces **−Z**
(toward a default camera). The web `assets/` convention is "face **+Z**." If a native port
reuses authored orientations, rotate 180° about Y (or build front-toward-−Z) so models face the
viewer. `Entity.look(at:from:upVector:relativeTo:)` and `Entity.ForwardDirection` exist for
explicit aiming; the default forward direction is −Z.
