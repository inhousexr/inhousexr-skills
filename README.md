# InHouseXR skills for Claude

Skills that teach Claude how to build on InHouseXR (IHXR) platforms such as SapphireXR.

| Skill | What it does |
|---|---|
| `inhousexr-scene-builder` | Plan and build any scene: generate meshes/textures/sounds, upload, place, script in C#, light, optimize, save and verify. |
| `inhousexr-scene-optimize` | Slim a scene for phones and VR. |
| `inhousexr-avatar-builder` | Avatars, rigged clothing, accessories, hair, shoes and emotes for the Avatar Editor. |

## Install

**Claude Code:**
```
/plugin marketplace add inhousexr/inhousexr-skills
/plugin install inhousexr@inhousexr-skills
```
Or copy a folder from `plugins/inhousexr/skills/` into `~/.claude/skills/`.

**Claude app (claude.ai / desktop):** download a zip from `zips/`, then Customize > Skills > + > Upload a skill.

## Notes
- Claude works in your platform's page through a browser it can control, and needs a project folder to keep every generated file.
- The skills never publish scenes, never delete inventory items, and keep secrets private.
