---
name: "inhousexr-scene-optimize"
description: "Slim a scene on any InHouseXR-powered platform (SapphireXR, etc.) for mobile/VR: measure weight in the editor, shrink textures, simplify meshes, thin terrain trees and details, then save and verify."
---

# Optimize an InHouseXR scene for mobile / VR

Applies to any platform built on the InHouseXR (inhousexr.com) engine. SapphireXR is one example. Work in the built-in browser on that platform's web viewer/editor with the scene open (the user logs in). The engine is `window.__inhousexrEngine` (call it `e`).

- **API base:** the platform's `virtualSceneBaseUrl`. On SapphireXR it's `/app2/`; on other platforms read it from the network requests (look for `inventory/list`, `scenes/load`) rather than assuming. Endpoints below are written as `<api>/...`.
- **Limits:** read them from the platform's own config (`platform.js` → `worldLimits`). On SapphireXR they are mobile 3M tris / 350 MB memory / 150 MB download and VR 3.5M / 500 MB / 300 MB; other platforms may differ.

## 1. Measure
- `e.measureWorldWeight()` is what Save stores in `sceneDocument.weight`. Memory = unique geometry bytes + textures, each texture capped at 1024 px and counted as w*h*4*4/3. Map and emissiveMap are separate texture objects, so they count twice.
- Break down per top-level object under `e.worldRoot.children[0]`: triangles (instanced = index/3 * count) and texture MB. `<api>/inventory/meta?id=` gives item names and sizes. `performance.getEntriesByType('resource')` shows download sizes.

## 2. Textures and meshes (GLB models)
- Inventory downloads are encrypted, so do not attempt decrypt them. Use the user's local source GLBs instead.
- Before trusting a local file, compare geometry signatures: per primitive, vertex count, index count and rounded sum of x/y/z, local file vs `userData.modelRoot` in the live scene. Only use files that match.
- Rewrite the GLB in Python (PIL): replace only the image bufferViews and keep JSON nodes, meshes, materials and accessors identical. That keeps colliders, deletedChildRendererPaths and child overrides valid.
- Rules: tiled color and normal maps 1024 → 512; skin-type textures 1024×512 → 512×256; huge unique ground 2048 → 1024; keep text, signs and posters, anything ≤512, and sprite sheets.
- Decor meshes: meshoptimizer (node, `MeshoptSimplifier.simplify`) to about 40%. Check the result with a matplotlib render. Only simplify objects nobody stands on.
- Leave sounds alone if they're already 22 kHz mono WAV; MP3 adds gaps to loops.
- The user imports the `_opt.glb` files (the browser can't fill a file picker). Get new ids from `<api>/inventory/list?kind=All`.
- Swap `meshRef` and `placedInventoryId` in `e.sceneDocument`. Hide or delete parts with `Mesh.deletedChildRendererPaths` (use `userData.sceneChildPath`). Remove a node by splicing it out of `root.children`.
- Reload in memory with `await e.openSceneDocument(JSON.parse(JSON.stringify(e.sceneDocument)))`, then re-measure.

## 3. Terrain trees and details
- Runtime: `T.userData.terrainRuntime` (settings, trees [{u,v,s,r,p}], detailMaps Uint8 0–255, treeColliders). u = x/width + .5, v = worldZ/length + .5; document z = -world z.
- Instances per chunk = area × density, kept where rand*255 < map value. Mesh-detail density has a minimum of 0.1, so scale the map values to go lower.
- Use the editor pipeline: `e.updateTerrainSettings(T, {detailDistance, details})`, set `rt.detailMaps[i]` / `rt.trees`, then `e.rebuildTerrainTrees(rt)`, `e.markTerrainDetailsDirty(rt)`, `e.refreshCollisionCache()`, `rt.dirty = true`, `await e.flushTerrain(rt)`. Clearing: `e.terrainAction(T, 'clearTrees' | 'clearDetails')`.
- Build exclusion distance fields from live water mesh vertices and prop mesh bounds (1 m grid), plus trails, clearing and slope from `rt.heights`.
- Concentrate grass in the play area, put flowers in clumps, bushes in an edge ring, and thin the forest scatter. Thin trees greedily, keeping tall ones first with a minimum spacing.
- Users usually want it lighter than a first pass. Start lean: at most about 1,000 grass clumps and 30 bushes visible from the main play area, about 120 trees on a 200 m map, and a detail distance of 30 m.
- Count detail as the player sees it: put the camera at ground level in the play area, call `e.updateTerrainDetails(rt, e.camera)`, wait, then count InstancedMesh.count. Restore the camera afterwards.

## 4. Verify and save
- Snapshot collision first (`e.collisionMeshes`: count, triangles and a world-position checksum per owner object) and compare after. Platforms must be identical; animated objects drift slightly.
- Save: File → Save (posts sceneDocument plus measured weight). Confirm with `<api>/scenes/load?id=<sceneId>` (cache 'no-store').
- Append a dated note to the project README or notes. The editor keeps scene backups for rollback.
- Ask before saving if a change touches anything players land on.