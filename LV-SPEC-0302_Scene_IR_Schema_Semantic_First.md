# LV-SPEC-0302 — Scene IR Schema (Semantic-First)
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory)  
Implements: LV-SPEC-0300, LV-SPEC-0301  
Language Target: Rust  

---

# 1. Purpose

This specification defines the Scene IR (Intermediate Representation) used inside RPACK.

Scene IR is:

- Semantic-first (meaning before mesh)
- Renderer-agnostic
- Deterministically serializable
- Compatible conceptually with USD/glTF-style constructs
- Free of pixel buffers

Scene IR encodes:

- Object graph
- Transforms
- Constraints
- Camera semantics
- Lighting semantics
- Animation references

It does NOT encode:

- Framebuffers
- Render targets
- Device-specific shader bytecode

---

# 2. Design Philosophy

Traditional pipelines are mesh-first.

Lite-Vision Scene IR is semantic-first:

Objects are defined by meaning and role,
not only geometry.

Example:

Instead of:
“Triangle mesh at coordinates”

Scene IR expresses:
“Table object with physical support constraint”

Rendering engines interpret semantics.

---

# 3. Scene IR Root Structure

SceneIR {
  version: u32,
  scene_id: Hash32,
  nodes: Vec<Node>,
  edges: Vec<Edge>,
  cameras: Vec<Camera>,
  lights: Vec<Light>,
  materials: Vec<Material>,
  constraints: Vec<Constraint>,
  animations: Vec<Animation>,
  metadata: SceneMetadata
}

Canonical serialization required.

---

# 4. Object Graph Model

Scene is a directed acyclic graph (DAG).

Nodes represent entities.

Edges represent relationships.

---

# 5. Node Structure

Node {
  node_id: u64,
  name: String,
  semantic_type: NodeSemantic,
  transform: Transform,
  geometry_ref: Option<Hash32>,
  material_ref: Option<Hash32>,
  parent_id: Option<u64>,
  visibility: bool,
  tags: Vec<String>,
}

semantic_type examples:

- Object
- Actor
- Environment
- UIElement
- CameraAnchor
- LightAnchor
- ProceduralGenerator
- PhysicsBody

---

# 6. Transform Model

Transform {
  translation: [f64; 3],
  rotation: [f64; 4],   // quaternion
  scale: [f64; 3],
  local_space: bool,
}

Transforms MUST:

- Be applied parent → child
- Be deterministic
- Avoid non-deterministic math in deterministic mode

Reference profile MAY use fixed-point transform math.

---

# 7. Constraints

Constraint {
  constraint_id: u64,
  type: ConstraintType,
  target_a: u64,
  target_b: Option<u64>,
  parameters: ConstraintParams
}

Constraint types:

- ParentConstraint
- LookAt
- IKChain
- DistanceConstraint
- PhysicsLock
- OrientationLock
- ProceduralBinding

Constraints express semantic relationships independent of renderer.

---

# 8. Geometry Reference Model

geometry_ref references RPACK chunk by:

Hash32

Geometry is not embedded in Scene IR.

Geometry may represent:

- Mesh
- SDF
- Procedural generator
- Point cloud
- Parametric surface

Renderer interprets geometry accordingly.

---

# 9. Camera Semantics

Camera {
  camera_id: u64,
  projection: ProjectionType,
  fov_y: f64,
  near_plane: f64,
  far_plane: f64,
  transform_node: u64,
  aspect_policy: AspectPolicy,
}

ProjectionType:

- Perspective
- Orthographic
- Custom

AspectPolicy:

- PreserveVertical
- PreserveHorizontal
- Adaptive

Camera semantics define meaning.
Renderer chooses implementation.

---

# 10. Lighting Semantics

Light {
  light_id: u64,
  type: LightType,
  color: [f32; 3],
  intensity: f32,
  transform_node: u64,
  range: Option<f32>,
  shadow: bool,
}

LightType:

- Directional
- Point
- Spot
- Area
- Environment

Lighting defines energy semantics, not shading model.

---

# 11. Materials

Material {
  material_id: u64,
  shading_model: ShadingModel,
  parameters: MaterialParams,
  texture_refs: Vec<Hash32>,
}

ShadingModel examples:

- PBR_MetallicRoughness
- Unlit
- Subsurface
- CustomDeterministic

Deterministic profile MUST restrict shading models to deterministic subset.

---

# 12. Animation

Animation {
  animation_id: u64,
  target_node: u64,
  channel: AnimationChannel,
  keyframes: Vec<Keyframe>,
}

AnimationChannel:

- Translation
- Rotation
- Scale
- MaterialParameter
- Visibility

Animation MUST be deterministic if deterministic render required.

---

# 13. Metadata

SceneMetadata {
  job_id: Hash32,
  kernel_id: Hash32,
  renderer_profile_required: Option<u16>,
  deterministic_mode: bool,
  creation_block_height: u64,
}

Metadata MUST NOT contain system wall-clock timestamps.

---

# 14. Compatibility Mapping (Conceptual)

Scene IR is conceptually compatible with:

USD-style constructs:
- Prim → Node
- Xform → Transform
- Relationship → Edge
- Material binding → material_ref

glTF-style constructs:
- Node hierarchy → nodes + parent_id
- Mesh reference → geometry_ref
- Camera → Camera
- Animation channels → Animation

Scene IR does NOT embed USD or glTF format directly.
It provides a superset semantic schema.

Adapters MAY convert:

SceneIR → USD
SceneIR → glTF
SceneIR → engine-native format

Conversion MUST preserve semantics where possible.

---

# 15. Determinism Considerations

If deterministic render required:

- Fixed traversal order (node_id ascending)
- Stable floating-point evaluation (or fixed-point)
- Deterministic shader subset
- No hardware-specific optimizations
- Stable animation interpolation

Scene IR ordering MUST be canonical.

---

# 16. Streaming Considerations

SceneIR chunk SHOULD appear first in RPACK.

Renderer MAY:

- Begin rendering once SceneIR parsed
- Stream geometry progressively
- Stream textures lazily

Scene graph must not depend on full asset load.

---

# 17. Security Constraints

Scene IR MUST:

- Avoid executable script injection
- Avoid arbitrary system calls
- Avoid external URL fetch without hash

All external references MUST be content-addressed.

---

# 18. Rust Implementation Requirements

Implementation MUST:

- Use canonical serialization
- Use ordered collections (BTreeMap if maps used)
- Validate node graph acyclic property
- Validate transform hierarchy
- Validate constraint targets exist
- Reject invalid references deterministically

Floating-point determinism MUST be handled in deterministic mode.

---

# 19. Compliance Requirements

To claim LV-SPEC-0302 compliance:

Implementation MUST:

- Implement Node graph structure
- Implement Transform semantics
- Implement Constraint schema
- Implement Camera and Light schema
- Support geometry and material references
- Support canonical serialization
- Preserve semantic-first design
- Support deterministic mode flags

---

# 20. Design Invariants

1. Scene IR is semantic-first.
2. Geometry is referenced, not embedded.
3. Rendering is client-side only.
4. Deterministic mode must be replayable.
5. Scene graph must be acyclic.
6. Pixel buffers must never appear in Scene IR.
7. Hash commitments must be stable.

---

# 21. Summary

LV-SPEC-0302 defines the Semantic-First Scene IR:

- Object graph
- Transforms and constraints
- Camera and lighting semantics
- Material and animation
- Compatibility with USD/glTF concepts
- Deterministic rendering hooks

It allows:

Structured reality.
Renderer independence.
Reproducibility when required.
Client-side pixel generation.

Lite-Vision remains:

Consensus-separated.
Compute-scalable.
Semantically rich.
Deterministically anchored.

---

End of LV-SPEC-0302