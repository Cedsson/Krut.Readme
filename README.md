# Krut Engine

An Open Source MIT licensed FPS game engine built from scratch in C#/MonoGame, with [TrenchBroom](https://trenchbroom.github.io/) as the level editor.

![Gameplay](docs/screenshots/gameplay.png)

## Tech stack

| Part | Used for |
|---|---|
| [.NET 9](https://dotnet.microsoft.com/) / C# | Platform and language |
| [MonoGame](https://monogame.net/) (DesktopGL) | Window, graphics device, input, game loop |
| [Sledge.Formats.Map](https://github.com/LogicAndTrick/sledge-formats) | Reads TrenchBroom/Quake `.map` files (brushes, faces, entities) |
| [ImGui.NET](https://github.com/ImGuiNET/ImGui.NET) | Debug overlay, main menu, pause menu, options |
| [SharpGLTF](https://github.com/vpenades/SharpGLTF) | Skeletal animation for rigged glTF models (CPU skinning, no custom shader) |
| Native AOT | Publishes to a single small `.exe` with no `.NET` runtime required on the target machine |

## What it can do right now

**Levels**
- Reads TrenchBroom `.map` files: brushes become both rendered geometry and collision geometry
- Reads entities (class name + all properties + origin) — `info_player_start` places the player's spawn point/direction
- Textures load per material name from a `Textures/` folder (PNG); a missing texture falls back to a procedural checkered placeholder so surfaces are still distinguishable
- Point lights: `light` entities (Quake/TrenchBroom convention) are baked into per-vertex lighting with distance falloff — large faces are subdivided first so the gradient stays smooth instead of only showing up at the corners. A map's `_sun_ambient`/`_ambient` worldspawn property sets a global base brightness (capped so it never washes out the point lights). Maps with no lights at all fall back to flat directional shading instead of going black
- Levels can ship either as loose files or packed into a single `assets.pak` archive next to the executable — the engine picks whichever is present, so a dev build (loose files, easy to hot-swap) and a shipped build (one packed file) both work with the same code

**Player**
- Half-Life 1-style movement: ground/air acceleration and friction (not an instant velocity snap), sprint (Shift), and air-strafing that lets speed build up beyond the ground max — strafejumping/bunnyhopping works
- Collision against level geometry: axis-aligned box vs. every brush plane (same base technique as Quake), giving wall sliding and floor rest
- A free-flying spectator camera also exists in the engine for flying around levels during development

**Models**
- Rigged glTF models (`.glb`) play back skeletal animation via SharpGLTF.Runtime — the engine re-skins every vertex on the CPU each frame from the current animation pose, so no custom GPU shader or content-pipeline step is needed
- A procedurally generated test weapon model (two bones, one bend animation) ships with the project as a placeholder

**UI & settings**
- Main menu automatically lists every `.map` file found (loose or packed) — no code changes needed to add a new level
- Pause menu (Escape): Resume / Options / Main Menu / Quit
- Options has two tabs:
  - **Graphics**: window mode (windowed/fullscreen/borderless), resolution, V-Sync, MSAA (x0/x2/x4/x8)
  - **Controls**: click a binding and press a new key to rebind it (Esc cancels), plus a "Reset to defaults" button
- All settings are saved automatically to `Saves/settings.json` next to the executable and reloaded on next launch

**Performance**
- 300 FPS cap (V-Sync optional via the menu)
- Mipmapped, anisotropically filtered textures (no shimmering at a distance)
- Publishes with Native AOT: a single ~15-17 MB folder, no `.NET` install required to run

## Screenshots

| Main menu | Level list |
|---|---|
| ![Main menu](docs/screenshots/main-menu.png) | ![Level list](docs/screenshots/map-list.png) |

| Graphics options | Controls options |
|---|---|
| ![Graphics options](docs/screenshots/options-graphics.png) | ![Controls options](docs/screenshots/options-controls.png) |

## Not done yet

- Directional/"sun" lighting and shadows (only point lights + a flat ambient term right now, no shadow casting)
- More entity types (doors, triggers, pickups)
- A real first-person weapon view (the test model currently just stands in the level, not attached to the camera)
- Sound
- Encrypting `assets.pak` (currently a plain zip — openable with any archive tool)

## Build & run

```bash
dotnet run --project Krut.Game
```

Drop a `.map` file (TrenchBroom, "Standard" or "Valve" format) into `Krut.Game/Maps/` and it shows up in the main menu's level list automatically on the next launch.
