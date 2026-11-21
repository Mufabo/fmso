---
title: "The State of Open Source Game Engines, Late 2026"
date: 2026-11-08
draft: true
description: "Where the main open source game engines stand at the end of 2026: Godot, Bevy, Fyrox, O3DE, Stride, Defold, MonoGame, raylib and the GPL id Tech engines, with their latest releases, the games made with them, how they compare to Unity and Unreal, and which one fits which kind of project."
summary: "2026 was a busy year for open source game engines. Godot shipped two big releases and drew a hard line on AI-written code, Fyrox hit 1.0, Bevy landed its new scene system, and O3DE, Stride, Defold, MonoGame and raylib all shipped major updates. Meanwhile, the old id Tech engines live on through source ports. Here is where each one stands."
tags: ["gamedev", "game engines", "open source", "godot", "bevy", "rust", "csharp", "id tech", "doom", "quake", "unity", "unreal engine"]
categories: ["Programming"]
---

A few years ago, "open source game engine" mostly meant "Godot, plus some hobby projects". That is no longer true. At the end of 2026 there is a handful of serious, actively developed engines and frameworks, each with a clear audience, and almost all of them shipped a major release this year.

This post is a snapshot of where things stand in November 2026: what each engine is, what its latest release brought, and who it's for. At the end there is also a section on the oldest open source engines of all, id Software's GPL id Tech engines, which are still very much alive.

## What is a game engine?

A game engine is the software that does the work every game needs, so developers can focus on what makes their own game different. Instead of writing everything from scratch, you get ready-made systems for:

- **Rendering:** drawing 2D sprites or 3D scenes on screen, with lighting, shadows and effects.
- **Physics:** gravity, collisions and things bouncing off each other.
- **Input:** reading the keyboard, mouse, gamepad or touchscreen.
- **Audio:** playing sound effects and music.
- **Scripting:** a way to write the game's own logic, in a language like C#, GDScript, Lua or Rust.
- **Assets:** loading and managing models, textures, sounds and levels.
- **Platforms:** building the same game for Windows, macOS, Linux, mobile, the web or consoles.

Most full engines also come with an **editor**: a visual tool where you place objects in a level, tweak their properties and press play to test. Not everything in this post works that way, though. It helps to know three rough categories:

- **Engines** such as Godot, Unity or Unreal give you all of the above, including an editor.
- **Frameworks** such as MonoGame give you the building blocks as a code library, but no editor; you build the game, and often your own tools, in code.
- **Libraries** such as raylib are smaller still: a simple API for graphics, input and sound, and the rest is up to you.

Which one you want depends on how much you'd rather build yourself and how much you'd rather have done for you.

## The short version

| Engine | Language | License | Latest stable (Nov 2026) | Best for |
| --- | --- | --- | --- | --- |
| **Godot** | GDScript, C#, C++ | MIT | 4.7 (4.8 in beta) | Most indie 2D and 3D games |
| **Bevy** | Rust | MIT / Apache 2.0 | 0.20 | Rust programmers who like ECS and don't mind churn |
| **Fyrox** | Rust | MIT | 1.0 | Rust with a Unity-style editor |
| **O3DE** | C++, Lua, Script Canvas | Apache 2.0 | 26.05 (26.10 imminent) | Large 3D projects, simulation, robotics |
| **Stride** | C# | MIT | 4.4 | C# developers who want a full 3D editor |
| **Defold** | Lua | Defold License (source available) | 1.13 | Small, fast mobile and web games |
| **MonoGame** | C# | MS-PL | 3.8.5 | Code-first C# games, XNA-style |
| **raylib** | C (many bindings) | zlib | 6.0 | Learning, tools, small games, embedded |
| **id Tech 1–4** (via source ports) | C, C++ | GPL | Varies by port | Retro shooters, modding, studying classic engines |

## Godot: the default, now with guard rails

Godot is still the centre of gravity. Two feature releases shipped this year, a third is in beta, and the project spent a good part of the year deciding how it wants to grow.

**Godot 4.6** came out on 27 January[^1]. The headline change was that Jolt, which had been an experimental option since 4.4, became the default physics engine for new 3D projects. It also brought a new default editor theme, docks that can be moved and floated freely, inverse kinematics, reworked screen-space reflections, unique node IDs that survive scene reorganisation, and the first version of **LibGodot**, which lets you embed Godot as a library inside another application on Linux, Windows and macOS[^2].

**Godot 4.7** followed in June[^3]. The big items:

- **HDR output** on Windows, macOS, Linux (Wayland), iOS and visionOS. Godot always rendered in HDR internally; now it can actually send HDR to the display.
- **AreaLight3D**, a new node for rectangular light sources (windows, screens, billboards) with soft shadows.
- **The new Godot Asset Store**, which is replacing the old Asset Library and is still in beta[^3].
- A stable **Godot Android Build Environment**, so you can export and publish from an Android device, plus offset transforms for Control nodes that let you move, rotate and scale UI elements without breaking container layouts[^4].

**Godot 4.8** reached beta 1 in mid-October and beta 2 at the end of October, so a stable release is likely around the turn of the year[^5].

The more interesting Godot story of 2026 isn't a feature, though. It's governance. In June the Godot Foundation published a written **vision statement** for the engine, formalising ideas that had until then lived mostly in the heads of the founders and core maintainers[^6]. Its core message is pragmatic: Godot is a tool for shipping games, and a smaller engine whose features work completely is better than a larger, brittle one[^7].

A week later came **new contribution policies**[^8]. Godot has been growing fast, its review backlog with it, and a wave of AI-generated pull requests made that worse. The Foundation's response: new contributors (three or fewer merged PRs) need explicit maintainer approval before submitting new features or significant refactors, and a stricter AI policy bans autonomous agents and vibe-coded submissions and prohibits AI-generated substantial pieces of code[^8][^9]. Whatever you think of AI coding tools, it's a clear statement that review time, not code, is the scarce resource in a large open source project.

**Games made with it:** the biggest is **Slay the Spire 2** from Mega Crit, which moved to Godot and sold 3 million copies in its first week of Early Access in March 2026[^39]. Before that, Godot's best-known commercial games included **Brotato**, **Dome Keeper**, **Cassette Beasts**, **Buckshot Roulette**, **Halls of Torment**, **The Case of the Golden Idol**, **Cruelty Squad** and **Backpack Battles**[^38].

**Who it's for:** almost anyone making a 2D game, and most small-to-medium 3D games. If you don't have a strong reason to pick something else, start here.

## Bevy: the Rust engine that keeps getting closer

Bevy is a data-driven engine built entirely around an entity component system (ECS), written in Rust. It still has no 1.0 and no committed date for one, and it still ships a new version with breaking API changes roughly every three months[^10]. If you pick Bevy, you sign up for regular migrations.

That said, 2026 was a strong year:

- **Bevy 0.18** (January) added atmosphere occlusion, PBR shading fixes and Solari raytracing improvements[^11].
- **Bevy 0.19** (June) finally landed **BSN, the Bevy Scene Notation**, the long-awaited next-generation scene system, along with more GPU-driven rendering work[^12]. It had 261 contributors and 1,185 merged pull requests.
- **Bevy 0.20** (8 October) rewrote **Solari**, Bevy's real-time path-traced renderer, making it faster and more accurate and bringing it to macOS via Metal. It also added DLSS Ray Reconstruction support, officially adopted the **WESL** shader language (dropping Bevy's custom WGSL dialect), added more editor-oriented "Feathers" UI widgets, `em`/`rem` units in Bevy UI, HDR display support and a pan-orbit camera[^13][^14].

The pattern is clear: the foundations for a real editor (a proper scene format, a UI toolkit aimed at tools, better asset processing) are falling into place one release at a time. The editor itself still isn't here.

**Games made with it:** Bevy's flagship is **Tiny Glade**, the cosy castle builder from the two-person studio Pounce Light, released in September 2024 and nominated for a BAFTA Games Award for Technical Achievement[^40]. Beyond that, Bevy games are mostly small indie and game-jam titles; there's no second commercial hit of that size yet.

**Who it's for:** programmers who want to write their game in Rust, like the ECS model, and are comfortable working mostly in code. Not yet the choice for a team that needs artists and designers working in an editor.

## Fyrox: Rust, 1.0, and an editor

Fyrox (formerly rg3d) is the other big Rust engine, and it takes the opposite approach to Bevy: it has a full editor and describes itself as "like Unity, but in Rust". In March 2026, after seven years of development, it shipped **Fyrox 1.0**[^15][^16].

1.0 is more a consolidation than a feature drop: a project export CLI for CI/CD builds, a text-based native asset format that can be merged in version control, a UUID-based resource system, strongly typed handles for scene and UI entities, and a fully updated book[^15]. The team is honest about its limits: it is small, has little funding, and planned to pause major features for a few months after 1.0. The renderer still has no global illumination, which is pencilled in for 2.0.

**Games made with it:** no well-known commercial releases yet. The official list includes **Station Iapetus**, a Dead Space-style horror game by the engine's author, **Fish Folly**, a Fall Guys-style 3D platformer, and open source multiplayer shooters such as **rustcycles** and **Breakfloor**[^41].

**Who it's for:** Rust developers who want a stable API and a Unity-like editor workflow, and who can live with a smaller community and ecosystem than Bevy's.

## O3DE: the heavyweight

The Open 3D Engine is the descendant of Amazon's Lumberyard, now run by the Open 3D Foundation under the Linux Foundation. It is modular, C++-based, has its own ray-tracing renderer (Atom), and positions itself as a "AAA-capable" engine for games, simulation and robotics.

Downloads grew by about 88% in 2025 compared to 2024[^17], and the project kept its two-releases-a-year rhythm:

- **O3DE 26.05** (May) added an experimental native **Open Particle System** with its own editor, AgX and Khronos PBR Neutral tonemapping, automatic LOD generation on FBX import, and made PhysX 5 the default, deprecating PhysX 4[^18].
- **O3DE 26.10** slipped from late October and is now scheduled for **11 November 2026**, after final testing found a defect in the Entity Inspector[^19].

**Games made with it:** very few so far. The showcase title is **MadWorld**, a third-person mobile shooter for iOS and Android by Carbonated, which the foundation announced in 2024 as the first mobile game powered by O3DE[^42]. Much of O3DE's use is outside games, in simulation and robotics.

**Who it's for:** teams that need a big, extensible C++ 3D engine and have the engineering capacity to handle its size and build complexity. Not a weekend-project engine.

## Stride: C# with a real 3D editor

Stride (formerly Xenko) is an MIT-licensed C# engine with a full editor, Game Studio, focused on realistic rendering and VR. It's supported by the .NET Foundation.

**Stride 4.4** is, in the project's words, one of its largest updates in years, with around 3,200 commits since the previous version[^20]. The focus was modernisation:

- Much more stable **Vulkan and Direct3D 12** backends, solid enough that the team plans to make one of them the editor default and may drop Direct3D 11 in the next major release.
- **Non-Windows platforms are no longer experimental.** Game Studio itself still targets Windows, but legacy code removal means it now runs on Linux through Proton/Wine.
- A complete rewrite of the **SDSL shader compiler** around a SPIR-V pipeline.
- A new `stride` CLI and `dotnet new` templates, so you can create and open a project entirely from the command line[^20].

**Games made with it:** the best known is **Distant Worlds 2**, a large real-time 4X space strategy game[^44]. Others include the 4X strategy game **Children of the Galaxy**, the steam-automation game **Pistonica**, and the VR streaming app **Virtual Desktop**[^43].

**Who it's for:** C# developers who want a Unity-like 3D workflow without a commercial license, and don't mind a smaller community.

## Defold: small, fast, and now properly 3D

Defold started as an internal engine at King and is now run by the Defold Foundation. It's free, Lua-scripted, and known for small builds and good mobile and web performance. One caveat for purists: its license is a modified Apache 2.0 that forbids selling the engine itself, so it is *source available* rather than open source in the OSI sense. Games you make with it are unrestricted and royalty-free[^21].

The first half of 2026 focused on gameplay, 3D, web and mobile[^22]. The 1.12 series added a `late_update()` lifecycle function, an engine throttling API to pause updates and rendering when idle, and camera helpers. **Defold 1.13** (June) brought morph targets (blend shapes) for glTF models, much larger Box2D scripting APIs, and made **Vulkan the default graphics API on Android**[^23].

**Games made with it:** Defold's games are less famous by name than by numbers. **Family Island** from Melsoft Games has more than 50 million installs on Android, and four of its web games on Poki, including **Monkey Mart**, drew more than 220 million plays between them in 2022[^45]. On PC and Switch there are **Void Scrappers** and **Fates of Ort**[^45], and in 2026 **MicroMacro: Downtown Detective** moved from mobile towards Steam on the same Defold codebase[^46].

**Who it's for:** mobile and web games where build size and startup time matter, and teams happy with Lua.

## MonoGame: XNA's heir gets a new foundation

MonoGame is a framework, not an engine: no editor, just a C# API in the style of Microsoft's old XNA, used by a long list of shipped indie games.

**MonoGame 3.8.5** (July) is a restructuring release[^24]. MonoGame now routes platform-specific work through a single native C/C++ library instead of a web of third-party C# wrappers, which should make bugs easier to fix. It adds a **Vulkan desktop target (DesktopVK)**, meant to eventually replace DesktopGL, and a **DirectX 12 target** that is compatible with the Microsoft GDK for Store and Game Pass publishing. The content pipeline gets a code-based Content Builder, and there are now ARM64 development and runtime packages for Windows, macOS and Linux[^24].

**Games made with it:** some of the best-loved indie games of the last decade: **Stardew Valley**, **Celeste** and **Streets of Rage 4**[^47]. Stardew Valley's creator, Eric Barone, has also given back, with a $125,000 donation to the project[^48].

**Who it's for:** C# programmers who want full control and prefer code over an editor.

## raylib: the simplest thing that works

raylib is a C library for games and graphics programming, with bindings for dozens of languages. **raylib 6.0** (April) is its biggest release yet, with over 2,000 commits and more than 210 new contributors[^25]. The headline is **rlsw**, a software renderer that lets raylib run purely on the CPU with no GPU at all. That opens it up to microcontrollers and GPU-less RISC-V boards. The release also brings a reorganised filesystem API, a reworked fullscreen and high-DPI system, and more than 70 new examples. Part of the work was funded by NLnet and the NGI Zero Commons Fund[^25].

**Games made with it:** raylib has no big commercial hit; it shows up mostly in game jams and small itch.io releases, where the "made with raylib" tag lists hundreds of games[^49]. That fits its purpose: it's the library people learn on and prototype with.

**Who it's for:** learning, game jams, tools, small games, and anything embedded.

## The id Tech family: old engines, living source ports

Long before Godot existed, id Software had a habit of releasing the source code of its engines once they were a few years old. Everything up to and including **id Tech 4** is free software: id Tech 0 to 3 under GPLv2 or later, and the later releases under GPLv3 or later. id Tech 5 (Rage), 6 (Doom 2016), 7 (Doom Eternal) and 8 have stayed proprietary[^26].

The last GPL release was the **Doom 3 BFG Edition** source in November 2012[^26]. The original Doom 3 code had come out a year earlier, on 22 November 2011, with one notable gap: the stencil shadow technique known as "Carmack's Reverse" was rewritten to avoid a patent dispute with Creative Labs[^27][^28]. Fourteen years on, nothing suggests id will open a newer engine, so "open source id Tech" means engines from 1993 to 2004, kept alive and modernised by community source ports.

| Engine | Original game | Main active source ports |
| --- | --- | --- |
| id Tech 1 | Doom (1993) | UZDoom, GZDoom, and many others |
| id Tech 2 | Quake (1996), Quake II (1997) | vkQuake, Ironwail, QuakeSpasm |
| id Tech 3 | Quake III Arena (1999) | ioquake3 |
| id Tech 4 | Doom 3 (2004) | dhewm3, RBDOOM-3-BFG |

### Doom: the GZDoom split

The biggest id Tech story of the past year is a fork. **GZDoom** has for years been the default engine for advanced Doom modding, and even for commercial games built on top of it. In late 2025 a conflict within its team led most of the developers to leave and continue as **UZDoom** ("Ultimate ZDoom")[^29][^30]. UZDoom picks up exactly where GZDoom left off: its first stable release, 4.14.3 on 28 November 2025, was a direct follow-up to GZDoom 4.14.2 with security fixes, support for cross-port standards such as MBF21 and UMAPINFO, and defaults closer to how the original Doom looked (texture filtering, for example, is now off by default)[^29][^30]. Work on UZDoom 5.0, which includes the features the team had been preparing for GZDoom 5.0, has been running through pre-release builds during 2026.

For anyone building a commercial game on the ZDoom family, this was an uncomfortable moment, but it looks like the community, and most of the people who knew the code, moved together.

### Quake: two ports trading features

Quake 1's engine is in rude health. **vkQuake**, a Vulkan port derived from QuakeSpasm, made SDL3 the default (with native Wayland on Linux), added Steam integration and support for the 2021 Nightdive re-release, and introduced order-independent transparency in 1.35.0 in July[^31]. Version 1.37.0 in October added gyro aiming, flick stick and rumble for controllers, live previews in the graphics menu and faster texture loading[^32]. **Ironwail**, another QuakeSpasm fork, focuses on raw performance by moving work from the CPU to the GPU, and picks up the 2021 re-release content automatically[^33]. Much of vkQuake's 2026 changelog is credited to Ironwail, so the two projects effectively develop together.

### Quake III: ioquake3

**ioquake3** has maintained the Quake III Arena engine for over twenty years and is still the base for most id Tech 3 projects, including free games like OpenArena[^28]. In 2025 it switched its build to CMake and deprecated the old Makefile, and it still runs its own game jams[^34].

### Doom 3: dhewm3 and RBDOOM-3-BFG

id Tech 4 has two main lines:

- **dhewm3** is the conservative one: it runs the original Doom 3 (not BFG) on modern systems with SDL. Version 1.5.5 came out in June 2026, and its mod SDK now also maintains the LibreCoop co-op mod[^35]. The game code of Quake 4, another id Tech 4 game, has also been adapted to run on it[^27].
- **RBDOOM-3-BFG** is the ambitious one. Building on the BFG Edition, it has added PBR rendering, baked global illumination and level editing with TrenchBroom, and in 1.5 replaced OpenGL with **Direct3D 12 and Vulkan** through NVIDIA's NVRHI, plus glTF model support[^36]. The latest release is 1.6.0, from May 2025[^37].

### Games made with them

These aren't just for replaying the old games. New games, some of them commercial, ship on these engines today. Note that the games below were built on the GPL source releases and their ports; commercial id Tech games of the era, such as Quake 4 or Prey, used separately licensed versions of the engines.

- **Doom engine (GZDoom family):** **Hands of Necromancy** (Frechou Games, 2022), a dark fantasy shooter[^51]; **Selaco** (Altered Orbit Studios, 2024), a F.E.A.R.-inspired sci-fi shooter on its own GZDoom fork[^52]; and the fantasy shooter **Hedon**[^50].
- **Quake engine:** **Wrath: Aeon of Ruin**, published by 3D Realms and fully released in 2024 on PC and consoles, runs on DarkPlaces, a modified Quake engine[^53]. The free arena shooter **Xonotic** runs on DarkPlaces too[^54].
- **Quake III engine (ioquake3):** the free multiplayer shooters **Urban Terror**, **OpenArena** and **Smokin' Guns**, and the Dragon Ball fighting game **ZEQ2-Lite**[^55].
- **Doom 3 engine:** **The Dark Mod**, a free Thief-style stealth game, became a complete standalone game on the GPL id Tech 4 code in 2013 and is still one of the few games built on it[^56].

### Should you build a new game on id Tech?

It's less crazy than it sounds. These engines are small, fast, run on almost anything, and have a proven "feel" for first-person shooters. There's a whole wave of retro-styled shooters, and a mature tool chain around them (TrenchBroom for maps, decades of community knowledge).

The catch is the **GPL**. The engine and any game code linked into it must be released under the GPL when you distribute the game. Your art, sound and levels can stay proprietary (that's exactly how id shipped its own source releases, without the game data), but your code can't. And these are engines with 1990s and early-2000s architecture: you get no general-purpose editor, no modern scene system, and a workflow that is closer to modding than to Godot or Unity.

**Who it's for:** retro shooter projects, modders, and anyone who wants to learn how classic engines were built by reading some of the most studied game code ever published.

## How they compare to Unity and Unreal

No look at open source engines is complete without the two engines most commercial games are actually made with. Neither Unity nor Unreal is open source, and both had an eventful couple of years.

**Unity** is still the most common choice for indie, mobile and mid-sized games, written in C#. The current release is Unity 6.4, from March 2026[^60]. Its pricing has calmed down since the 2023 Runtime Fee controversy: Unity cancelled the per-install fee in September 2024, went back to per-seat subscriptions, raised the free Personal tier's revenue and funding limit from $100,000 to $200,000, and made the splash screen optional for Unity 6 Personal games[^57]. Unity Pro now costs $2,310 per seat per year, after a 5% increase in January 2026[^58]. Personal can't publish to consoles; that needs a paid plan[^59]. The most recent licensing wrinkle came in June 2026, when Unity's updated terms said AI agents and MCP tools may only interact with the platform through frameworks Unity operates or designates, wording that has caused some confusion[^59].

**Unreal Engine** is the standard for high-end 3D. Unreal Engine 5.8, released on 17 June 2026, is planned as the last major 5.x release. It added Lumen Lite, a lighter global illumination mode about twice as fast as standard Lumen, experimental Mesh Terrain, and made MegaLights and the Iris networking system production-ready[^62][^64]. Licensing is royalty-based: free until a game's lifetime gross revenue passes $1 million, then 5%, with sales on the Epic Games Store royalty-free[^61]. The full engine source is available to licensees, but it is *source available*, not open source: you can read and modify it, not redistribute it freely.

The big Unreal story is **Unreal Engine 6**. Epic plans early access at the end of 2027, with the full release 12 to 18 months later. UE6 will merge Unreal and the Unreal Editor for Fortnite into one editor and introduce the Verse programming language[^62]. Blueprints and Actors will still be there in early UE6 so projects can carry over, but Epic says they will eventually be deprecated in favour of a new framework, with conversion tools promised[^63]. For Unreal users, that's a multi-year migration, a reminder that breaking changes aren't unique to pre-1.0 open source engines like Bevy.

### Side by side

| | Open source engines (Godot as the example) | Unity | Unreal Engine |
| --- | --- | --- | --- |
| **Cost** | Free, no royalties, no seats | Free under $200k revenue/funding; Pro $2,310 per seat per year | Free until $1M per game, then 5% royalty (Epic Games Store sales exempt) |
| **Source code** | Open; you can fork and ship your own version | Engine core is closed | Available to licensees, not redistributable |
| **Can terms change on you?** | No; the license you have is the license you keep | Yes, and they have | Yes, though Epic's terms have been stable |
| **High-end 3D** | Improving (Godot 4.7, O3DE), not at AAA level | Strong | Best in class (Nanite, Lumen, MegaLights) |
| **2D** | Godot is excellent; Defold, MonoGame, raylib are strong | Strong | Weak, not a focus |
| **Consoles** | Possible, usually via third-party porting, because console SDKs are under NDA and can't ship in open source code | Built in (paid plans) | Built in |
| **Ecosystem** | Smaller asset stores and plugin libraries | Huge Asset Store, large talent pool | Huge marketplace (Fab), large talent pool |
| **Scripting** | GDScript, C#, Rust, Lua, C++ | C# | C++, Blueprints; Verse coming in UE6 |

### Where open source wins

- **Cost and control.** No royalties, no per-seat fees, no revenue thresholds to track, and no reporting. For a small studio, the difference between Godot and a few Unity Pro seats is real money every year.
- **Nobody can change the deal.** Unity's 2023 Runtime Fee is the reason a lot of developers looked at Godot in the first place. An MIT or GPL license can't be revised retroactively.
- **You can fix or fork the engine.** If a bug blocks you, you can patch it yourself. If the project goes in a direction you don't like, you can fork it, which is exactly what happened with GZDoom and UZDoom.
- **Lightweight and fast to iterate,** especially for 2D, where Godot is arguably better than either commercial engine.

### Where Unity and Unreal still win

- **High-end 3D.** If you need photorealism, huge open worlds or film-quality lighting, Unreal is still far ahead. Nothing open source matches Nanite and Lumen.
- **Consoles.** Both commercial engines support PlayStation, Xbox and Switch out of the box (on the right plan). With open source engines you usually need a porting partner or your own console work.
- **Ecosystem and hiring.** Asset stores, plugins, tutorials, middleware integrations and the number of developers who already know the engine are all much larger.
- **Proven at scale.** Thousands of shipped games, including big-budget ones, and support contracts for studios that need them.

### The gap is narrowing

The clearest sign is Slay the Spire 2: a major commercial studio moved to Godot and sold 3 million copies in a week[^39]. For 2D and most small-to-mid-sized 3D games, an open source engine is now a genuine first choice, not a compromise. For AAA 3D and console-first projects, Unreal and Unity remain the safer bet.

## Trends worth noticing

**Maturity over novelty.** Godot's vision statement talks about features that work "reliably and completely". Fyrox's 1.0 is a polish release. Stride 4.4 and MonoGame 3.8.5 are both about rebuilding foundations. The big engines are less interested in feature checklists than in being dependable.

**Modern graphics APIs are now table stakes.** Vulkan became Defold's default on Android, MonoGame and Stride both shipped Vulkan and D3D12 backends, and Bevy adopted WESL and added mesh shader support. Even 20-year-old id Tech engines now run on Vulkan and D3D12 through their source ports. OpenGL is becoming the fallback, not the main path.

**Rust has two credible engines.** Bevy and Fyrox represent two philosophies (code-first ECS vs. editor-first, Unity-like), and both made real progress this year. Neither is yet a safe default for a commercial team, but the gap is closing.

**Review capacity is the real bottleneck.** Godot's contribution policy is the most visible example, but the underlying problem, more incoming code than maintainers can review, affects every project on this list. Expect other projects to follow with their own rules on AI-generated contributions.

**Governance matters as much as code.** Godot wrote down its vision; GZDoom showed what happens when maintainers and contributors stop agreeing on direction. The good news is that open source licensing made the UZDoom fork possible at all: the community didn't lose the engine, just the old name.

**Funding is still the weak spot.** Godot has a foundation and corporate members, O3DE has the Linux Foundation, and raylib got public grant money. Fyrox, by contrast, describes its team as small with little to no funding. Before committing a multi-year project to a smaller engine, look at who pays for it.

## Which one should you pick?

- **You want to make a game and don't know where to start:** Godot.
- **You want to write Rust:** Bevy if you like ECS and code-first; Fyrox if you want an editor and a stable API.
- **You want C# with an editor:** Godot (C#) or Stride.
- **You want C# without an editor:** MonoGame.
- **You're making a mobile or web game and care about size:** Defold.
- **You're building something large in C++, or a simulation:** O3DE.
- **You're learning, jamming, or targeting tiny hardware:** raylib.
- **You're making a retro FPS and are fine with the GPL:** an id Tech source port, such as UZDoom or a Quake engine.
- **You need AAA-level 3D or are console-first:** Unreal, or Unity, is still the safer choice.

## References

[^1]: [Godot 4.6 se lanza con Jolt Physics y un nuevo editor](https://app.cinevva.com/es/news/2026-02-27-godot-4-6-released), Cinevva: release date and overview of Godot 4.6.
[^2]: [Godot 4.6 Release](https://godotengine.org/releases/4.6/), Godot Engine; summarised in [Godot 4.6 released](https://www.linuxcompatible.org/story/released-11a), LinuxCompatible.
[^3]: [Godot 4.7 – Lights, Camera, Action!](https://godotengine.org/releases/4.7/), Godot Engine; overview in [The release of Godot 4.7 has taken place](https://wnhub.io/news/engines/item-51174), WN Hub.
[^4]: [Godot 4.7 adds AreaLight3D, Asset Store overhaul, GABE support, and HDR output](https://alternativeto.net/news/2026/6/godot-4-7-adds-arealight3d-asset-store-overhaul-gabe-support-and-hdr-output/), AlternativeTo.
[^5]: [godotengine/godot-builds releases](https://github.com/godotengine/godot-builds/releases), GitHub: Godot 4.8 beta 1 (15 October 2026) and beta 2 (29 October 2026).
[^6]: [Vision for the Engine](https://godotengine.org/article/godot-vision-statement-2026/), Godot Foundation, 25 June 2026.
[^7]: [Vision for the Godot Engine](https://godot.foundation/policies-and-procedures/project-vision-statement), Godot Foundation.
[^8]: [Changes to our Contribution Policies](https://godotengine.org/article/contribution-policy-2026/), Godot Foundation, 30 June 2026.
[^9]: [Godot Tightens Rules on AI-Contributed Code](https://letsdatascience.com/news/godot-tightens-contribution-policy-to-restrict-ai-code-e58bf90a), Let's Data Science; see also [Godot Engine to get stricter on AI contributed code](https://www.gamingonlinux.com/2026/07/godot-engine-to-get-stricter-on-ai-contributed-code/), GamingOnLinux.
[^10]: [Bevy README](https://docs.rs/crate/bevy/0.18.0/source/README.md), docs.rs: release cadence of roughly one breaking release every three months.
[^11]: [Bevy 0.18 brings atmospheric occlusion, PBR shading, Solari improvements & font variations](https://alternativeto.net/news/2026/1/bevy-0-18-brings-atmospheric-occlusion-pbr-shading-solari-improvements-and-font-variations), AlternativeTo.
[^12]: [Bevy 0.19 brings next-gen scenes, faster rendering, contact shadows & new feathers widgets](https://alternativeto.net/news/2026/6/bevy-0-19-brings-next-gen-scenes-faster-rendering-contact-shadows-and-new-feathers-widgets/), AlternativeTo.
[^13]: [Bevy 0.20](https://bevy.org/news/bevy-0-20/), Bevy Engine, 8 October 2026.
[^14]: [Bevy 0.20 Rust Powered Game Engine Released](https://gamefromscratch.com/bevy-0-20-rust-powered-game-engine-released/), GameFromScratch.
[^15]: [Fyrox Game Engine 1.0.0](https://fyrox.rs/blog/post/fyrox-game-engine-1-0-0/), Fyrox blog; see also [Fyrox 1.0 Release Candidate](https://fyrox.rs/blog/post/fyrox-game-engine-1-0-0-rc-1/).
[^16]: [Fyrox Game Engine Reaches Version 1.0.0](https://www.i-programmer.info/news/144-graphics-and-games/19015-fyrox-game-engine-reaches-version-100.html), I Programmer.
[^17]: [O3DE 2025 In Review and A Look Into 2026](https://o3de.org/o3de-2025-in-review-and-a-look-into-2026/), Open 3D Engine.
[^18]: [Open 3D Foundation releases O3DE 2026.5](https://www.cgchannel.com/?p=176448), CG Channel.
[^19]: [O3DE Release 26.10.0 (scheduled for November 11th, 2026)](https://github.com/o3de/sig-release/issues/372), o3de/sig-release on GitHub.
[^20]: [Stride 4.4.0 release notes](https://github.com/stride3d/stride/releases/tag/releases%2F4.4.0), stride3d/stride on GitHub.
[^21]: [Defold](https://en.wikipedia.org/wiki/Defold), Wikipedia: history and the Defold License.
[^22]: [Defold in the middle of 2026 – a summary](https://defold.com/2026/06/30/Defold-H1-2026/), Defold Foundation.
[^23]: [Defold Newsletter June 2026](https://gamedev.net/news/4248/), GameDev.net: Defold 1.13.0 highlights.
[^24]: [MonoGame 3.8.5 Released](https://gamefromscratch.com/monogame-3-8-5-released/), GameFromScratch; release date from [3.8.5 Release](https://gamedev.net/news/4495/), GameDev.net.
[^25]: [raylib 6.0 released!](https://raysan5.itch.io/raylib/devlog/1497342/raylib-60-released), raylib devlog on itch.io.
[^26]: [id Tech](https://en.wikipedia.org/wiki/Id_Tech), Wikipedia: source releases and licenses of each id Tech engine.
[^27]: [id Tech 4](https://en.wikipedia.org/wiki/Id_Tech_4), Wikipedia: the Doom 3 source release, its forks, and the Quake 4 port to dhewm3.
[^28]: [Doom 3 source code](https://doomwiki.org/wiki/Doom_3_source_code), Doom Wiki; and [Doom 3 is open sourced](https://www.omgubuntu.co.uk/2011/11/doom-3-is-open-sourced/amp), OMG! Ubuntu, on the missing "Carmack's Reverse" and earlier id releases such as Quake III.
[^29]: [UZDoom](https://www.doomwiki.org/wiki/UZDoom), Doom Wiki.
[^30]: [UZDoom is now available, opening up new chapter for Doom modding](https://www.gamepressure.com/newsroom/uzdoom-is-now-available-opening-up-new-chapter-for-doom-modding/z5890d), Gamepressure; see also [GZDoom successor project UZDoom gets a first preview release](https://gamingonlinux.com/2025/11/gzdoom-successor-project-uzdoom-gets-a-first-preview-release), GamingOnLinux.
[^31]: [vkQuake 1.35.0 brings SDL3, Steam integration, support for the 2021 re-release and more](https://gamingonlinux.com/2026/07/vkquake-1-35-0-brings-sdl3-steam-integration-support-for-the-2021-re-release-and-more), GamingOnLinux; and the [vkQuake 1.35.0 release notes](https://github.com/Novum/vkQuake/releases/tag/1.35.0).
[^32]: [vkQuake 1.37.0 release notes](https://github.com/Novum/vkQuake/releases/tag/1.37.0), Novum/vkQuake on GitHub.
[^33]: [Ironwail](https://modstalgia.com/engines/ironwail), Modstalgia.
[^34]: [Breaking Changes: ioquake3 Switching to CMake; Makefile Deprecated](https://ioquake3.org/ioquake3/breaking-changes-ioquake3-switching-to-cmake-makefile-deprecated/), ioquake3.
[^35]: [dhewm3](https://dhewm3.org), project homepage and news.
[^36]: [RBDOOM-3-BFG downloads](https://www.moddb.com/mods/rbdoom-3-bfg/downloads), ModDB: release notes for 1.3.0 to 1.5.1.
[^37]: [RBDOOM-3-BFG](https://doomwiki.org/wiki/RBDOOM-3-BFG), Doom Wiki.
[^38]: [Famous Godot Games](https://gamedesignskills.com/game-design/famous-godot-games/), Game Design Skills.
[^39]: [The Godot-powered Slay the Spire 2 has already hit over 3 million sales](https://gamingonlinux.com/2026/03/the-godot-powered-slay-the-spire-2-has-already-hit-over-3-million-sales), GamingOnLinux.
[^40]: [Tiny Glade](https://en.wikipedia.org/wiki/Tiny_Glade), Wikipedia.
[^41]: [Games built using Fyrox](https://fyrox.rs/games.html), Fyrox.
[^42]: [Open 3D Foundation Announces MadWorld, New Mobile Game Built on Open 3D Engine](https://www.linuxfoundation.org/press/open-3d-foundation-announces-madworld-mobile-game-built-on-o3de), Linux Foundation, March 2024.
[^43]: [Games and demos](https://doc.stride3d.net/latest/en/community-resources/games-and-demos.html), Stride documentation.
[^44]: [A Look Inside Distant Worlds 2: Developing with the Stride Engine](https://www.stride3d.net/blog/distant-worlds-2-development-with-stride/), Stride blog.
[^45]: [Is Defold Production Ready?](https://defold.com/2023/09/28/Is-Defold-Production-Ready/), Defold Foundation.
[^46]: [Defold News](https://gamedev.net/news/topic/defold/), GameDev.net: MicroMacro: Downtown Detective.
[^47]: [What is MonoGame](https://docs.monogame.net/articles/tutorials/building_2d_games/01_what_is_monogame/), MonoGame documentation.
[^48]: [Eric Barone makes a $125,000 donation to the MonoGame project](https://tildes.net/~comp/1rzk/eric_barone_makes_a_125_000_donation_to_the_monogame_project), Tildes.
[^49]: [Games made with raylib](https://itch.io/games/made-with-raylib), itch.io.
[^50]: [Hedon](https://doomwiki.org/wiki/Hedon), Doom Wiki.
[^51]: [Hands of Necromancy](https://en.wikipedia.org/wiki/Hands_of_Necromancy), Wikipedia.
[^52]: [Selaco](https://doomwiki.org/wiki/Selaco), Doom Wiki.
[^53]: [Wrath: Aeon of Ruin](https://en.wikipedia.org/wiki/Wrath:_Aeon_of_Ruin), Wikipedia.
[^54]: [Xonotic](https://en.wikipedia.org/wiki/Xonotic), Wikipedia.
[^55]: [ioquake3 engine](https://www.moddb.com/engines/ioquake3), ModDB.
[^56]: [The Dark Mod 2.0 Now A Complete Standalone Game](https://www.phoronix.com/news/MTQ4MjY), Phoronix; see also [The Dark Mod](https://en.wikipedia.org/wiki/The_Dark_Mod), Wikipedia.
[^57]: [A message to our community: Unity is canceling the Runtime Fee](https://unity.com/blog/unity-is-canceling-the-runtime-fee), Unity, 12 September 2024.
[^58]: [Unity announces its upcoming 2026 price changes](https://80.lv/articles/unity-announces-its-upcoming-2026-price-changes), 80 Level.
[^59]: [State of Unity Licensing in 2026](https://enginesdatabase.com/blog/state-of-unity-licensing-in-2026/), Engines Database.
[^60]: [Unity 6.4 and Unity Studio Released](https://www.pulse.bot/gaming/news/unity-64-and-unity-studio-released-0148627e-bf36-4856-8730-93f94b04bafc/), GameFromScratch via Pulse.
[^61]: [Unreal Engine licensing](https://www.unrealengine.com/en-US/license), Epic Games.
[^62]: [Epic Games has released Unreal Engine 5.8 and announced the release date for Unreal Engine 6](https://wnhub.io/news/engines/item-51157), WN Hub.
[^63]: [Unreal Deprecating Blueprints and Actors](https://gamefromscratch.com/unreal-deprecating-blueprints-and-actors/), GameFromScratch.
[^64]: [Unreal Engine 5.8 release notes](https://dev.epicgames.com/documentation/unreal-engine/unreal-engine-5-8-release-notes), Epic Games.
