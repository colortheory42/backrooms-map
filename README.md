# The Backrooms - Modular Engine

A from-scratch 3D engine with destructible walls, pixel debris physics, geometry-aware spatial audio, and live microphone processing.

## File Structure

```
├── config.py          # All configuration constants
├── debris.py          # Debris physics system
├── procedural.py      # Zone generation (5 types)
├── textures.py        # Procedural texture generation
├── audio.py           # Procedural sound generation + mic processor
├── raycasting.py      # Ray-triangle intersection + audio raycasting
├── save_system.py     # JSON save/load system
├── engine.py          # Main 3D engine
└── main.py            # Game loop and UI
```

## Dependencies

```bash
pip install pygame numpy sounddevice
```

`sounddevice` is optional — mic processing is silently disabled if not installed.

## Usage

```bash
python main.py
```

## Controls

### Movement
- **WASD** - Move
- **SHIFT** - Run
- **C** - Crouch (toggle)
- **SPACE** - Jump

### Camera
- **M** - Toggle mouse look
- **JL** - Keyboard turn
- **Mouse** - Look around (when mouse look enabled)

### Actions
- **LEFT CLICK** or **E** - Destroy wall/pillar (aim crosshair)

### System
- **F5** - Quick save (slot 1)
- **F9** - Quick load (slot 1)
- **R** - Toggle performance mode (0.5x/1.0x render scale)
- **H** - Toggle help overlay
- **ESC** - Pause / release mouse

---

## Module Responsibilities

### `config.py` - Configuration Hub
All tunable constants in one place: display, colors, physics, camera, world generation, effects, audio.

**When to edit**: Tuning gameplay feel, adjusting visual style, changing world density.

### `debris.py` - Debris Physics
Pixel-sized debris particles with velocity, gravity, bounce, settling detection, and age-based cleanup.

**When to edit**: Changing debris behavior, particle count, physics constants.

### `procedural.py` - Zone System
Five zone types (`normal`, `dense`, `sparse`, `maze`, `open`) with different pillar density, wall chance, ceiling height variance, and color tints. All deterministic from world seed.

**When to edit**: Adding new zone types, adjusting zone distribution.

### `textures.py` - Texture Generation
Procedurally generates all textures at runtime via NumPy — carpet, ceiling tiles, walls, pillars.

**When to edit**: Changing visual appearance, adding texture variations.

### `audio.py` - Sound Synthesis + Mic Processing
All game sounds generated from waveform synthesis (no audio files). Also contains `MicProcessor` — a live duplex sounddevice stream that captures mic input, applies geometry-derived room acoustics, and plays it back through your speakers in real time.

Acoustic treatment applied to mic audio:
- Comb-filter reverb with 4 delay lines, wet mix driven by room tightness
- Stereo panning biased by nearest left/right wall distances
- Occlusion attenuation

**When to edit**: Tuning sound design, adding new sound effects, adjusting reverb parameters.

### `raycasting.py` - Raycasting
Two separate raycasting systems:

**Visual raycasting** — Möller–Trumbore ray-triangle intersection for detecting which wall or pillar the player is looking at.

**Audio raycasting** — 2D horizontal ray marching through the wall grid:
- `cast_audio_ray` — single ray, returns distance to first wall hit
- `sample_room_acoustics` — fires 16 rays in a full ring, returns an `AcousticSample`
- `occlusion_between` — marches from player to emitter, counts wall crossings, returns attenuation factor
- `AcousticSample` — snapshot of nearest left/right reflectors, openness, reverb mix, and geometry-shaped stereo pan

**When to edit**: Adding new raycasting features, tuning audio ray count or step size.

### `save_system.py` - Persistence
JSON save/load with 5 slots. Stores player position, rotation, destroyed walls, playtime, and world seed.

**When to edit**: Adding new saveable data, changing save format.

### `engine.py` - Core Engine
- **Initialization**: Texture generation, state setup
- **Render scaling**: Smooth transition between 0.5x and 1.0x
- **Zone management**: Procedural zone caching
- **Visual raycasting**: Wall/pillar targeting from screen center
- **Destruction**: Wall/pillar removal + debris spawning (up to 1200 particles)
- **Audio system**: Geometry-aware directional sound — ambient emitters are placed in the world, occluded through walls, and panned using `AcousticSample`
- **Visual effects**: Fog, zone tinting, surface noise, ambient occlusion, light flickering
- **Collision**: AABB collision against walls and pillars, doorway/hallway passthrough, destroyed wall awareness
- **Player update**: Movement, jumping, crouching, head bob, camera shake
- **Camera**: Smoothing, world-to-camera transform, perspective projection, near-plane clipping
- **Rendering**: Polygon depth sorting, frustum culling, debris rendering
- **World generation**: Procedural walls with doorways/hallways, pre-damaged walls, rubble piles

**When to edit**: Core gameplay changes, rendering optimizations.

### `main.py` - Game Loop
Three game states (`MENU`, `PLAYING`, `PAUSED`) with clean transitions. Handles pygame init, engine creation, event loop, input, UI, save/load triggers, ambient hum, and mic processor lifecycle.

**When to edit**: UI changes, adding new input controls, new game states.

---

## Features

### Destructible Walls & Pillars
- Visual raycasting detects targeted wall or pillar
- Breaks into 250–1200 pixel debris particles
- Debris has realistic physics (gravity, bounce, settling)
- Distance-based pixel size (closer = bigger)
- Age-based and distance-based cleanup, hard cap at 12,000 particles
- Pre-damaged walls spawn at world generation with rubble piles

### Geometry-Aware Spatial Audio
- 16 audio rays cast outward every frame to build an acoustic snapshot
- Ambient sounds (footsteps, buzz) placed as virtual world emitters
- Line-of-sight occlusion — sounds behind walls are attenuated per wall crossed
- Stereo panning shaped by real reflector distances on each side
- Room reverb mix driven by average ray distance (tight room = wetter)

### Microphone Processing
- Live duplex stream: mic in → room acoustics → speaker out
- Same acoustic model as game audio — reverb and panning update as you move
- Comb-filter reverb with persistent ring buffer (no cutoff between frames)
- Requires `sounddevice`; degrades gracefully if not installed

### Procedural World
- Infinite grid-based world, deterministic from seed
- 5 zone types with varying density and color tints
- Procedural doorways and hallways
- Pre-damaged and destroyed walls baked at generation time

### Physics & Movement
- Jump with gravity, crouch with smooth height transition
- Head bob synced to footstep sounds
- Camera shake for atmosphere
- Solid collision against walls, wall thickness, doorway gaps, and pillars

### Visual Effects
- Zone-based color tinting
- Surface noise for texture variation
- Ambient occlusion at wall tops and bottoms
- Optional fog (toggle in `config.py`)
- Light flickering
- Smooth camera interpolation with separate movement/rotation smoothing

### Save System
- JSON-based, 5 save slots
- Stores position, rotation, destroyed walls, playtime, world seed

---

## Technical Details

- **Coordinate system**: Y-up, right-handed
- **Rendering**: Custom perspective projection, no external 3D library
- **Physics**: Frame-rate independent (delta time)
- **Audio**: Procedurally synthesized at runtime, no audio files
- **Textures**: Generated via NumPy, no image files
- **Collision**: AABB per wall segment + circle-vs-AABB per pillar, 3×3 cell neighbourhood only

## Performance Notes

- Render scale mode (R key) renders at 0.5x then upscales (2–4x FPS boost)
- Collision checks only the immediate 3×3 grid of cells — O(1) regardless of world size
- Audio rays are 2D grid marches, not triangle tests — very cheap
- Debris culled beyond 900 units, hard cap at 12,000 particles
- Debris auto-cleanup after 8–18 seconds, or 2–6 seconds after settling
- Frustum culling and distance culling on all geometry

## Extending the Engine

### Adding New Zone Types
Edit `procedural.py`, add to `ZONE_TYPES`:
```python
'new_zone': {
    'pillar_density': 0.5,
    'wall_chance': 0.3,
    'ceiling_height_var': 10,
    'color_tint': (1.0, 1.0, 1.0)
}
```

### Adding New Sounds
Add a generator function in `audio.py`, initialize in `main.py`, pass through `sound_effects` dict.

### Changing Physics
Tune constants in `config.py`:
```python
JUMP_STRENGTH = 150  # Higher = jump higher
GRAVITY = 300        # Higher = fall faster
WALK_SPEED = 50      # Higher = move faster
```

### Tuning Audio Raycasting
In `raycasting.py`:
```python
AUDIO_RAY_COUNT = 16    # More rays = more accurate acoustics, more CPU
AUDIO_RAY_STEP  = 20.0  # Smaller = more precise wall detection
AUDIO_RAY_MAX   = 2000.0
```

### Tuning Mic Reverb
In `audio.py`:
```python
REVERB_DELAYS = [0.013, 0.027, 0.043, 0.067]  # seconds
REVERB_GAINS  = [0.50,  0.35,  0.22,  0.14]   # per delay line
```

---

## Credits

Built from scratch — custom 3D math, procedural generation, physics simulation, waveform audio synthesis, and geometry-aware acoustic modelling. No external 3D libraries, no texture files, no audio files.
