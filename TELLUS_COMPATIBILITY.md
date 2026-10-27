# Tellus Distant Horizons fork: 3.3.1-tellus.1

Based on upstream Distant Horizons 3.3.1 (`f0cefb7a994a670ca653e3dd52ff345275018724`)
and core/API `b0a5f3506364485eb9e4cd94b5343304adb6c88c`. The fork implements official
API 7.1.0 plus the optional generation-availability extension.

Published source repositories:

- Loader: https://github.com/Void0312Aurora/Distant-Horizons-Tellus
- Core/API: https://github.com/Void0312Aurora/Distant-Horizons-Core-Tellus

Clone the loader with `git clone --recurse-submodules` so that its pinned core
commit is checked out from the companion repository. Both repositories retain
the upstream history and LGPL-3.0 licensing.

The loader retains the 3.3.1 Shadow 9.0.0 pin fixing NeoForge translations. Tellus
patches preserve the 14-bit vertical format, old-cache conversion, child-upsample
metadata, Iris atlas allocation/upload state, and the neutral atlas with textures
disabled. The new mipmap filter and upstream renderer remain in place. Automatic
updates to stock DH are disabled even when an old config enabled the updater.

The generation queue calls `getGenerationAvailability` before starting work.
READY begins generation, SPLIT subdivides useful partially cached regions, and
WAIT leaves unavailable regions queued without consuming workers. Generators that
do not implement the extension remain ready by default. Queue selection avoids
per-candidate allocations; generation failure and cancellation release capacity
and pooled results. Completed columns at the required generation stage avoid
redundant child database scans.

Use the matching updated Tellus jar compiled against API 7.1. Tellus marks complete
LOD columns as FEATURES and checks managed coverage without scanning enormous
empty regions. Terrain sampling and detail settings are unchanged.

## Building

Use JDK 25 and the included Gradle wrapper. Build versions sequentially because
Manifold definitions and common build outputs are shared. Clean the loader/common
outputs when switching targets, then stage the jars before building the next one.

Supported `mcVer` values: `26.3.0`, `26.2.0`, `1.21.11`, `1.21.1`, and `1.20.1`.
Use `:fabric:remapJar` for every version; use `:neoforge:remapJar` except for 1.20.1,
which uses `:forge:remapJar`. Run `:api:test :core:test` once for core regressions.
The Tellus repository's `compat/distant-horizons/build.py` automates this matrix.

For the Minecraft 1.20.1 Fabric target validated on 2026-10-27:

```sh
./gradlew --no-daemon --max-workers=2 -PmcVer=1.20.1 \
  :api:test :core:test :fabric:remapJar --console=plain
```

Ship only the loader-specific `DistantHorizons-LOADER-3.3.1-tellus.1-MC.jar`, replacing
the old DH jar for that instance. Do not install two loader/version variants.

## Verification

All 80 API/core tests pass, including height/cache conversion, child upsampling,
readiness selection, cancellation, and synchronous/asynchronous failure cleanup.
All ten jars preserve the official API and resolve all 107 DH class/member
references in Iris 1.11.4 and 1.11.6. Language resources and core/API bytecode are
identical across loaders. The Apple M4 OpenGL smoke check validates initial atlas
allocation, neutral pixels, API texture IDs, pixel/sampler state, and atlas growth.
The Tellus repository contains the complete runtime/test notes in
`docs/distant-horizons-3.3.1.md` and reproducible source patches in
`compat/distant-horizons/patches/`.

A full interactive shader-pack playtest on Windows/NVIDIA remains separate from
these local checks.

## Iris render-pass ordering fix (2026-10-27)

DH initializes render parameters before firing `DhApiBeforeRenderEvent`, because
its event handlers require a valid client level. Iris can change
`deferTransparentRendering` in that event. Recompute the render pass and update
the parameters after the event, before calling the LOD renderer, so the first
shader frame does not use a stale `OPAQUE_AND_TRANSPARENT` pass.

The 1.20.1 Fabric build and API/core tests passed. An isolated Prism client using
Tellus, Sodium 0.5.13, Iris 1.7.6 and Complementary Reimagined r5.5.1 completed
forward/reverse native seam transitions (1296 sections each), chunk reload and
resource reload without the previous Iris pass error. Tellus detected the managed
generation extension without the stock-DH fallback warning. This was a bounded
Mesa llvmpipe run; long-distance generation, sustained flight and NVIDIA
performance remain unvalidated. The separate `BIOME_PALE_GARDEN` uniform warning
from the shader-pack/Iris combination remains.
