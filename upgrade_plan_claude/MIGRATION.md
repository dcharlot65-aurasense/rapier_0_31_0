# Rapier API Migration Guide: 0.22 → 0.31

This document tracks breaking changes in the Rapier Rust API that affect the JS bindings.

## Version Progression

The official rapier.js was at Rapier 0.22.0. To reach 0.31.0, we traverse:
- 0.22 → 0.23 → 0.24 → 0.25 → 0.26 → 0.27 → 0.28 → 0.29 → 0.30 → 0.31

## Breaking Changes by Version

### 0.23 (nalgebra 0.33, parry 0.16)

```rust
// QueryFilterFlags values changed (divided by 2)
// This was actually a JS binding fix, but may affect constants

// BroadphaseMultiSap serialization changed
// colliders_proxy_ids now Vec<(ColliderHandle, BroadPhaseProxyIndex)>

// solve_character_collision_impulses signature changed
// Old: fn solve_character_collision_impulses(..., collisions: &[...])
// New: fn solve_character_collision_impulses(..., collisions: impl Iterator<Item = &...>)
```

### 0.24 (April 2025)

- Trimesh/heightfield internal edge correction now opt-in
- Shape-casting functions take `ShapeCastOptions` parameter

```rust
// Old:
query_pipeline.cast_shape(
    &collider_set,
    &pos, &vel, &shape,
    max_toi,
    stop_at_penetration,
    filter
)

// New:
use rapier3d::geometry::ShapeCastOptions;
query_pipeline.cast_shape(
    &collider_set, 
    &pos, &vel, &shape,
    ShapeCastOptions {
        max_time_of_impact: max_toi,
        stop_at_penetration,
        target_distance: 0.0,  // New field
        compute_impact_geometry_on_penetration: true,  // New field
    },
    filter
)
```

### 0.25 (parry with Voxels support)

```rust
// New: Voxels shape (experimental)
// ColliderBuilder::voxels() added
// Collider::set_voxel() added

// Trimesh construction now returns Result
// Old: Collider::trimesh(vertices, indices)
// New: Collider::trimesh(vertices, indices)? // Returns Result
```

### 0.27 (Broad-phase refactor)

```rust
// BroadPhase is now a trait, not a struct
// Old: BroadPhase (concrete type)
// New: BroadPhaseMultiSap implements BroadPhase trait
//      DefaultBroadPhase type alias available

// KinematicCharacterController::autostep now disabled by default
// Must explicitly enable if needed
```

### 0.28

```rust
// Performance improvements, no major API breaks
// CCD improvements
```

### 0.29

```rust
// RigidBody inertia methods renamed
// Old: rigid_body.inv_principal_inertia_sqrt()
// New: rigid_body.inv_principal_inertia()

// Old: rigid_body.effective_world_inv_inertia_sqrt()  
// New: rigid_body.effective_world_inv_inertia()

// Legacy PGS solver methods removed:
// - World.numAdditionalFrictionIterations (JS)
// - switchToStandardPgsSolver (JS)
// - switchToSmallStepsPgsSolver (JS)
// - switchToSmallStepsPgsSolverWithoutWarmstart (JS)
```

### 0.30

```rust
// Voxels storage changed to sparse
// Allows much larger voxel maps without hitting WASM 4GB limit
// API unchanged
```

### 0.31 (November 2025)

```rust
// New features:
// - PdController, PidController for velocity-level rigid body control
// - RigidBody::local_center_of_mass()
// - RigidBodyPosition::pose_errors()
// - Implement Sub for RigidBodyVelocity

// Dependencies:
// - parry 0.19
// - nalgebra (check version)
```

## JS Binding Updates Needed

### 1. ShapeCastOptions Wrapper

```rust
// In src/geometry/shape.rs or similar
#[wasm_bindgen]
pub struct RawShapeCastOptions {
    pub max_time_of_impact: f32,
    pub stop_at_penetration: bool,
    pub target_distance: f32,
    pub compute_impact_geometry_on_penetration: bool,
}

impl From<&RawShapeCastOptions> for ShapeCastOptions {
    fn from(opts: &RawShapeCastOptions) -> Self {
        ShapeCastOptions {
            max_time_of_impact: opts.max_time_of_impact,
            stop_at_penetration: opts.stop_at_penetration,
            target_distance: opts.target_distance,
            compute_impact_geometry_on_penetration: opts.compute_impact_geometry_on_penetration,
        }
    }
}
```

### 2. Trimesh Result Handling

```rust
// Old binding:
#[wasm_bindgen]
pub fn trimesh(vertices: &[f32], indices: &[u32]) -> RawColliderSet {
    // ...
}

// New binding needs error handling:
#[wasm_bindgen]
pub fn trimesh(vertices: &[f32], indices: &[u32]) -> Result<RawColliderSet, JsValue> {
    // ... handle Result from Rapier
}
```

### 3. Remove Deprecated Methods

Remove these from the JS bindings if present:
- `World.numAdditionalFrictionIterations`
- `World.switchToStandardPgsSolver()`
- `World.switchToSmallStepsPgsSolver()`
- `World.switchToSmallStepsPgsSolverWithoutWarmstart()`

### 4. Rename Inertia Methods

```typescript
// TypeScript definitions update:
// Old:
invPrincipalInertiaSqrt(): Vector3;
effectiveWorldInvInertiaSqrt(): Matrix3;

// New:
invPrincipalInertia(): Vector3;
effectiveWorldInvInertia(): Matrix3;
```

## Compilation Checklist

When building, watch for these errors:

```
error[E0599]: no method named `inv_principal_inertia_sqrt` found
→ Rename to `inv_principal_inertia`

error[E0308]: mismatched types, expected `ShapeCastOptions`
→ Wrap max_toi in ShapeCastOptions struct

error[E0599]: no method named `cast_shape` found for struct `QueryPipeline`
→ Check signature changed, may need to update call site

error: cannot find type `BroadPhase` in this scope
→ Use `BroadPhaseMultiSap` or `DefaultBroadPhase`
```

## Testing After Build

```javascript
// Quick smoke test
import RAPIER from './pkg/rapier_wasm3d.js';

async function test() {
    await RAPIER.init();
    
    const gravity = { x: 0, y: -9.81, z: 0 };
    const world = new RAPIER.World(gravity);
    
    // Test shape cast with new API
    const shape = RAPIER.ColliderDesc.ball(1.0);
    // ... additional tests
    
    console.log("Build OK");
}

test();
```
