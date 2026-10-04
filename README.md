# Building a “bridge mod” between two games

This note is a technical starting point for a mod that makes content from two
existing games playable as one continuous experience. “Bridge mod” is a useful
description, but not a standard technical category. Projects usually call
themselves a **total conversion**, **campaign merger**, or **cross-game port**.

The short version: these projects almost never run two game executables and join
them at runtime. They choose **one game as the host engine**, convert the other
game’s data into resources that host understands, patch conflicts, and add a
small piece of original content that moves the player between campaigns.

## Two proven examples

### Tale of Two Wastelands (TTW)

TTW makes *Fallout 3* and its DLC playable inside *Fallout: New Vegas*, then adds
travel between the Capital Wasteland and Mojave. Its public documentation calls
it a total conversion and requires legitimate, clean installations of both
games. The important architectural point is that the result runs in the New
Vegas engine; it is not synchronization between two running games.

The installer model is as important as the runtime model. Instead of shipping a
download containing Bethesda’s *Fallout 3* assets, the installer reads the
player’s own installations and builds the converted mod locally. This also gives
the installer an opportunity to reject unsupported editions, check source-file
versions, apply deterministic transformations, and write a troubleshooting log.

Sources:

- [TTW documentation hub](https://mod.pub/ttw/133/docs)
- [The Best of Times introduction and installation model](https://thebestoftimes.moddinglinked.com/intro.html)
- [TTW’s older conversion workflow](https://taleoftwowastelands.com/viewtopic.php%40t%3D1015)

### Baldur’s Gate: Enhanced Edition Trilogy (EET)

EET imports the earlier Enhanced Edition campaigns into *Baldur’s Gate II:
Enhanced Edition*. It merges world maps, maintains one party and protagonist,
and patches resources and scripts so the saga can continue in one host game.
Unlike a one-off binary merge, it uses WeiDU’s installer/patch language and
documents conventions that other mods can target.

EET demonstrates the less visible half of a bridge: continuity data. Its modder
notes describe generated import code, campaign transitions, world-map support,
and naming conventions. Joining two maps is easy compared with deciding which
variables, companions, inventory, experience, and plot decisions survive the
crossing.

Sources:

- [EET project overview](https://github.com/Gibberlings3/EET)
- [EET modder’s notes](https://github.com/Gibberlings3/EET/blob/master/EET/docs/Modder%27s%20Notes.html)
- [WeiDU documentation](https://weidu.org/WeiDU/README-WeiDU.html)

## What is actually being built?

Think of the finished project as five layers:

```text
player-owned Game A files ─┐
                           ├─> converter/installer ─> generated host-format mod
player-owned Game B files ─┘                            │
                                                       v
                                             Game B host executable
                                                       │
                         bridge quest + state policy + compatibility patches
```

1. **A host runtime.** One executable owns rendering, input, physics, scripting,
   saving, UI, and plugin loading.
2. **A conversion pipeline.** It reads source records/assets and emits data in
   the host’s formats. Ideally the same inputs always produce the same output.
3. **An adaptation layer.** Patches reconcile different record schemas,
   mechanics, scripts, animations, shaders, and assumptions.
4. **A continuity bridge.** A door, vehicle, chapter transition, or menu moves
   the player between campaigns and deliberately maps persistent state.
5. **A distribution-safe installer.** It ships the team’s code and deltas, then
   derives publisher-owned output from files already present on the user’s PC.

This is closer to building a compiler and compatibility layer than to making a
normal quest mod.

## First feasibility gate

Answer these before writing the bridge scene. If several answers are “no,” the
project is probably a remake rather than a conversion.

| Question | Why it matters |
| --- | --- |
| Do both games share an engine lineage and data formats? | Shared record, archive, model, animation, and script formats can turn a rewrite into a conversion. |
| Can the host load multiple world spaces/campaigns? | If not, engine hooks or extensive map restructuring may be required. |
| Is there an official editor or a mature community toolchain? | You need parsers, validators, archive tools, script compilers, and record conflict inspection. |
| Can every source resource be represented in the host? | Removed script opcodes, animation types, UI systems, or hard-coded mechanics need replacements. |
| Can the project legally require both games and build locally? | Do not design around redistributing another game’s copyrighted files. |
| Are executable modification and anti-cheat absent or expressly allowed? | A single-player, offline, officially moddable host is much safer technically and contractually. |
| Can a tiny vertical slice be completed? | One room, one NPC, one quest, one transition, save, and reload should work before bulk conversion. |

The best candidates are adjacent releases built on related technology. Games
with unrelated engines, online services, or substantially different gameplay
models usually require manual recreation of content and should be scoped and
described that way.

## Technical workstreams

### 1. Inventory both games

Create a machine-readable census rather than browsing assets by hand:

- executable and content versions, storefront editions, DLC, and language;
- archives and loose files, with cryptographic hashes;
- records by type and identifier;
- maps/cells, navigation data, spawn points, and world-map links;
- meshes, skeletons, animations, textures, materials, audio, video, and fonts;
- quests, dialogue, scripts, global variables, and localization tables;
- UI definitions and configuration; and
- save-game data that must persist.

Produce a report such as `inventory.json` plus totals and duplicate identifiers.
This becomes the baseline for completeness tests and helps expose regional
editions before users do.

### 2. Select the host deliberately

Usually choose the later or more extensible game, but score both candidates.
Consider engine stability, address space, scripting features, renderer and asset
compatibility, save limits, editor quality, community tooling, and publisher
policy. Once chosen, **all source behavior must be expressed in host semantics**.
Avoid an architecture that launches each executable in turn: inventory, quest
state, mods, saves, and engine-specific object identity make that handoff brittle.

### 3. Map identities and namespaces

Collisions are inevitable when both games call something `Player`, use the same
numeric record ID, or reference resources by short filename.

Maintain a durable mapping database:

```text
source game + source ID + source type
                -> host plugin + host ID + converted path + converter version
```

Reserve ID ranges or use a stable prefix, preserve the mapping between builds,
and never let conversion order allocate IDs implicitly. Rewrite every reference,
including references hidden in scripts, dialogue, leveled lists, map markers,
and save-visible globals. A changed mapping can invalidate saves even when the
converted content otherwise looks correct.

### 4. Convert data, then adapt behavior

Separate mechanical conversion from authored fixes:

```text
extract -> parse -> normalize -> remap IDs -> transform -> validate -> package
                                                │
                                     handwritten overrides/patches
```

- **Records:** translate fields and defaults; do not blindly copy binary
  structures that only happen to look similar.
- **Scripts:** recompile from source where possible. Otherwise build an opcode
  compatibility table and flag every unsupported instruction for human review.
- **Worlds:** convert cells, links, collision, navigation, lighting, water,
  encounter zones, and streaming metadata—not just visible geometry.
- **Art:** retarget skeletons/animations when required and translate materials.
  Preserve original audio rather than recompressing repeatedly.
- **UI and localization:** account for changed widgets, font metrics, string-table
  IDs, plural rules, and every supported language.
- **Gameplay:** create an explicit table for skills, perks, damage formulas,
  economy, level scaling, crafting, reputation, and difficulty.

Keep handwritten fixes outside generated output. That makes regeneration safe
and lets tests distinguish converter bugs from creative balance choices.

### 5. Design the state contract

Write down what crosses the bridge before implementing it:

| State | Typical policy |
| --- | --- |
| Character appearance/name | Preserve, converting unsupported options. |
| Level, attributes, skills | Preserve or transform with a documented formula. |
| Inventory/currency | Preserve, replace unmapped quest items, prevent duplicates. |
| Companions | Dismiss, transport only compatible companions, or restore on return. |
| Active quests | Pause source-region quests; never leave required actors running in an unloaded world. |
| Factions/reputation | Map only intentional equivalents; otherwise keep namespaces separate. |
| Time/weather | Decide whether worlds share one clock. |
| One-shot events | Store an idempotent transition flag so re-entry cannot replay setup. |

Treat travel as a transaction: validate prerequisites, snapshot state, perform
the transition, commit new state, and recover cleanly if loading fails. Test
travel in both directions, repeated travel, death/reload around the boundary,
and saves created on either side.

### 6. Build the bridge last

The visible connector can be small—a train, portal, ship, chapter cutscene, or
world-map node. It should not contain conversion logic. It should call a tested
transition routine and provide an in-world explanation for mechanical changes.

Good bridge content also solves pacing: minimum level, lost equipment, time
skip, respec, or difficulty warning. These are design decisions, not merely
technical necessities.

### 7. Package without shipping the games

A conservative installer should:

1. locate or ask for both installations;
2. verify supported editions and exact input hashes;
3. refuse missing DLC or modified inputs unless explicitly supported;
4. build into a staging directory, never destructively edit either game;
5. extract/convert only on the user’s machine;
6. emit a normal host-game mod for a mod manager to mount;
7. record tool version, source hashes, warnings, and every applied patch; and
8. support clean uninstall by deleting generated output.

Do **not** treat local conversion as automatic legal permission. Read the EULAs,
modding policies, and tool licences for the exact games and storefronts, and ask
the rights holders when the rules are unclear. Bethesda’s current terms, for
example, retain broad control over game mods, and its published community rules
forbid copyright infringement and illegal distribution. This is practical risk
management, not legal advice.

Relevant policies:

- [Bethesda Terms of Service](https://bethesda.net/data/tos/en.html)
- [Bethesda Community Standards](https://bethesda.net/en-US/news/bethesda-softworks-community-standards)
- [WIPO overview of legal issues raised by mods](https://www.wipo.int/edocs/mdocs/enforcement/en/wipo_ace_15/wipo_ace_15_4.pdf)

## A sensible project architecture

```text
bridge-project/
├── docs/
│   ├── compatibility-matrix.md
│   ├── state-contract.md
│   └── supported-builds.md
├── schemas/                 # parsed source and normalized representations
├── mappings/                # stable IDs, types, mechanics, script opcodes
├── converter/
│   ├── readers/             # source archives and records
│   ├── transforms/          # pure, independently tested transformations
│   ├── writers/             # host records/assets
│   └── validators/
├── patches/                 # reviewed overrides; never generated in place
├── bridge-content/          # quest, map marker, dialogue, transition script
├── installer/
├── tests/
│   ├── fixtures/            # tiny synthetic data, not copyrighted game assets
│   ├── golden/              # expected converter output where distributable
│   └── integration/
└── tools/
```

Use a neutral intermediate representation when formats differ enough to warrant
it. Readers translate Game A into normalized objects; writers emit Game B data.
Pure transforms are far easier to test than a single script that edits archives
in place.

## Testing strategy

### Automated checks

- every source record has a converted, intentionally skipped, or unsupported
  disposition;
- all emitted references resolve;
- IDs and output hashes are deterministic across two clean builds;
- scripts compile and unsupported opcodes fail the build;
- archive manifests contain no accidentally bundled source assets;
- all world links and navigation islands pass validators;
- all localized strings resolve;
- a clean rebuild matches the checked conversion manifest; and
- install, upgrade, and uninstall leave the original installations unchanged.

### In-game matrix

Test clean host installs separately from popular mod stacks. Cover new game,
both campaign starts, transition both ways, every ending, fast travel, companion
dismissal, scripted scenes, save/load, death/reload, and long-running saves.
Include every supported game version, storefront, DLC set, and language.

For debugging, make the converter log source ID, destination ID, transformation,
warning severity, and patch provenance. Add an in-game diagnostic command that
prints campaign, transition phase, key globals, and plugin versions.

## Compatibility with other mods

A bridge changes the host’s effective “base game,” so ordinary compatibility
rules are not enough. Publish:

- a canonical plugin/load order;
- records and scripts intentionally owned by the bridge;
- renamed IDs and paths;
- safe extension points and events;
- known incompatible categories, not only individual mods;
- a patching guide and machine-readable version API; and
- save-compatibility promises for each release.

Patch records surgically rather than replacing whole tables. EET’s use of WeiDU
illustrates this approach: commands such as `COPY_EXISTING` read resources from
the installed game and patch them, which composes better than distributing a
complete replacement resource.

## Recommended first milestone

Do not begin by converting a whole campaign. Build this vertical slice:

1. Detect two clean, supported installs and hash a few known files.
2. Parse one source record and one asset without modifying either install.
3. Convert a single interior room into a standalone host mod.
4. Convert one NPC with one line of localized dialogue and one simple script.
5. Enter through a host-game door, complete a one-step quest, return, save, and
   reload on both sides.
6. Build the output twice and prove that file manifests and hashes match.
7. Have a second person reproduce the build from clean installations.

Only then add a small exterior cell and navigation, followed by one complete
quest. Bulk conversion should wait until the pipeline, mappings, and diagnostics
survive those milestones.

## Decisions needed for a concrete plan

The exact implementation depends overwhelmingly on the two games. Before tool
selection or estimates, record:

1. the exact games, editions, patches, DLC, storefronts, and target OS;
2. which game should be the host, if there is a preference;
3. whether they share an engine or supported import/export formats;
4. whether the desired result is seamless travel, a one-way character import,
   or merely sequential campaigns;
5. what state must carry over;
6. whether script extenders or executable hooks are acceptable; and
7. whether this is a private experiment or a publicly distributed project.

With those answers, the next research pass should inspect the games’ actual
editors, file formats, script runtimes, existing open-source tools, licences, and
publisher policies. That is the point at which an architecture can move from the
general pipeline above to specific record types, APIs, and milestones.
