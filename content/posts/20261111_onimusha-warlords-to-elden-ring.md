---
title: "Planning a Port: Onimusha: Warlords into Elden Ring"
date: 2026-11-11
draft: true
tags: ["modding", "elden-ring", "onimusha", "reverse-engineering"]
summary: "A practical plan for bringing Onimusha: Warlords into Elden Ring's engine, borrowing the converter approach from the Morrowind port."
---

After looking at how InfernoPlus ported Morrowind into Elden Ring[^1], I wanted to work out what it would take to do the same with **Onimusha: Warlords**. I originally sketched this with Sekiro as the target, but Elden Ring is the better first choice, for two reasons:

1. **Tooling.** Elden Ring has the most actively developed FromSoftware modding tools, including collision and navmesh support that older games lack (more on that below).
2. **Prior art.** JortPob, the open-source Morrowind converter, already solves "generate new maps for Elden Ring from another game's data"[^2]. Its code and setup are a working reference.

This post is a plan, not a finished project.

## The core challenge

Onimusha: Warlords is built on **pre-rendered backgrounds**: each room is a fixed camera looking at a painted image, with 3D characters drawn on top[^3]. Characters, enemies and items are real models. The castle is not.

So the Morrowind approach, where a program reads the old level files and writes new ones, only gets you part of the way. The plan has to combine:

- **A converter** for everything that is 3D or data.
- **A rebuild** of the environments as real geometry.

The trick I want to try for the second part is **camera projection mapping**: recover each room's camera, build simple blockout geometry, and project the original painted background onto it from that camera. Near the original camera angle, that looks almost identical to the original game. It won't hold up from every angle, but it's a far better starting point than modelling Inabayama Castle from scratch.

## Legal approach

I'll follow the JortPob model. The mod ships no Morrowind data and requires players to own both games[^4]. For Onimusha, that means:

- Publish only the converter and original work (blockout and environment models, scripts, configs).
- The converter reads the player's own copy of Onimusha: Warlords and builds everything else, including the projected background textures, on their machine.

## Toolchain

**Elden Ring side**

- **UXM Selective Unpack**, to unpack the game. JortPob relies on unpacked Elden Ring files as a source of assets[^2].
- **ModEngine3**, to load the mod from a separate folder without touching the game install. JortPob's README includes a working profile[^2].
- **Smithbox**, for map, param, model, text and material editing[^5]. As of release 2.2.5, it also has:
  - a Havok editor that covers map collision and navmesh files for Elden Ring;
  - the ability to generate a collision mesh from a FLVER model, which is exactly what's needed to turn a rebuilt room into walkable geometry[^6].
- **DarkScript3**, for EMEVD event scripts; Smithbox points to it for that job[^5].
- **SoulsFormats**, the .NET library for FromSoftware formats that most of these tools build on. The converter will be C# on top of it.

**Onimusha side**

- **The 2019 remaster on Steam**, the version that came to Windows that year[^7].
- **A hex editor**, to identify the archive and file formats. I haven't confirmed which engine or archive format the remaster uses, so this is step one.
- **Noesis**, for previewing and converting model, texture and animation formats, extended with Python plugins where formats are unknown[^8].
- **Blender**, for blockouts, projection mapping and cleanup.

There is evidence the assets are reachable: a fan-made 4K texture pack for the remaster replaces both background and model textures[^9].

## Mapping Onimusha systems to Elden Ring

| Onimusha: Warlords | Elden Ring equivalent |
|---|---|
| Rooms with fixed cameras | Map pieces and map layouts in a new map ID |
| Painted backgrounds | Projection-mapped textures on rebuilt geometry |
| Room walk-mesh and collision | Collision generated from FLVER in Smithbox |
| Doors and room transitions | Asset objects plus EMEVD warp events |
| Red souls (upgrade currency) | Runes |
| Yellow souls (health) | HP restore on pickup via an EMEVD event, or a Flask refill |
| Blue souls (magic) | FP restore |
| Oni gauntlet soul absorption | Event-driven pickup triggered on enemy death |
| Lightning, fire and wind Oni orbs | Weapons, each with its own Ash of War |
| Issen critical counter | Guard counter, or a custom weapon skill |
| Herbs and medicine | Consumables in item params |
| Save mirrors | Sites of Grace |
| Puzzle boxes and locked doors | EMEVD flag logic plus custom menus |
| Genma enemies | New characters with converted models |

## The plan

### Milestone 0: Reconnaissance

Goal: know what is in the game files before writing any converter code.

1. Inventory the install folder and identify container formats from their headers.
2. Extract textures and confirm which files are backgrounds and which are model textures.
3. Get one character model (Samanosuke) into Noesis or Blender with its skeleton.
4. Find the per-room data. This is the most important reverse-engineering target. Each room should have:
   - camera position, rotation and field of view;
   - floor and collision geometry, so characters line up with the painting;
   - door and exit links;
   - enemy spawns and item placements.

If the room data can't be found or decoded, the plan changes. Rooms would have to be blocked out by eye from the backgrounds, which is slower but still workable.

### Milestone 1: One room, walkable

Goal: stand in one Onimusha room inside Elden Ring.

1. Fork or study JortPob's map generation to learn how it registers a new map ID and writes map layouts, then strip it down to a minimal "write one map" tool.
2. In Blender, use the room's floor and collision data and the recovered camera to block out the room.
3. Project the background image from the recovered camera onto the blockout and bake it to a texture.
4. Export the room as a FLVER map piece.
5. Generate collision from that FLVER in Smithbox[^6].
6. Place the map piece, collision and a player spawn point in a map layout.
7. Add a debug warp, as JortPob does with its warp room in the Stranded Graveyard[^2].

This milestone proves the whole environment pipeline end to end. Everything after it is repetition and content.

### Milestone 2: Automate the rooms

Goal: the converter builds room blockouts and projected textures for every room without manual work.

1. Generate blockout geometry directly from each room's collision data.
2. Script the projection and bake step, using Blender run headless from the converter.
3. Handle rooms with several camera angles. Project from each camera and blend, preferring the camera that sees each surface most head-on.
4. Connect rooms using their door links, converting them to EMEVD warp events or, better, to physically adjacent geometry in one continuous map.

### Milestone 3: Characters and enemies

Goal: Samanosuke as the player character, and one Genma enemy that fights back.

1. **The player.** Start with an armour set that reskins the Tarnished as Samanosuke, rather than a new player character. It reuses Elden Ring's player skeleton and animations, which avoids the hardest work. An Onimusha-themed armour mod for Elden Ring already exists[^10].
2. **Enemies.** Convert Genma models to FLVER, then retarget them onto the skeleton of an existing Elden Ring enemy with a similar body shape. That gives you working animations and AI to start with.
3. **Navmesh.** Generate or hand-author navmeshes for the room so enemies can move. Smithbox supports editing Elden Ring navmesh files[^6], but navmesh generation is the step that has worried the Morrowind team most[^11], so budget for it.

### Milestone 4: Systems

1. Set the three Oni weapons up as weapons in params, each with an elemental Ash of War.
2. Implement soul absorption using EMEVD: when an enemy dies, spawn coloured pickups that grant runes, HP or FP.
3. Port the puzzle and lock logic to EMEVD event flags. The soulsmods project publishes EMEDF instruction references for each game (Sekiro's is linked here; check for the Elden Ring equivalent)[^12].
4. Add item pickups, the shop/upgrade mirror and the save points as Sites of Grace.

### Milestone 5: The full castle

Run the converter over every room, connect them, populate enemies and items, and port the boss encounters. The bosses will need the most hand-work, because their attacks have to be rebuilt on top of Elden Ring animations.

## Risks

- **Unknown formats.** If the remaster's model or room formats are undocumented, Milestone 0 could take longer than everything else combined.
- **Projection artifacts.** Painted backgrounds stretch badly when seen from angles they weren't painted for. Some rooms will need real modelling.
- **Navmesh.** This is the problem most likely to stall progress, as it has for the Morrowind project[^11].
- **Feel.** Onimusha's combat is fast and arcade-like. Elden Ring's is heavier. A faithful conversion of the content will still play like Elden Ring unless animations and timings are reworked.

## Next step

Milestone 0. Install the remaster, open the data files in a hex editor, and find out what we are dealing with.

## References

[^1]: [InfernoPlus, "I ported Morrowind to Elden Ring" (YouTube)](https://www.youtube.com/watch?v=n89NFtIUWDI)
[^2]: [infernoplus/JortPob on GitHub](https://github.com/infernoplus/JortPob)
[^3]: [Wikipedia, "Onimusha: Warlords"](https://en.wikipedia.org/wiki/Onimusha:_Warlords)
[^4]: [Wccftech, "Here's Morrowind Running in the Elden Ring Engine"](https://wccftech.com/heres-morrowind-running-elden-ring-engine/amp/)
[^5]: [Smithbox on GitHub](https://github.com/vawser/Smithbox)
[^6]: [Smithbox 2.2.5 release notes](https://github.com/vawser/Smithbox/releases/tag/2.2.5)
[^7]: [Lowyat.NET, "Onimusha: Warlords Is Getting Remastered For Current-Gen Platforms"](https://www.lowyat.net/2018/168824/onimusha-warlords-is-getting-remastered-for-current-gen-platforms/)
[^8]: [AppNee, "Noesis: view and extract models and animations from 3D games"](https://appnee.com/noesis/)
[^9]: [GamePressure, "A fan has remastered the first Onimusha into 4K"](https://www.gamepressure.com/newsroom/a-fan-has-remastered-the-first-onimusha-into-4k/zf7bb4)
[^10]: [Nexus Mods, "Elden Ring x Onimusha"](https://www.nexusmods.com/eldenring/mods/4757?tab=docs)
[^11]: [GamesRadar, "Mod team's unhinged quest to port Morrowind into Elden Ring"](https://www.gamesradar.com/games/the-elder-scrolls/mod-teams-unhinged-quest-to-port-the-elder-scrolls-morrowind-into-elden-ring-is-almost-already-at-full-parity-for-npcs-damn-this-project-has-hands)
[^12]: [soulsmods, "Sekiro EMEDF for DarkScript3"](https://soulsmods.github.io/emedf/sekiro-emedf.html)
