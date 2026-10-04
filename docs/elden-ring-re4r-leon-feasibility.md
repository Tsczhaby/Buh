# Feasibility study: Leon in Elden Ring

**Candidate games:** *Elden Ring* (host) and *Resident Evil 4* (2023 remake,
source reference)  
**Requested scope:** Leon as the playable character, functioning in Elden Ring's
world  
**Research date:** 2026-10-04

## Executive finding

This is feasible **if “Leon in Elden Ring” means an Elden Ring player with
Leon's appearance and, later, a curated Leon-like Elden Ring moveset**. It is not
a conventional bridge between the two games. There is no reason to load RE
Engine code or RE4R campaign data into Elden Ring for this scope.

The clean technical boundary is:

```text
Leon-shaped mesh + Elden Ring skeleton + Elden Ring animations
                     +
Elden Ring player state, collision, attacks, equipment, events, and saves
```

The player must remain Elden Ring's native player actor internally. That is what
makes Sites of Grace, ladders, doors, Torrent, weapons, damage, death, warps,
cutscenes, enemy targeting, and save data continue to work. Leon is the visual
and authored gameplay layer, not an imported RE4R actor runtime.

### Verdict by scope

| Scope | Feasibility | Main risk |
| --- | --- | --- |
| Leon appearance over a fixed Elden Ring body | **High for a prototype** | Rigging, clipping, materials, face/hair, and asset-distribution rights |
| Appearance plus selected Elden Ring equipment | **High–medium** | Each armor/weapon combination multiplies clipping and visibility work |
| Leon-inspired moveset using Elden Ring actions | **Medium** | Animation events, behavior graphs, balance, and mod conflicts |
| RE4-style handgun with lock-on projectiles | **Medium as an approximation** | It will behave like an Elden Ring projectile/skill, not RE4 aiming |
| Over-the-shoulder free aim, laser sight, ammo/reload, contextual melee | **Low–medium; major custom systems work** | Camera/input hooks, UI, animation state, targeting, native-code maintenance |
| Direct reuse of RE4R skeleton, animations, or gameplay code | **Poor approach** | Incompatible engines/formats plus legal and update risks |

The recommended first release target is the first row only. Do not make a
custom gun, camera, or animation set part of the proof of concept.

## Why Elden Ring is a workable host

The community toolchain can already edit the relevant host-side layers:

- [Smithbox](https://github.com/vawser/Smithbox) supports Elden Ring map, model,
  parameter, text, material, texture, and Havok editing. Its model and file
  browsers can inspect/extract host resources without requiring a permanently
  unpacked game.
- [SoulsFormatsNEXT](https://github.com/soulsmods/SoulsFormatsNEXT) is a .NET
  library that reads and writes FromSoftware containers and formats. The older
  [format reference](https://github.com/JKAnderson/SoulsFormats/blob/master/FORMATS.md)
  identifies FLVER models, TPF textures, PARAM gameplay tables, FMG strings,
  TAE animation events, EMEVD events, and other pieces of the data model.
- [DS Anim Studio](https://github.com/Meowmaritus/DSAnimStudio) supports Elden
  Ring animation archives and TAE events. Its documentation explicitly lists
  events for hit behavior, projectiles, stamina use, invulnerability, parries,
  sounds, effects, aim tracking, and animation cancellation.
- [Mod Engine 2's Elden Ring configuration](https://github.com/soulsmods/ModEngine2/blob/main/installer/dist/config_eldenring.toml)
  documents a loose-file overlay that mirrors FromSoftware's asset paths, such
  as `mod/parts/...partsbnd.dcx`. It also warns that conflicting files—including
  `regulation.bin`—are selected by priority rather than merged.
- [Soulstruct for Blender](https://github.com/Grimrukh/soulstruct-blender) can
  work with FromSoftware FLVER models and Elden Ring material sampler data.

There is an important maintenance warning: the public Mod Engine 2 repository
was archived in July 2024. It remains useful evidence of the loading model and
is widely referenced by the other tools, but an implementation should verify a
currently maintained loader against the exact Elden Ring build before treating
it as a release dependency.

## What comes from RE4R—and what should not

RE4R uses Capcom's RE Engine rather than FromSoftware's engine. Its community
tools confirm that the source assets are not drop-in Elden Ring resources:

- [RE Mesh Editor](https://github.com/NSACloud/RE-Mesh-Editor) imports and
  exports RE Engine `mesh` and `mdf2` material files in Blender and lists RE4R
  as supported. It also handles LODs and texture conversion.
- [REFramework](https://github.com/praydog/REFramework) is a runtime scripting
  and modding platform for RE Engine games. It is useful for studying Leon in
  RE4R, but it cannot run inside Elden Ring and is not part of the target mod.
- Fluffy Mod Manager is an RE Engine mod installer. Like REFramework, it is
  relevant to source-side inspection, not to loading the Elden Ring result.

There is no direct `RE mesh -> Elden Ring player` conversion. Blender is the
practical interchange point, but the work after import is a **manual character
adaptation**: cleanup, topology decisions, material recreation, retargeting, skin
weights, LOD policy, and export into host-native resources.

Do not plan to transfer RE4R's character controller, hit reactions, animation
state machine, IK, camera, weapon code, or inventory. Those are engine systems,
not properties of Leon's mesh.

## The minimum viable implementation

### 1. Lock the target configuration

Record the precise Elden Ring executable/content version, whether *Shadow of
the Erdtree* is required, language, operating system, loader version, Smithbox
version, Blender version, and model plug-in commit. Game and tool updates can
change container compression, parameter schemas, and expected resource versions.

Develop and test in **offline mode only**. Do not attempt to bypass anti-cheat or
make this usable in official online play. Keep an unmodified game launch path
separate from the modded launch path.

### 2. Choose one fixed Elden Ring body target

For a first prototype, replace one known player equipment set—or a deliberately
reserved full-body set—instead of trying to replace every possible naked body
and armor combination.

This provides three useful constraints:

1. Leon can use a fixed silhouette and proportions.
2. The mod can hide redundant Elden Ring head/body pieces.
3. Testing has one deterministic set of part files rather than the entire
   equipment matrix.

The result is still the native Elden Ring player. Armor selection merely chooses
which host part resources display the Leon mesh.

### 3. Build against the Elden Ring player skeleton

Import a known-good Elden Ring player body and armature into Blender. Community
[tool documentation for Elden Ring physics work](https://github.com/tlarok/blender-fbximporter)
points to the male player bundle `fc_m_0000.partsbnd.dcx` as a skeleton source
and requires the custom mesh to be rigged to the same armature.

Then:

1. import or create the Leon reference mesh;
2. fit it to the Elden Ring rest pose and proportions;
3. bind it to the Elden Ring player bones;
4. transfer weights from a known-good host body, then hand-correct shoulders,
   elbows, wrists, coat tails, face, and hair;
5. triangulate and validate bone influences, vertex groups, normals, and seams;
6. preview representative host animations before export; and
7. export a host-native FLVER inside the expected part bundle and path.

Do **not** preserve the RE4R skeleton as the runtime skeleton. Bone names,
hierarchy, bind pose, helper bones, constraints, and animation expectations do
not match. Keeping it would break ordinary host interactions or force a much
larger animation/behavior replacement project.

### 4. Rebuild materials for Elden Ring

Treat RE4R materials as visual references. Recreate the look with Elden Ring
material definitions and texture slots rather than carrying `mdf2` data across.
At minimum validate base color, normal maps, roughness/specular response, alpha,
hair cards, skin, eye shading, mipmaps, and texture compression.

Expect Leon's hair, eyes, skin, jacket, and small accessories to need separate
handling. Cloth simulation is optional for the prototype; rigidly weighted coat
tails are preferable to adding custom physics before the base model is stable.

### 5. Let the host own every interaction

Do not write special interaction code initially. Validate the replacement with
the unmodified Elden Ring player controller:

- idle, walk, sprint, crouch, jump, roll, backstep, and fall;
- one- and two-handed weapons across several classes;
- casting, consumables, gestures, critical attacks, and being hit;
- doors, ladders, levers, chests, Sites of Grace, summoning pools, and lifts;
- mounting/dismounting Torrent;
- death, respawn, warp, cutscene, save, quit, and reload; and
- common camera distances and field-of-view changes.

If these work, “functional interaction with the host world” is already achieved.
Failures at this stage are most likely skeleton, part visibility, bounding,
material, or packaging problems—not evidence that RE4R runtime code is needed.

## Leon-like gameplay as a separate second phase

Once the visual replacement is stable, add flavor using host-native systems.
Keep each feature optional so the cosmetic mod does not require the gameplay
mod.

### Reasonable host-native approximations

- map a kick-like animation to a weapon skill or unarmed heavy attack;
- represent a knife with an Elden Ring dagger and tuned guard counter/parry;
- implement a handgun-shaped weapon whose action emits an Elden Ring projectile;
- use TAE events for projectile timing, stamina, sound, effect, cancel windows,
  and attack behavior; and
- use parameters for damage, range, stamina cost, poise damage, and item text.

These are Elden Ring mechanics wearing a Resident Evil presentation. That is a
feature, not a compromise: enemies, networking assumptions, saves, resistances,
and world events continue to see valid host behaviors.

### Features to defer

Free over-the-shoulder aiming, locational crosshair behavior, a laser sight,
manual reloads, magazine ammunition, RE4 inventory UI, context-sensitive melee,
and RE4-style stagger prompts require coordinated camera, input, HUD, animation,
targeting, state, and combat changes. Some may require a native DLL hook and
will be brittle across game patches.

Treat that bundle as a distinct project with its own design document. Do not
silently expand the character-port milestone to include it.

## File ownership and likely conflicts

A full-body replacement is relatively contained until it starts changing shared
player animation or gameplay files. Maintain a manifest containing every output
path and its origin.

| Layer | Likely host resource | Conflict profile |
| --- | --- | --- |
| Body/head/clothing | `parts/*.partsbnd.dcx` and enclosed FLVER/TPF data | Conflicts with mods replacing the same equipment parts |
| Item names/descriptions | FMG text | Requires language-aware merging |
| Stats/projectiles/actions | `regulation.bin` PARAM rows | High; Mod Engine 2 does not merge competing files |
| Player animation timing | ANIBND/TAE | Very high; moveset mods commonly touch the same archives |
| Player behavior/scripts | behavior/HKS or native hooks | Very high and patch-sensitive |

The cosmetic prototype should contain only part bundles and their necessary
textures/materials. Avoid `regulation.bin`, animation archives, and DLLs until a
specific feature proves they are necessary.

## Distribution and rights gate

The largest nontechnical blocker is distributing Capcom's Leon assets inside a
mod for another publisher's game.

Capcom's [video policy](https://www.capcomusa.com/video-policy/Capcom_Video_Policy.pdf)
expressly says that its permission for gameplay videos is not permission to
create mods or derivative works. Its [published EULA](https://game.capcom.com/eula/eng.html)
also does not provide a clear cross-game asset-port licence. Elden Ring's
[Steam-listed EULA](https://store.steampowered.com/eula/1245620_eula_1) and the
terms applicable in the intended distribution territory must be reviewed too.

Accordingly:

- do not publish extracted RE4R meshes, textures, sounds, animations, or code
  without express permission;
- owning both games does not by itself grant redistribution rights;
- a private technical experiment and a downloadable public mod have different
  risk profiles;
- a “user supplies both games” installer reduces direct asset distribution but
  is not automatically authorized and would require a reliable local converter;
- an original, from-scratch character model avoids copying game files but does
  not automatically resolve character, likeness, trademark, or derivative-work
  questions; and
- obtain permission or qualified legal advice before a public release.

For a portfolio-safe prototype, use an original agent character with similar
functional goals while proving the complete Elden Ring pipeline. Swap in any
licensed character art only after the rights question is resolved.

## Recommended proof-of-concept plan

### Gate 0 — rights and environment

- Decide private experiment versus public distribution.
- Preserve proof of legitimate game ownership and clean installations.
- Freeze exact game/tool versions and establish an offline modded launch.
- Confirm that a trivial loose-file replacement loads and uninstalls cleanly.

**Exit criterion:** a documented, reversible host setup and an acceptable asset
provenance plan.

### Gate 1 — grey-box body

- Export a simple original mannequin on the Elden Ring male player skeleton.
- Replace one full-body equipment set.
- Test locomotion, combat, interactions, Torrent, death, and save/reload.

**Exit criterion:** no crashes, exploding vertices, missing materials, invisible
body parts, or interaction regressions in the minimum test matrix.

### Gate 2 — character-quality visual pass

- Adapt the final licensed/original mesh.
- Complete weights, materials, hair, eyes, LODs, and clipping fixes.
- Test all target weapons and cutscenes; explicitly document unsupported armor.

**Exit criterion:** a standalone cosmetic package containing no gameplay tables,
animation archives, native code, or unlicensed source files.

### Gate 3 — one gameplay feature

- Add exactly one optional feature, preferably a kick-like weapon skill using
  host-native animation events and combat behavior.
- Measure conflicts and provide a clean uninstall/rollback path.

**Exit criterion:** the cosmetic package still works without the gameplay add-on,
and the add-on does not corrupt existing saves.

Only after Gate 3 should a handgun approximation be considered.

## Acceptance test for “functional Leon”

A first release is complete when the player can:

1. start or load a normal Elden Ring save offline;
2. equip the designated appearance and look correct at rest and in motion;
3. traverse, fight, use items, interact with world objects, ride Torrent, die,
   respawn, warp, trigger a cutscene, save, and reload;
4. remove the mod and return to unmodified visuals without changing the save;
5. reproduce the installation from a clean supported game build; and
6. account for the provenance and distribution permission of every shipped file.

That delivers the requested experience without pretending that two unrelated
engines can share a character implementation directly.
