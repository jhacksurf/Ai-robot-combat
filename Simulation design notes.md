# AI Battle Bot – Simulation Design Notes

These notes describe how the **training simulator** should work so it matches the robot + sensors + game rules you’ve described.

---

## 1. Arena Configuration & Randomisation

### 1.1 Base Arena

- 2D top-down rectangle.
- Units: meters (or arbitrary units, but be consistent).

### 1.2 Arena Size Randomisation

For each **arena config**:

- Sample width/height from a small range, e.g.:
  - `width  ∈ [W_min, W_max]`
  - `height ∈ [H_min, H_max]`
- Keep the same size for both rounds of a given config.

This gives:
- Slightly different match sizes each time.
- But both bots see the exact same arena for their 2 swapped rounds.

### 1.3 Pit

- Pit = a **hole in one corner** of the arena.
- Represent as a rectangular/square region cut out at one corner, e.g.:
  - Bottom-left corner: `x ∈ [0, pit_size_x]`, `y ∈ [0, pit_size_y]`
- If a bot’s position (or front reference point) enters pit region:
  - That bot is considered **fallen in** (round ends).

### 1.4 Compactor Walls

- One or more **moving walls** that shrink arena *towards the pit corner*.
- For simplicity:
  - Start walls at normal arena bounds.
  - After some time `t_compactor_start`, walls start moving inward at a slow rate toward the pit corner.
- Compactor is just extra walls for collision and sensor hits.

---

## 2. Match Structure & Bot Placement

### 2.1 Rounds Per Arena Config

For each **arena configuration**:

- There are **2 rounds**:
  1. Bot A starts in Corner 1, Bot B in Corner 2.
  2. Positions are **swapped**:
     - Bot B in Corner 1, Bot A in Corner 2.
- Same:
  - Arena size
  - Pit position
  - Compactor behavior

### 2.2 Randomised Start Positions

For each round:

- Each bot is placed in a corner **with random offsets**.

Example (Corner near pit vs opposite corner):

- **Corner near top-right**:
  - Base corner position: `(x_corner, y_corner)` 
  - Add random:
    - `x_offset ∈ [-offset_max, offset_max]`
    - `y_offset ∈ [-offset_max, offset_max]`
- **Random yaw**:
  - `yaw ∈ [yaw_center - yaw_spread, yaw_center + yaw_spread]`
  - E.g. facing roughly towards center, but with some random angle.

This gives slightly different starts each time while preserving “corner spawn” behavior.

---

## 3. Robot Model

### 3.1 Shape & Geometry

- Bot shape: simplified **3/4 moon** (three-quarter circle or rounded wedge).
  - Approximate as:
    - A circle with radius `R` with a flat or cut-out rear section.
  - Used for:
    - Collision with walls
    - Collision with other bots
    - Sensor mounting positions.

### 3.2 2D State

Per bot, main state:

- Position: `(x, y)` on arena floor.
- Orientation: `yaw` (heading in radians).
- Velocity: `v_forward` (forward in body frame).
- Angular velocity: `omega` (yaw rate).

### 3.3 Tilt Model (2-Wheel Bot Pitching)

Because the first design is **2-wheel drive**, the bot will pitch/tilt forward/back a bit when:

- Accelerating / braking.
- Colliding.
- Being pushed.

Represent tilt as:

- `tilt_angle` (pitch in radians/degrees, positive = nose-up or nose-down).

Simple model ideas:

- `tilt_angle` responds to:
  - `dv_forward/dt` (acceleration).
  - Collisions (impulses).
- It relaxes back toward 0 over time:
  - `tilt_angle += (some physics response)`
  - `tilt_angle *= decay_factor` each step (e.g. 0.9–0.99).

**Important:**  
Floor / pit sensors readings depend on `tilt_angle` (see below).

---

## 4. Sensors in the Sim

The simulator must generate all inputs the NN expects, at each timestep.

### 4.1 Horizontal ToFs (Front, Sides, Rear)

For each ToF sensor:

- Known **mount position** on the bot (relative to center).
- Known **direction vector** (depends on yaw and tilt for angled ones).

Simulation:

1. Cast a ray from sensor position in its facing direction.
2. Intersect with:
   - Arena walls
   - Compactor walls
   - Other robots (3/4 moon shapes)
3. Distance = min intersection distance, clamped to `[0, max_range]`.
4. Normalize to `[0, 1]`.

> You’ll be able to tweak sensor locations in code to test different layouts.

### 4.2 Ground / Pit ToFs (Tilt-Dependent)

We have:

- 4 downward/angled sensors near base:
  - `tof_ground_fl_down`
  - `tof_ground_fr_down`
  - `tof_ground_rl_down`
  - `tof_ground_rr_down`
- 2 front/rear “up” sensors:
  - `tof_ground_front_up`
  - `tof_ground_rear_up`

Simulation:

- Each sensor has a **local tilt angle** (e.g. mostly downwards).
- Combine with `tilt_angle`:
  - When the bot pitches forwards, front ground sensors see the floor closer.
  - When it pitches backwards, rear sensors change distance.
- For pit detection:
  - If ground ray falls inside pit hole region, distance is “large” (no floor).
- Normalise readings.

> This way, floor sensors “see” tilt (front/back rocking) and pits correctly.

### 4.3 Mouse / Motion Sensor

Sim uses bot’s true motion relative to the floor:

- Compute body-frame velocities `v_forward`, `v_sideways` from `vx, vy, yaw`.
- Add noise.
- Convert to normalized values (-1..1).

This approximates what an optical mouse would see on the ground.

### 4.4 IMU – Full Raw

Simulate:

- **Accel**:
  - Include gravity in robot body frame:
    - Based on `tilt_angle` (pitch) and any roll (if you add it later).
    - Add linear acceleration from motion.
- **Gyro**:
  - Angular velocity around x, y, z axes.
  - For yaw rotation, `gyro_z` is main component.
  - `gyro_x`, `gyro_y` can reflect pitching/rolling from collisions.

Add noise and small bias if needed.

---

## 5. Collisions & Flipping Rules

### 5.1 Bot–Wall Collisions

- Use circle or 3/4-moon approximation to detect:
  - Overlap with walls or compactor.
- Response:
  - Push bot out of wall.
  - Adjust velocities (bounce / friction).
- These collisions can also alter `tilt_angle` slightly (e.g. rear hit → nose dips).

### 5.2 Bot–Bot Collisions

- Detect collision when their shapes overlap:
  - Simplest: treat both as circles with radius `R` for first version.
- Compute relative velocity along the collision normal.
- Both bots get:
  - New velocities (push apart).
  - Changes to `tilt_angle` based on hit direction:
    - Front-on collision → nose dip, etc.

### 5.3 Flipping Logic (Chance of Flip)

On impacts:

- Identify which bot is “lower” at the front:
  - Use `front_tilt_value` derived from `tilt_angle`:
    - E.g. more nose-down = “lower front tilt number”.
- Compute impact speed:
  - Example: magnitude of relative velocity along collision normal.

Logic:

- If collision is:
  - sufficiently strong (impact_speed > threshold)
  - and one bot has significantly “lower front tilt” than the other
- Then:
  - The **lower-front bot** has a **small chance** to flip the other bot:

Example pseudo:

```pseudo
if impact_speed > impact_threshold and
   front_tilt_A << front_tilt_B then
    flip_prob = base_flip_prob * (impact_speed / max_impact_speed)
    with probability flip_prob:
        flip bot B (set its tilt/orientation to upside-down or side)
