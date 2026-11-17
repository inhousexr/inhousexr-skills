---
name: "inhousexr-scene-builder"
description: "Build or extend any kind of scene on an InHouseXR (IHXR) platform (SapphireXR, etc.): plan, generate GLBs/textures/sounds, place, script in C#, light, optimize for mobile/VR, save and verify."
---

# InHouseXR scene builder

A field guide for building whole worlds on any platform that runs the InHouseXR engine (SapphireXR is one; others share the viewer at `https://<host>/viewer/`). It is distilled from many real builds: a holiday parade with floats and a skinned marching band, mini golf, a bowling alley, an obby, a skate park, an office, a campfire, a magic show, a fly simulator, a jump-rope studio, MIDI bands and more. The patterns are general: use them for games, social spaces, music shows, rides, venues, anything.

**Out of scope:** making avatars, wearable clothing and Avatar Editor emotes. Use the `inhousexr-avatar-builder` skill for those. This skill only covers characters that live *in the scene* (animated scene models, avatar clones, NPCs).

For a deep performance-only pass, also see the `inhousexr-scene-optimize` skill.

Work in the platform's own page through the browser tools (built-in browser or Claude in Chrome): JS in the viewer page plus a few UI clicks.

## 0. Ground rules (always)

- **A project folder is required.** If the person hasn't connected one, ask for it before building. Every generated mesh, texture, sound, script and generator is written there *before* it is uploaded, so the scene can be rebuilt. Typical layout: `gen/` (generators), `models/`, `textures/`, `sounds/`, `scripts/` (C#), `scene/` (doc snapshots), `docs/` (notes, engine requests), `tools/` (Node runners).
- **One inventory folder per project** at the root. Superseded uploads go to `<Project>/_Unused`: **move, never delete**. Re-uploading the same name makes a second item, not a replacement, so swap placed nodes to the new id.
- **Never click Publish / Public toggles** (the "● Private / ✓ Public" button in the Scenes list) unless asked.
- **No binary through chat.** Generate meshes, textures and audio *in the page* (JS) and upload with the file input. Keep the same generator runnable in Node so files on disk match.
- **Inventory downloads may be encrypted** (e.g. `IHXENC02`). Do not try to decrypt. Script source should not be re-downloaded either: keep every current `.cs` in the project folder.
- **Preview before uploading** (failed attempts must not clutter the inventory), save in the editor, then **verify after a reload in the real world**. Editor lighting and culling differ from the world.
- Keep a running notes file in the project (`NOTES.md`, `Layout.md`) with one pass per round: what changed, ids, coordinates, Inspector values, engine findings.
- When you find an engine bug or a missing API, say so plainly with evidence, give the closest thing that works today, and write a separate note (`docs/engine-request-<topic>.md`: symptom, evidence, proposed change, code sketch). Don't paper over it with fragile hacks unless the person wants a stopgap.
- Treat the person's account email and secrets (API keys, data-store keys) as private.

## 1. Two ways of working

Ask which one the person wants if it isn't clear.

- **A. Claude drives the editor**: generate, upload, place, set Inspector values, save. The person logs in and opens the scene.
- **B. Claude plans and scripts, the person places.** Give whole ready-to-import files, a parts list (every object, name, components, slots) and a test checklist.

Either way: **plan first** (layout table in metres, spawn point, scripts and how they talk, build order), wait for OK on big builds, then build in rounds and finish each round with a screenshot check (editor panels hidden) and a save.

## 2. Orientation: the engine from the page

- `window.__inhousexrEngine` (`E`): the three.js engine.
  - `E.sceneDocument` (`.root` node tree, `.skybox`, `.name`), `E.editMode`, `E.scene`, `E.worldRoot`, `E.camera`, `E.playerRoot`, `E.renderer`
  - `E.spawnNode(docNode, parentObj)` (re)builds a node; `E.deleteObjects([obj])` is the editor's delete (works for nodes the hierarchy doesn't list); `E.pasteObjects([nodeJson], null)` adds doc nodes with fresh ids and remaps internal refs (**one node per call**, or clones share the id map)
  - `E.parseModel(url)` loads GLB/FBX (blob URLs work) for previews and rebuilds
  - `E.sceneAudios`, `E.drawDistanceList`, `E.drawDistanceHiddenCount`, `E.instancing` (static batches), `E.midiState`; `E.previewMidi(id,{instruments,bindings})` / `E.stopMidiPreview()` test MIDI without Play mode
  - Play-mode tests: `E.playerRoot.position.set(...)` teleports locally, `E.requestGrab(obj)`, `E.releaseHeld()`, `E.onInteract({objectId, action:'click'})`
  - `window.__inhousexrFrameProfile`: the cheapest perf readout (draw calls, triangles, shadow passes, batches, objects beyond draw distance). `?perf` shows it on screen.
- A doc node: `{ id, name, transform:{pos, rot, scale, hasLocalTransform, localPos, localRot, localScale}, components:[...], children:[...], drawDistance?, hiddenInWorld? }`. Components: `Mesh {meshRef, inlineMaterials, castShadows, receiveShadows, animationDefaultClip...}`, `Collider`, `Light`, `AudioSource`, `Particle`, `Sitpoint`, `Grabbable`, `Scripts {scripts:[{behaviorTypeName, publicProps, publicMembers}]}`, `SystemPrefab` (Terrain, Water System, Media Screen, Base Plane...).
- **Script prop formats in the doc:** SceneObject/Sitpoint = node id string; `List<>` = JSON string of ids; InventoryItem / Emote = inventory item id.
- **Coordinates:** the doc is Unity-style left-handed: doc z = −three z; quaternion (x,y,z,w) ↔ three (−x,−y,z,w). Mesh geometry itself is not flipped. glTF models face +Z. A clean habit: author geometry in doc space and mirror z on GLB export, so placement = P + R·v.
- Find a node: walk `E.sceneDocument.root`. Its object: `E.scene.traverse(o => o.userData.doc === node)`. Things attached inside a model hang under `__Imported attachment origin` (a prop in a hand bone, a rider on a float deck).
- **React inputs:** native value setter → dispatch `input` → call the element's React `onBlur`/`onChange` (`el[Object.keys(el).find(k=>k.startsWith('__reactProps'))]`). Ranges/selects: same with `change`.
- **Rotations in the Inspector:** commit X, then Z, then Y last. Yaws beyond ±90° get rewritten as X=180/Z=180, and writing Z afterwards breaks them.
- **Opening the editor:** `/viewer/` → Scene Editor → click the scene row (thumbnail, never the Private/Public button) → Edit. Wait ~30 s, check `E.editMode === true` and `E.sceneDocument.name`. World test: Worlds → search → world card → Visit, check `E.editMode === false`.
- **Helper libraries:** earlier sessions may have saved helpers in the site's `localStorage` (keys like `__studio*`, `__mg_*`). Try booting them first, e.g. `(0,eval)('('+localStorage.getItem('__studioBoot')+')')()`; otherwise rebuild equivalents. Never busy-wait in the page (it starves the renderer): use timer-based sleeps.

## 3. Inventory: upload, place, swap, tidy

- **Upload from bytes** (GLB/PNG/JPG/WAV/MP3): open the project folder first (uploads land in the open folder), then
  ```js
  const inp=[...document.querySelectorAll('input[type=file]')].find(i=>/\.glb/.test(i.accept))||document.querySelector('input[type=file]');
  const dt=new DataTransfer(); dt.items.add(new File([bytes],'Name.glb',{type:'model/gltf-binary'}));
  inp.files=dt.files; inp.dispatchEvent(new Event('change',{bubbles:true}));
  ```
  Poll the "Inventory: N item(s)" count until it rises. Scripts (`.cs`) import the same way; use **Update Script** for new versions so slots stay filled.
- **Card id:** call the card's React `onDragStart` with a fake `dataTransfer`, read `text/plain` → `{id,name,kind}`.
- **Move a card:** card `onDragStart` → folder `onDragEnter`/`onDragOver`/`onDrop` → card `onDragEnd`.
- **Fill an InventoryItem slot:** synthetic DragEvent with `application/x-inhousexr-inventory` `{id,name,kind}` on the slot.
- **Place:** double-click a card with the folder open (lands at camera focus), then set the transform.
- **Swap a model in place** (keeps transform, children, scripts, draw distance, ids):
  ```js
  mc.meshRef = newId;              // the node's Mesh component
  const par = obj.parent; E.clearEditorSelection?.(); par.remove(obj); await E.spawnNode(node, par);
  ```
  Keep `inlineMaterials` only if they still match the new model. Then save.
- **Parent while keeping the world transform:** Cut → "Paste into" the new parent.
- **Prefabs:** Hierarchy → "Save to Inventory" (lands at the inventory root; move it).
- **Save:** File → Save; confirm `E.historySavedJson === JSON.stringify(E.sceneDocument)`. Snapshot the doc JSON into `scene/` after big rounds.

## 4. Generating content in the page

- Generators are plain JS (UMD: `module.exports` in Node, `window.X` in the page) so the same file writes to disk and uploads. A shared lib (geometry, rasters, GLB writer) plus one generator per area. Store page copies in `localStorage` under a prefix and `eval` them after reloads.
- **GLB writer:** POSITION/NORMAL/TEXCOORD_0(/JOINTS_0/WEIGHTS_0) + indices in one BIN, PNG/JPG images as bufferViews, JSON padded with spaces and BIN with zeros to 4 bytes, POSITION min/max. Validate with the Khronos validator in Node if unsure.
- **Textures:** canvas → PNG/JPG. Tiling textures: draw each element at ±size offsets. Keep them small (128–512; 1024 only for hero art). World-projected UVs on floors/walls/patios make joints seamless.
- **Vertex colours are ignored:** use textures or a palette texture.
- **Winding / normals:** mirrored geometry (scale −1, mirrored outlines) needs reversed triangle order or it renders black/inside-out — suspect this before lighting when one copy of a surface is black. Lathe/profile surfaces face right of the profile's travel direction: run profiles bottom → top for outward faces.
- **No coplanar faces** (z-fighting): recess soil below a pot rim, set trims 1 cm proud, stagger frame pieces in depth.
- **Real detail on small props:** hollow mugs with handles, tapered branches with stemmed leaves, alpha-cutout foliage. Real-world scale in metres.
- **Text in 3D:** trace fonts or stroke fonts (e.g. Hershey) into tubes/prisms for neon signs and lettering.
- **Skinned scene characters** (riders, band members, animated props on the IHXR skeleton): one template skeleton, per-character variants (retint, rest pose = first frame of the main clip so paused ones don't T-pose), props as child nodes under hand/spine bones.
- **Audio:** generate WAVs (OfflineAudioContext or raw PCM) for placeholders; loops need click-free ends. Real recordings must be CC0 or supplied by the person; list sources in `sounds/CREDITS.md`.
- **Sprite sheets** ("Sprite Sheet Lit" shader) for fire, water, mist, slime: rows play **bottom-to-top**, lay frames out in that order. E.g. flames 8×8 at 0.04 s/frame, smoke 4×4 at 0.12 s.
- **UV scroll** (Material Editor Scroll X/Y) for cheap motion: conveyors, flowing water, snow inside a globe (two inward-facing transparent spheres with a tiling flake PNG scrolling at different speeds instead of particles).
- **Preview first:** `E.parseModel(URL.createObjectURL(new Blob([bytes])))` → add `result.root` temporarily (or swap it onto one node), hide panels, screenshot, remove.

### Editing deployed models safely
1. Rebuild the model in the page from its generator.
2. Compare with the loaded scene object: per material, vertex count, index count and a position checksum (sum every 7th float). It must match before you change anything.
3. Apply the change, upload under a new name (`_v2`, `_Lite`, `_Pal`, `_Fix`), swap every node using the old id, move the old card to `_Unused`, save, verify in the world.

## 5. Materials

- **Material Editor overrides (`inlineMaterials`) are stored by part index.** Deleting a child part shifts every override after it. Prefer baking final values into the GLB; only delete parts from meshes with zero overrides.
- Metals: with no reflections, full metalness renders near black. Use ~0.5–0.6, or turn on sky **Reflections**.
- **Emissive / neon:** intensity ~1–1.5 so bloom isn't overwhelming. Signs: texture as both Albedo and Emission Map, colours white, intensity ~1.25.
- Glass: BLEND, metal ~0.6, roughness ~0.04, reflections on.
- Foliage: Cutout + Both Sides + Foliage Wind; bark Opaque + Front with the same wind.
- **Wet / glossy ground:** darken ~15–20 %, raise smoothness (asphalt ~0.8, concrete ~0.6), reflections on.
- Runtime tint from scripts: `SetMaterialColor(obj, 0, "_EmissionColor", c)`, `SetMaterialFloat(obj, 0, "_EmissionIntensity", x)`.

## 6. Colliders, sitpoints, grabbables, physics

- **Colliders:** Mesh for walkable sets (rooms, terrain, ramps), boxes for props (pot/base size), off for wall decor, fixtures and things that must not block. Surface audio (Carpet, Hard...) on floors. "Fit to Object" includes sitpoint preview avatars, so size those by hand.
- **Sitpoints:** the avatar faces the sitpoint's **doc +Z**; don't trust the editor ghost. Place at seat-top centre + 0.16 m toward the facing + 0.04 m up. Each seat is its own mesh with the sitpoint parented to it (Cut → Paste into) so it highlights. Exit points on clear floor. **Facing goes wrong easily:** place one, check it visually or with the person, *then* copy. Leg poses: Chair, Legs Out, Legs Crossed, Standing, T-Pose.
- **Grabbables:** handedness Left/Right/Both, Hold Pose, Grip (Open/Fist), Aim With Camera. Without Aim With Camera the held arm follows each avatar's own pose, so a fixed grip can't stay upright for everyone. Clicking while holding starts a throw aim. A held object's script transform may still report its rest position.
- **Moving platforms:** a collider only updates when each lerp finishes, lerps can arrive in bursts, and baked model animation never moves collision. Use short lerp steps (~0.25 s) and avoid long rides; bob in place instead.
- **Triggers** only re-check when a player moves.
- **Physics** (Jolt-like): spin is capped (~47 rad/s), so small balls skid. For rolling balls: friction 0 (combine Minimum), drag 0, and apply your own deceleration in script. Bouncy rails: bounciness ~0.6 (combine Maximum).

## 7. Scripting (C#, server side)

- `using IHXR.Simulation;` `public class X : ScriptBase` with `public override void Start()`. **Not Unity:** no `UnityEngine`, `MonoBehaviour` or `Update()`; loop with `StartCoroutine(Loop())` + `yield return new WaitForSeconds(0.1f)`. Files, network, threads, tasks and reflection are blocked. The import dialog shows ALL compiler errors as its own dialog that will each need cleared script upload errors occur. Check `https://<host>/scriptapi/reference/` and `/scriptapi/examples/`, or compile-probe and record results in `docs/API_NOTES.md`.
- Scripts run on the instance server (~20 ticks/s); every change is sent to every player. **Don't move things every tick for looks**: use client-side `Lerp...`, model animations, MIDI bindings.
- Units/axes as Unity (metres, degrees, Y up, Z forward); `Quaternion.Euler`, `Slerp`, `LookRotation` exist. Never convert to three yourself. `TeleportPlayer` / `SetPlayerCamera` yaw: 0 = −Z, 90 = +X.
- **Inspector fields:** `SceneObject`, `Sitpoint`, `InventoryItem`, `TextureItem`, `Emote`, `MidiItem`, `AudioItem`, `List<...>`, numbers, bools, strings, `Color`, enums, `Vector3`; `[Secret]` hides a value (API keys, data keys), `[Tooltip]`, `[Range]`.
- **One script per job**, header comment = what it does, what goes in each slot, chat commands. Public fields for anything tunable. A `LogEvents` toggle (off by default); no per-frame logs. Give whole files. Don't send the person on log hunts: read the code and API first; ask once, precisely, if truly needed.
- Scripts talk with `PostScriptEvent` / `ListenForScriptEvent`; document each event's fields in the notes.
- Common calls (verify names against the reference):
  - Objects: `SetLocalPosition/Rotation`, `LerpLocalPosition/Rotation/Scale`, `SetRendererVisible`, `SetLightIntensity/Color`, `SetParticlesPlaying`, `GetComponent<Animation>().Play("Clip")`, `SpawnInventoryItem` / `DespawnSpawnedObject`, rigidbody kinematic/velocity
  - Interaction: `MakeInteractable(obj, cb, ...range, hint)`, `SetInteractable`, `ListenForGrab`, `GrabObject` / `ReleaseObject`, `ShowActionButton` / `ListenForActionButton`, `RegisterSitpointInput`, `OnUserSit` / `OnUserStand`, `SeatPlayer` / `StandPlayer`
  - Players: `OnUserJoin` / `OnUserPart`, `TeleportPlayer`, `SetPlayerScale`, `PlayPlayerEmote(user, Emote)`, `SetPlayerCameraMode`, `TintPlayer` (reset before hiding). A seated user's position is the seat pose.
  - Audio: `PlayAudio`, `PlayAudioLoop`, `StopAudio`, `SetAudioPitch`, `SetAudioVolume`
  - UI: `ShowToast`, `SendScriptChat`, chat commands, `CreatePanel(...)` with `SetMode(PanelMode.Overlay | WorldSpace)` (world space works in VR), labels, buttons, sliders, dropdowns, progress bars; `CreateCanvas(obj, matIndex, w, h, bg)` for live boards (**about 10 script canvases per scene max**: use plain emissive textures for static signs)
  - Persistence: `SaveData` / `LoadData` / `LoadInt` with a `[Secret]` key (change the key to reset a leaderboard)
  - Characters: `SpawnAvatarClone(user, pos[, euler])` → `Sit`, `PlayEmote`, `SetPosition`, `Destroy`; AI NPCs (`AiNpc` walk/follow/say + `AiBrain` with the provider key in a `[Secret]` field)
- **MIDI:** songs load async (`OnMidiLoaded` / `LoadMidi(song, cb)`); find tracks by name/instrument (`info.FindTrack("drums")`), not number. Sound and motion that must look in time run in the viewer: `AddInstrument` / `AddSynth` / `AddDrumKit`, `BindNoteAnimation`, `BindNoteToChild` (direct children in Hierarchy order, lowest note first), `BindDrums`, `BindBeatAnimation`, `SetEmoteTempoSync`. `OnMidiNote` arrives late in batches: logic only. One player id = one shared transport, owned by one conductor script. Limits: 2 MB files, 64 voices per viewer (24 on phones/VR); keep ~6 sounding parts. Pivots matter (keys pivot at the back, beaters at the handle).

## 8. Building blocks that worked

- **Games:** lobby with start/stop buttons, timer from leaving a start trigger, finish trigger records the time, kill triggers → respawn, hazards that launch players, best times on a script-canvas leaderboard saved to the data store, `/restart` `/top` chat commands, a scripted bot run to prove a course is beatable, ease difficulty after the first playtest.  A how to play the game board near leaderboards if often desired too for games.  Make it look like a small billboard 2x human size that visually uses a graphic to show how the game is to be played.  Usually all of this is near the spawn point starting area, which usually makes sense to also be the world's 0,0,0 position.  
- **Vehicles / rides / creatures:** a sitpoint on the vehicle + `RegisterSitpointInput`; rider input follows the seat's facing, so keep a per-model facing sign. State broadcast every ~0.2 s via script events.
- **Moving parades / paths:** one server script moves members around a rounded-rectangle loop; vehicles tow trailers with hitch offsets; riders are child nodes of their float (never members themselves). If riders drift in the world but the doc is right, suspect engine state sync (§11).
- **MIDI band / music shows:** a conductor script with a playlist and a transport panel; an instruments script binds tracks to instrument objects (sound from the instrument; lift, strike, glow animations).
- **Social spaces:** sitpoint-rich seating, media screens (system Media Screen object), light switches that swap scenes of lights, ambient loops; no timers or toasts.
- **Content-only effects:** sprite-sheet fire/water, UV-scroll snow, emissive bulbs instead of lights.

## 9. Lighting, sky and sound

- **One directional light (the Sun).** A second shadow-casting DL doubles every shadow and the shadow pass.
- **"Sky lights the scene"** (Scene Settings → Lighting) drives the Sun from the time of day. For a fixed look turn it **off** (or lock the hour) and aim/colour the Sun yourself. Example evening: hour 18:15, Sun pitch ~14° from the west, intensity ~0.8, colour `#ff8a4c`, thin clouds. Campfire night: 20:30, fog 0.006.
- No ambient/hemisphere light type, and none is added when a scene has its own lights. Sky **Reflections** act as fill; without them shadowed areas go black.
- Physical falloff: point ~10–30, spot ~40–70 (no-shadow neon spills can go higher). Spot Angle is three's half-angle. Fixtures under a light: castShadows off.
- Shadows and real lights are expensive in VR: consider Sun shadows off, cast/receive off on moving things, emissive "bulbs" instead of point/spot lights.
- Fog in Scene Settings (~0.004–0.006); snow weather adds its own haze.
- Sound: area loops as emitters with sensible ranges (fire 2.5–32 m, waterfall 7–80 m), moving things carry their own source, stingers from scripts. Keep always-on spatial sources few and quiet.

## 10. Performance for mobile and VR

Measure first (`__inhousexrFrameProfile`, the platform's "too large for phones" message, Editor → Windows → Scene Stats). Then, in order of payoff:

1. **Remove what can't be seen**: particles (→ sprite sheets / UV scroll), leftover lights and scripts, hidden figures, base planes (`Visible in World` off, keep collider).
2. **Draw distance** on **every** rendering node, children included (1 unit = 5 ft, 100 = 152 m, 0 = none; field `node.drawDistance`). Typical: buildings 0, moving set pieces and their riders/lights ~30, walking characters ~45, hand props ~25. It uses the bounding sphere, so invisible helpers inflate it: check `E.drawDistanceList[i].local.radius`.
3. **Fewer draw calls**, especially on skinned characters (they can't batch): merge every untextured, opaque, non-emissive material into one primitive reading colour + roughness/metal from two tiny palette textures (4×4-texel cells, nearest filtering). Typically 10–16 pieces → 2–6 per character. Keep eyes/textured/emissive/transparent separate.
4. **Triangles**: edge-collapse decimation keeping attributes, locking borders/seams/shared vertices, within the same main bone (0.5–0.6 for characters); thin straight runs, coarser grids away from detail. Don't decimate hero curves. Watch system objects (each Water System surface is a fixed dense grid).
5. **Textures**: 1000² eye textures → 256, atlases 1024 → 512, tiles 256 → 128.
6. **Audio**: fewer, quieter spatial sources (VR uses equal-power panning automatically).

Save, reload the world, compare the profile, and look up close after each step.

## 11. Engine pitfalls (check these first)

- **Draw-distance bounds** must skip hidden/editor-only helpers, zero-area meshes and child scene objects.
- **Nested state sync:** server states for children under an attachment origin must apply relative to the nearest ancestor with a doc, or riders wander.
- **Audio sleep/wake** must reconnect after the volume node, or sounds bypass mute and volume.
- Find who moves an object wrongly by trapping the setter:
  ```js
  const p=obj.position; let v=p.x; Object.defineProperty(p,'x',{configurable:true,get(){return v},set(n){ if(Math.abs(n-v)>0.05) console.log(new Error().stack); v=n; }});
  ```
  then read the bundle near the stack offset (`fetch(document.scripts[0].src)`).

## 12. Wrap-up checklist for each round

- [ ] Every new asset/script is in the project folder; generators updated
- [ ] Uploaded into the project's inventory folder; old cards moved to `_Unused`
- [ ] Saved in the editor; saved JSON equals the document; doc snapshot kept
- [ ] Reloaded and visited the **world**: looked up close, checked the frame profile, tested the play path (sitpoints, grabs, triggers, scripts)
- [ ] Notes updated with a new pass (what, why, ids, coordinates, values, engine findings)
- [ ] Reply: what changed, what you saw in the world, open issues with evidence; for scripted builds also the parts list and test checklist