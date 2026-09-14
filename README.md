# Krut Engine

An FPS game engine built from scratch in C#/MonoGame, with [TrenchBroom](https://trenchbroom.github.io/) as the level editor. Solo project, early development.

![Gameplay](docs/screenshots/gameplay.png)

## Tech stack

| Part | Used for |
|---|---|
| [.NET 9](https://dotnet.microsoft.com/) / C# | Platform and language |
| [MonoGame](https://monogame.net/) (DesktopGL) | Window, graphics device, input, game loop |
| [Sledge.Formats.Map](https://github.com/LogicAndTrick/sledge-formats) | Reads TrenchBroom/Quake `.map` files (brushes, faces, entities) |
| [Sledge.Formats.Bsp](https://github.com/LogicAndTrick/sledge-formats) | Reads compiled Quake/GoldSrc `.bsp` files (geometry, embedded textures, baked lightmaps) |
| [ImGui.NET](https://github.com/ImGuiNET/ImGui.NET) | Debug overlay, main menu, pause menu, options |
| [SharpGLTF](https://github.com/vpenades/SharpGLTF) | Skeletal animation for rigged glTF models (CPU skinning, no custom shader) |
| Native AOT | Publishes to a single small `.exe` with no `.NET` runtime required on the target machine |

## What it can do right now

**Levels**
- Reads both TrenchBroom `.map` source files and compiled Quake/GoldSrc `.bsp` files — the same level can be iterated on in TrenchBroom or dropped in as a finished `.bsp`, both end up as the same neutral brush/entity data internally
- Reads entities (class name + all properties + origin) — `info_player_start` places the player's spawn point/direction. Brushes belonging to `trigger_*` entities are recognized as non-solid logic volumes and excluded from collision/rendering, instead of being treated as ordinary solid geometry
- Textures load per material name from a `Textures/` folder (PNG) first; `.bsp` files fall back to their own embedded textures (miptex + palette) if no loose override exists; a still-missing texture falls back to a procedural checkered placeholder so surfaces stay distinguishable
- Lighting: a `.bsp`'s own baked lightmaps are read and used directly. For `.map` sources without baked lighting, `light` entities (Quake/TrenchBroom convention) are baked into per-vertex lighting with distance falloff instead — large faces are subdivided first so the gradient stays smooth instead of only showing up at the corners. A map's `_sun_ambient`/`_ambient` worldspawn property sets a global base brightness (capped so it never washes out the point lights). Maps with no lighting data at all fall back to flat directional shading instead of going black
- Levels can ship either as loose files or packed into a single `assets.pak` archive next to the executable — the engine picks whichever is present, so a dev build (loose files, easy to hot-swap) and a shipped build (one packed file) both work with the same code

**Player**
- Half-Life 1-style movement: ground/air acceleration and friction (not an instant velocity snap), sprint, crouch, and air-strafing that lets speed build up beyond the ground max — strafejumping/bunnyhopping works
- Crouch and sprint each have their own "hold" or "toggle" behavior, switchable per-binding in the options menu; crouching lowers the collision box, eye height, and move speed, and won't stand back up if there isn't clear headroom
- Automatic step-climbing: small ledges/stairs (up to a configurable step height) are climbed without jumping, with the camera smoothly catching up to the new height instead of snapping — walking down stairs is smoothed the same way
- Collision is built from the exact same triangulated geometry the renderer draws (not simplified brush planes), so what's solid always matches what's visible, including for non-convex or oddly-shaped brushes
- A free-flying spectator camera also exists in the engine for flying around levels during development

**Models & weapons**
- Rigged glTF models (`.glb`) play back skeletal animation via SharpGLTF.Runtime — the engine re-skins every vertex on the CPU each frame from the current animation pose, so no custom GPU shader or content-pipeline step is needed
- Three real first-person weapons (pistol, rifle, shotgun) — full arms + gun, switchable with `1`/`2`/`3`, reload on `R`, with idle/walk/run picked automatically from the player's own speed and sprint state
- The weapon is rendered with its own, narrower field of view than the world, so long guns don't look stretched at the world's wide FOV — a standard trick borrowed from classic FPS engines
- Animation changes (drawing a weapon, switching between idle/walk/run, reloading) crossfade instead of snapping, blending the actual skinned vertex positions between the old and new pose over a short window

**UI & settings**
- Main menu automatically lists every level file found (`.map` or `.bsp`, loose or packed) — no code changes needed to add a new one
- A minimal Steam-style FPS counter (just the number, no window chrome) sits in the corner during gameplay, updated once a second so it's actually readable
- A small crosshair dot sits in the center of the screen during gameplay
- Pause menu (Escape): Resume / Options / Main Menu / Quit
- Options has two tabs:
  - **Graphics**: window mode (windowed/fullscreen/borderless), resolution, V-Sync, MSAA (x0/x2/x4/x8)
  - **Controls**: click a binding and press a new key to rebind it (Esc cancels), a hold/toggle checkbox next to Sprint and Crouch, and a "Reset to defaults" button that covers all of it
- All settings are saved automatically to `Saves/settings.json` next to the executable and reloaded on next launch
- `-map <name>` command-line argument skips straight to a level, for launching from TrenchBroom's own "Launch Engine" tool during map testing

**Performance**
- 300 FPS cap (V-Sync optional via the menu)
- Mipmapped, anisotropically filtered textures (no shimmering at a distance)
- Publishes with Native AOT: a single ~28 MB folder (`Game.exe`), no `.NET` install required to run
- The weapon arms/gun rigs are CPU-skinned every frame; that's cheap enough in an optimized build but can get slow in an unoptimized Debug build — run with `-c Release` (see below) if framerate looks off during development

## Screenshots

| Pistol | Rifle | Shotgun |
|---|---|---|
| ![Pistol](docs/screenshots/weapon-pistol.png) | ![Rifle](docs/screenshots/weapon-rifle.png) | ![Shotgun](docs/screenshots/weapon-shotgun.png) |

| Main menu | Level list |
|---|---|
| ![Main menu](docs/screenshots/main-menu.png) | ![Level list](docs/screenshots/map-list.png) |

| Graphics options | Controls options |
|---|---|
| ![Graphics options](docs/screenshots/options-graphics.png) | ![Controls options](docs/screenshots/options-controls.png) |

## Not done yet

- Directional/"sun" lighting and shadows for `.map` sources without baked lightmaps (only point lights + a flat ambient term, no shadow casting)
- Trigger volumes are recognized and excluded from collision, but nothing reacts to them yet — no scripting/logic layer (doors, buttons, level triggers, pickups)
- Actually firing the weapons — no shooting logic, ammo tracking, aim-down-sights, muzzle flash, or hit detection yet, just the viewmodel and its movement animations
- Sound
- Encrypting `assets.pak` (currently a plain zip — openable with any archive tool)

## Build & run

```bash
dotnet run --project Krut.Game -c Release
```

Drop a `.map` file (TrenchBroom, "Standard" or "Valve" format) or a compiled `.bsp` into `Krut.Game/Maps/` and it shows up in the main menu's level list automatically on the next launch.
