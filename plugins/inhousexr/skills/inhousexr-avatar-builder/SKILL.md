---
name: inhousexr-avatar-builder
description: Build avatars, clothing, hair, accessories and emotes for the Avatar Editor on any InHouseXR (IHXR) platform (SapphireXR, etc.): rig to the IHXR skeleton, verify, upload, set up materials and outfits.
---

# InHouseXR avatar builder

A field guide for making things people *wear or play on themselves* on platforms that run the InHouseXR engine (SapphireXR is one; others share the viewer at `https://<host>/viewer/`): full avatar bodies (humans and creatures, e.g. a Christmas tree, a donut, a squirrel), rigged clothing, static accessories, hair, shoes and emotes. It is distilled from real builds.

**Out of scope:** building scenes and scene characters (parade riders, NPCs, avatar clones). Use the `inhousexr-scene-builder` skill for those.

## 0. Ground rules (always)

- **A project folder is required.** Ask for one if none is connected. Keep the whole pipeline there: `scripts/` (generators, build, verify, render), `tex/`, `out/` (GLB/FBX to upload), `renders/` (previews), `verify/` (reports), `README.md` (final ids, settings, lessons).
- **No binary through chat.** Generate meshes and textures with code (Blender `bpy` in Python, numpy/Pillow, or JS in the page). Save upload files to the person's computer (e.g. Downloads) through the computer link.
- **The person picks files in native dialogs.** You can't fill a native file picker: tell them the exact filename to choose.
- **Original designs only:** no brand logos, no copyrighted characters. Mixamo or other third-party animations: link the person to the source; never host or re-share those files.
- **Modest by default:** human bases get underwear painted into the skin texture (no extra geometry).
- **Never delete the person's inventory items or outfits yourself.** At the end, tell them which old items are safe to delete.
- **Send a preview render before uploading**, and iterate until it genuinely looks good.

## 1. The IHXR skeleton

- Ask for (or find) the official **IHXR Skeleton template FBX**. List its bones first. It has 61 bones: `Root`, `Hips`, `Spine`, `Spine1`, `Spine2`, `Neck`, `Head`, `Jaw`, `LowerMouth`, `Eye_Left/Right`, four eyelid bones (`Eyelid_*` top/bottom left/right), shoulders, arms, forearms, hands, 15 finger joints per hand (Thumb, Pointer/Index, Middle, Ring, Pinky), `LeftUpLeg`/`RightUpLeg`, legs, feet, toe bases.
- A Body upload must use it: the editor checks for at least 3 IHXR bone names. Mixamo/Maya name prefixes are fine.
- The template FBX is in **centimetres**: the armature object is scaled 0.01 and rotated 90° on X, Hips at ~106 units.
- **Prep in Blender** (`pip install bpy --break-system-packages`, run scripts with `python3 -I`, never name a script after a stdlib module such as `inspect.py`):
  1. Delete the template body mesh.
  2. On every object: `animation_data_clear()`; delete all actions. If actions stay, the glTF exporter silently puts the 0.01 scale back.
  3. Apply rotation + scale on the armature.
  4. After export, re-import and check: armature scale 1, Hips ≈ 1.065 m above the root.
- **Keep bone names, hierarchy and orientations unchanged.** For non-humans you may *translate* face leaf bones (eyes, lids, jaw, lower mouth) to the new face, keeping orientation. Leave Head/Neck/Spine/Hips/limbs alone so auto-scale and sitting stay humanoid.

## 2. How the engine treats avatars (verified)

- **Retargeting uses bone rotations only.** Position and Root tracks are stripped from locomotion, sit and emote clips. The body must hinge believably from rotations alone.
- **Auto-scale:** the avatar is scaled so head-to-feet bone height = 1.5685 (a ~1.72 m human ≈ ×0.912). Outfits also have a **Global Scale**.
- **Sitting:** root = seat − (0, 0.838, 0.162) × avatar scale, so hip height matters. Legless bodies go through the seat; accept or avoid.
- **Blink (bones):** every bone matching `eye[_ -]*lid` + left/right + top/bottom rotates about its own local Z by ±32° (top +, bottom −) in a 0.5 s eased pulse at random intervals. Design lids so a 32° turn closes them with a few degrees of overlap.
- **Jaw (bones):** `jaw` / `lowerjaw` / `lowermouth`; opening = max(rest, mic level) × max open. Defaults Rest 2°, Max 11°; overridable per outfit (`JawRestOpen`, `JawMaxOpen`). Talk preview should show the jaw peaking near the max.
- Sprite-sheet eyes/mouth (Material Editor "Blinking Eyes" / "Talking Mouth") replace the bone blink/jaw for that avatar.
- Transparent objects sort back to front by their real position.
- No reflection environment in the Avatar Editor: full metal looks dark; use metallic ~0.55–0.95 with high smoothness only where it reads well.

## 3. Bodies (humans and creatures)

- **Shape:** build from code. Useful methods: parametric surfaces (torus, lathe, superellipse tubes), SDF unions → marching cubes (~4 mm voxels) → decimate with a detail group (face, mouth, hands, feet), or a CC0 human base with morph targets.
- **Weights:**
  - Every vertex weighted, sums = 1, max 3–4 influences. Unweighted vertices get pinned to the root and break sitting.
  - Simple bodies: weight by height bands (Hips → Spine → Spine1 → Spine2 → Neck → Head) with smoothstep blends.
  - Rigid decorations (baubles, sprinkles, bulbs) take the weights of their anchor point. Shells (icing, fur, clothes) use the same weight function as the surface under them.
  - Limbs: blend ±10 cm around elbows/knees; fingers on their own bones (merge Ring/Pinky 50/50 for mitten hands).
  - Don't add stubs where no bone fits: e.g. arm bones pivot at shoulder height, so "branch arms" lower down swing wildly.
- **Eyes and lids:**
  - Eye domes and lid shells share one pivot on the lid bones. Lid shells sit a few mm outside the eye; open margins + eye height must stay under 64° so 32° + 32° closes with ~3° overlap.
  - Push eyes into sockets (crown ~5 mm under the brow skin) so lids hide cleanly. Taper lid shells into the surface where a closed lid isn't needed.
  - Measure against the real decimated mesh (BVH), not the SDF.
- **Mouth:** lower lip (and a small chin patch) weighted to Jaw, corners on Head, a few mm proud of the surface.
- **Budget:** about 15–26k triangles, ~1 MB, 2–7 materials, textures 1024–2048 (JPEG atlases).
- **Verify before upload** with a small skinning simulator (GLB reader that mirrors the engine): node scales, Hips height, weights of sample vertices, triangle count, file size, plus poses (blink, jaw 2/11/15°, head yaw/pitch, sit, arms down, hands at chest, walk). Write pose OBJs, an edge-stretch report and `verify_report.json`, and render previews of the poses.

## 4. Clothing, hair and accessories

- **Rigged clothing** (jackets, dresses, pants, scarves): FBX with an `Avatar` armature (all 61 IHXR bones, centimetres, armature scale 0.01) + one skinned mesh. To fit several bases with one file: put bones at the average of the bases and shape the garment around all of them. Copy weights from the avatar's own skin, smooth, max 4 bones, all weighted, sums = 1. Test T-pose, arms raised and arms lowered on every base.
- **Static accessories** (hats, goggles, glasses): no armature; one parent null with one mesh, centimetres, Y-up, transforms baked. Put the origin at the **Head bone** so "Attachment bone: Head" lands it with no offset. Layering (goggles over beanies, scarves outside collars) must be designed in.
- **Shoes:** one parent null with exactly two meshes `ShoeL` / `ShoeR` and one material; soles flush with the foot bottoms; cover the toes/heels of every base.
- **Hair:** separate meshes or cards; alpha via a texture, not alpha-clip.
- Textures: BaseColor + Normal (OpenGL) + Roughness (+ Metallic) in a 1024 atlas per item, embedded and also saved to the folder. Typical: 5–26k triangles, 0.5–3 MB per item. Make a contact sheet of every item on every base.
- **No alphaTest / alpha-clip** on transparent materials: the editor reads any alphaTest > 0 as Cutout and saves it that way.

## 5. Emotes

- An emote is a GLB/FBX animation on the IHXR rig (61 bones). Upload it in the Avatar Editor with the **Emote** tab selected (the Body tab files it as a Body item).
- Rotation tracks only (positions and root are stripped). Fingers are stripped too unless the engine says otherwise; check before relying on them.
- Write emotes in code: bind-pose rig JSON + quaternion helpers + keyframes per bone, exported as GLB. Name them clearly (`01 Jump Rope - Basic Bounce`).
- **Loops:** make the first and last frame match, and give loops a whole number of beats/turns (e.g. a 0.5 s loop for 2 rope turns/s) so they stay in sync with scripts (`SetEmoteTempoSync`, spinning props).
- Contact timing matters for gameplay emotes (e.g. a golf stroke's hit at 0.56/0.62/0.70 s): record those times for the script.
- Scene scripts reference emotes through `Emote` fields; the Inspector shows the person's own emotes.

## 6. Avatar Editor workflow (do it exactly this way)

1. Save the upload file to the person's computer. In the Avatar Editor pick the right category (Body = torso icon, clothing slot, accessory, Emote tab) and click **Upload Item**; the person picks the file.
2. Put the outfit in **Edit mode** (click the brown "Baked" button) before adding items. "Add to Outfit" makes a body upload the Body.
3. Configure materials in the **Material Editor** (row → Materials → Mesh/Material dropdown): shader (Main Lit / Sprite Sheet Lit), base colour, metallic/smoothness, emission colour + intensity (0–8), surface (Opaque/Cutout/Transparent + opacity), tile, UV Scroll X/Y + Scroll step (texture widths per second; step quantises), render side, Talking Mouth, Blinking Eyes, Foliage Wind, Billboard (one mesh per material for billboards). **Use real mouse clicks for every checkbox and confirm with a zoomed screenshot**: settings changed only through scripts don't stick.
4. **Save as New Item** with a clean name, then add *that* item to the outfit as the Body. Otherwise Edit mode restores the raw upload with blank materials.
5. Outfit settings: Global Scale, Jaw Rest / Max open.
6. **Bake & Save** (never on an empty outfit). Verify: switch to Edit mode and confirm materials come back; load the outfit fresh from My Outfits and check blink, talk preview and materials again.
7. Record the final item ids, outfit id, settings and verified numbers in the README.

## 7. Effects that work on avatars

- UV scroll with a step makes chasing lights: a strip texture of time cells × colour bands, each bulb's UVs collapsed to one cell (e.g. Scroll X 0.25, step 0.0625 → 4 steps/s).
- Sprite Sheet Lit animates on avatars (flames, screens, faces).
- Emission ~2–4 for small bright details (stars, bulbs); keep large areas lower.

## 8. Wrap-up checklist

- [ ] Pipeline scripts, textures, upload files, renders and verify reports saved in the project folder
- [ ] GLB/FBX verified (scales, Hips height, weights, tris, size, pose simulation)
- [ ] Uploaded in the right category; materials set with real clicks; **Save as New Item** done and used
- [ ] Baked & saved; reloaded from My Outfits; blink, talk, materials, sitting checked
- [ ] README updated with ids, settings, numbers and lessons; list of old items safe to delete given to the person
