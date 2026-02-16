---
title: "Virtual Aquarium: System Design"
date: 2026-02-09
cover:
  image: /projects/virtual_aquarium.png
  hidden: true
---

The following system design was AI generated from my code and manually checked over by me.

# Aquarium System Design

## Tank

- **Dimensions**: 700×430×400 (width × height × depth), positioned at (75, 200)
- **Water level**: 10% from top, gravel layer 20px at bottom
- **Perspective**: Back wall scaled to 0.7× via `BACK_SCALE`, position/size interpolated by z-depth
- **Water rendering**: 20 translucent layers from back to front, opacity fading from 30 → 0
- **Render order**: Back wall → algae (non-front) → gravel → objects sorted by z-depth interleaved with water layers → front algae → front wall
- **Manages**: fish, microfauna, snails, food, algae, egg clumps

## Wall System

Five walls: `front`, `back`, `left`, `right`, `bottom`. Each has a 2D coordinate system:
- Front/Back: (worldX, worldY)
- Left/Right: (worldZ, worldY)
- Bottom: (worldX, worldZ)

Walls have adjacency relationships (for snail pathfinding) and opposites (front↔back, left↔right). Coordinate conversion handled by `wall-utils.ts`.

## Factor System

Unified pattern for state variables: `value`, `delta` (velocity), `ddelta` (acceleration). Each frame: `value += delta`, `delta += ddelta`.

- **Position**: Velocity decay 0.96 (fish) / 0.80 (snails). Wall avoidance within 50 units. Hard boundary clamping with bounce.
- **Hunger**: Range [0, 1]. Passive increase 0.0003/frame (fish) or 0.0001/frame (snails).
- **Fear**: Range [0, 1]. Passive decay 0.002/frame. Directional — tracks source of threat.
- **Initiative**: Range [0, 1]. Controls movement probability and force magnitude for fish.

---

## Fish (Ember Tetra)

### Perception
- **Visual field of view**: 45° cone, 300 unit range → detects fish, microfauna, food
- **Lateral line**: 50 unit radius sphere (non-visual) → detects nearby fish, splashes
- **Smell**: Inverse-square falloff for food detection
  - Strengths: sinking 0.10, floating/settled 0.0001
  - Direction vector decays 0.98/frame

### Behavioral State Machine
Priority order — higher states override lower:
1. **Fear** (`fear > 0.5`): Escape force away from threat, increased schooling attraction, exits feeding
2. **Hunger/Feeding**: Probabilistic entry based on hunger and food presence. Strike when target within 100 units. Eat when within 10 units (hunger −0.1).
3. **Normal**: Schooling via attraction/repulsion forces, random movement when alone

### Forces
- **Attraction**: Pull toward fish when distance > 150 (`distance × 0.005`, capped at 0.001)
- **Repulsion**: Push away when distance < 150 or lateral line triggered (`−0.2 / distance²`, capped at 0.1)
- **Random**: 25% chance when no fish in view
- **Fear escape**: `(fear − 0.7) × 0.05` when fear > 0.7

### Initiative System
- Builds proportional to net force magnitude (`forceMagnitude × 2.0`)
- Movement probability: `min(0.2, initiative)`
- Movement magnitude: `initiative × 10` with ±20% variance
- Decays to 90% after movement

### Rendering
- 5-directional sprites (front, front-left, left, back-left, back) with horizontal mirroring
- Vertical tilt from horizontal vs vertical velocity ratio
- Depth-scaled size: 1.0× at front → 0.7× at back

---

## Snails

### Lifecycle
States: `normal` → `egg-laying` → back to `normal`, or `normal` → `dying` → `shell` → `dead`

| Transition | Condition |
|------------|-----------|
| normal → egg-laying | hunger < 0.2, size ≥ 15, 0.03% chance/frame |
| egg-laying → normal | After 120 frames (2s). Spawns egg clump, hunger +0.60 |
| normal → dying → shell | hunger ≥ 0.7, 10% chance/frame. Dying phase skipped — goes straight to shell |
| shell → dead | Shell falls with size-scaled gravity, settles on gravel |
| dead → removed | Fades over 1800 frames (30s) |

### Navigation
Snails traverse tank walls with a waypoint-based pathfinding system:
- **Goal priority**: (1) nearest settled food, (2) best algae hotspot, (3) random
- **Same/adjacent wall**: Direct path
- **Opposite wall**: Routes through an intervening adjacent wall (picks shortest)
- **Wall transitions**: Triggered when within `size/2 + 5` units of connecting edge
- **Goal re-evaluation**: 0.3% chance/frame after cooldown (20 frames)

### Algae Sensing (Size-Scaled)
Smaller snails sense algae over shorter distances and prefer nearby targets:
```
sizeFraction = size / maxSize          (maxSize = 50)
sensingRange = 600 × (0.3 + 0.7 × sizeFraction)    → 180 to 600
distancePenalty = 50 × (1 + 2 × (1 − sizeFraction)) → 150 to 50
score = hotspot.strength / (distance + distancePenalty)
```
When close (< 40 units), refines to precise 8×8 grid search within a 40×40 region.

### Eating
- **Algae**: Checked every 5 frames within `size/2` radius. Two-pass system (cover then eat). Hunger decrease: `level × 0.005 × (20 / size)` — larger snails need more food.
- **Settled food**: Eaten within `size` radius. Hunger −0.05 × (20 / size).
- **Eating counter**: Increases on eating (algae: `level × 3`, food: +50), decays 2.0/frame. Reduces movement speed via sqrt curve down to 15%.

### Movement
- Acceleration: `0.042 × √size` — larger snails are faster
- Position decay: 0.80 (high friction on walls)

### Growth
- Chance/frame: `(1 − hunger) / 400`. Size +1, hunger +0.05. Max size: 50.

### Serialization Support
- Public getters expose private state for save: `getWall()`, `getLifeState()`, `getLifeStateCounter()`, `getEatingCounter()`, `getOpacity()`, `getShellSettled()`, `getCanSetGoals()`
- `loadSaveState(state)` restores lifecycle state, eating counter, opacity, shell status, goal-setting ability, and velocity in one call
- `getSpriteInfo()` exposes sprite index, rotation, and mirror state for thumbnail rendering

### Rendering
- 7 sprites: left, diagonal-front, front, back, top (back wall), bottom (front wall), empty shell
- Wall-specific sprite/rotation selection based on movement direction
- Shell/dead states always render as empty shell sprite, no rotation
- Opacity fades during dead phase

---

## Algae

### Growth
- Grows on `front`, `back`, `left`, `right` walls (not bottom)
- Grid: 4×4 pixel cells, levels 0–4
- Batch processing: samples 0.1% of active cells per frame, probabilities scaled 100×
  - New growth: 1% chance/frame on random empty cell
  - Spread: 0.1% chance to adjacent cells (including cross-wall at edges)
  - Level up: 0.01% chance (1 → 2 → 3 → 4)

### Hotspot System
Provides snails with aggregated algae density information:
- Updated every 50 frames (or on-demand via `requestHotspotUpdate`)
- Scans overlapping 40×40 regions (20px step) across all walls
- Returns top 10 hotspots sorted by strength (sum of algae levels in region)

### Rendering
- Color: green `rgb(0, 128, 0)`, alpha scales with level (0.1 per level)
- Left/right walls rendered as trapezoids for perspective
- Front wall rendered last for correct layering

---

## Egg Clumps

- Created by snails during egg-laying phase at the snail's position/wall
- Hatch after 600 frames (10s) → spawn 5–10 baby snails (size 3, hunger 0.5)
- Babies scattered within 20-unit radius, clamped to wall surface
- Visual: gray ellipse, pulses in final 20% of timer

---

## Food

- Dropped from above tank, falls with gravity 0.4
- Enters water → floats (stationary)
- 0.1% chance/frame to start sinking (acceleration 0.02, random drift)
- Settles on bottom → can be eaten by snails
- Fish detect via sight (10% if sinking, 0.1% if settled) or smell (inverse-square)

---

## Microfauna

- Size range 1.0–3.5, grow +0.1 at 1% chance/frame
- Reproduce: 0.4% base chance/frame, scales down with nearby count (max 20 within 100 units)
- Movement: random drift with slight upward bias
- Eaten by fish (detection scales with microfauna size: 0% at 1.5 → 100% at 3.5)

---

## Save/Load System

State is serialized into a compact JSON format, compressed, and steganographically encoded into a PNG thumbnail image. Loading decodes the image to restore the full aquarium state.

### Serialization (`serializeTank` / `deserializeTank`)
All entity types are serialized with short keys for compactness:
- **Fish**: position, velocity, size, hunger, initiative, fear
- **Snails**: position, velocity, size, wall, hunger, life state, life state counter, eating counter, opacity, shell settled, can set goals
- **Microfauna**: position, velocity, size
- **Food**: position, in-water, floating, settled flags
- **Egg clumps**: position, wall, hatch timer
- **Algae**: active cells as `[encodedPosition, level]` pairs

Deserialization clears existing tank state and reconstructs all entities. Fish are given a small random velocity if their saved velocity rounds to zero, to avoid division-by-zero in field-of-view calculations.

### Compression
Uses the browser's native `CompressionStream` / `DecompressionStream` API with `deflate-raw` encoding. Typical state compresses from ~800–1000 bytes JSON to ~400–600 bytes.

### LSB Steganography
Data is embedded in the 2 least significant bits of each RGB channel (alpha untouched). Each pixel stores 6 bits of data.
- **Header**: 2-byte magic (`0x41 0x51` = "AQ") + 4-byte little-endian data length
- **Capacity**: 200×123 image → ~18KB, far exceeding typical ~600-byte compressed state
- **Constraint**: Image must remain as lossless PNG — JPEG or chat-app compression destroys the encoded data

### Thumbnail Rendering
A custom renderer draws a stylized 200×123 thumbnail (not a canvas screenshot):
- **Margin**: 8px padding around the tank
- **Tank frame**: Front face border, back face rectangle (0.7× scale), four connecting perspective edges, water lines
- **Background**: Light blue-gray fill with semi-transparent water tint
- **Organisms**: Fish and snail sprites drawn at 2.5× inflated size using actual spritesheets and correct sprite selection (direction, wall, mirroring, rotation)
- **Omitted**: Food, microfauna, algae, egg clumps, gravel images — only a simple gravel strip is drawn

### UI
- **Save/Load buttons**: Rendered below the side pane in the p5.js canvas. Click-guarded by `isModalOpen()` to prevent double-opens.
- **Save modal**: Shows thumbnail preview at 2× display size, download button (timestamped filename), copy-to-clipboard button, compression stats
- **Load modal**: Drag-and-drop zone or file picker for PNG images. Shows image preview, entity count summary, and load button. Validates magic number before enabling load.
