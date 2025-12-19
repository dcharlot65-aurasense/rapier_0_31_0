# Rapier.js 0.31.0 Build

Build updated NPM packages for Rapier physics engine matching Rust crate version 0.31.0.

## Background

The official `@dimforge/rapier3d` NPM package (v0.19.3) lags behind the Rust crate (v0.31.0). This project provides tooling to build aligned versions.

## Quick Start

```bash
# Prerequisites
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
cargo install wasm-pack
rustup target add wasm32-unknown-unknown

# Build
chmod +x build.sh
./build.sh

# Output in ./dist/
```

## What the Build Script Does

1. Clones official `dimforge/rapier.js` repository
2. Patches `Cargo.toml` files to use `rapier3d = "0.31.0"`
3. Builds with `wasm-pack` targeting web/bundler
4. Generates NPM packages in `./dist/`

## Manual Build Process

If the automated script fails due to API changes, follow this manual process:

### Step 1: Clone and Patch

```bash
git clone https://github.com/dimforge/rapier.js.git
cd rapier.js

# Edit rapier3d/Cargo.toml
# Change: rapier3d = { version = "0.22.0", ... }
# To:     rapier3d = { version = "0.31.0", ... }
```

### Step 2: Check for Breaking Changes

Between 0.22 and 0.31, Rapier had several API changes. Key ones to check:

```rust
// Renamed methods (check src/*.rs for these patterns):
// - TOI → ShapeCastHit
// - toi field → time_of_impact
// - BroadPhase → BroadPhaseMultiSap (trait now)
// - QueryPipeline::cast_shape() signature changed

// New features that may need bindings:
// - PdController, PidController
// - RigidBody::local_center_of_mass()
// - Voxels shape (experimental)
// - rapier3d-urdf, rapier3d-stl companion crates
```

### Step 3: Fix Binding Compilation Errors

Common fixes needed:

```rust
// If ShapeCastHit not found:
// Old: use rapier3d::geometry::TOI;
// New: use rapier3d::geometry::ShapeCastHit;

// If cast_shape signature changed:
// Old: query_pipeline.cast_shape(&collider_set, &pos, &vel, &shape, max_toi, stop_at_penetration, ...)
// New: query_pipeline.cast_shape(&collider_set, &pos, &vel, &shape, ShapeCastOptions { max_time_of_impact, ... }, ...)
```

### Step 4: Build

```bash
cd rapier3d
wasm-pack build --target web --release

# For bundler-friendly build:
wasm-pack build --target bundler --release
```

### Step 5: Package

```bash
cd pkg
npm pack
# Creates dimforge_rapier3d-0.20.0.tgz
```

## Build Variants

| Package | Features | Use Case |
|---------|----------|----------|
| `rapier3d` | standard | General use, wide browser support |
| `rapier3d-simd` | SIMD | Performance, requires simd128 |
| `rapier3d-deterministic` | cross-platform determinism | Replay validation |
| `rapier3d-compat` | base64 WASM | Bundlers that struggle with .wasm |

## SIMD Build

For SIMD builds, you need to modify the Rapier features:

```toml
# rapier3d/Cargo.toml
rapier3d = { version = "0.31.0", features = [
    "wasm-bindgen",
    "serde-serialize",
    "simd-stable",  # Add this, remove enhanced-determinism
    "debug-render",
] }
```

Note: `enhanced-determinism` and `simd-stable` are mutually exclusive.

## Integration with NeuroPlay/Forge

For your WebOS Runtime architecture:

```typescript
// Direct import (requires bundler WASM support)
import RAPIER from '@aurasense/rapier3d';

// Or async init for compat builds
import RAPIER from '@aurasense/rapier3d-compat';
await RAPIER.init();

// Create world
const gravity = { x: 0.0, y: -9.81, z: 0.0 };
const world = new RAPIER.World(gravity);
```

## Version Alignment

| NPM Package Version | Rapier Rust Version | Status |
|---------------------|---------------------|--------|
| 0.19.3 (official) | ~0.29.0 | Current NPM |
| 0.20.0 (this build) | 0.31.0 | Target |

## Troubleshooting

### "wasm-bindgen version mismatch"

Ensure `wasm-bindgen` CLI matches the crate version:
```bash
cargo install wasm-bindgen-cli --version 0.2.90
```

### "feature X not found"

Check the changelog for 0.31.0 feature renames. Common:
- `parallel` feature may have different requirements
- Some features renamed or removed

### Build fails on SIMD

SIMD requires:
```bash
# Install nightly for simd-nightly feature
rustup install nightly
rustup target add wasm32-unknown-unknown --toolchain nightly

# Or use simd-stable with stable Rust
```

## Publishing

```bash
# Login to NPM (use your own scope)
npm login --scope=@aurasense

# Publish each package
cd dist/rapier3d
npm publish --access public

cd ../rapier2d
npm publish --access public
```

## License

Apache-2.0 (same as Rapier)
