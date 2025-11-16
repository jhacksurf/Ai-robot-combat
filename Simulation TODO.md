# AI Battle Bot – Simulation TODO / Setup Checklist

This is the checklist of things we still need to define/implement for the simulator.

---

## 1. Core Sim Loop & Timing

- **Time step `dt`**
  - Choose a fixed step, e.g. `dt = 0.02 s` (50 Hz) or `0.01 s` (100 Hz).
  - This drives:
    - Physics update
    - Sensors
    - Reward calculation
    - Policy calls

- **Main loop structure**
  - `reset(arena_config, round_idx)`:
    - Builds arena (size, pit, compactor).
    - Places bots in corners.
    - Zeros scratch memory for both bots.
  - `step(action_A, action_B)`:
    1. Apply actions (`v_linear`, `v_turn`) → forces/velocities.
    2. Integrate physics.
    3. Handle collisions & flipping.
    4. Update tilt.
    5. Update compactor walls.
    6. Compute sensor readings.
    7. Build observations.
    8. Compute rewards.
    9. Check termination (win/loss/draw).

---

## 2. Robot Dynamics & Control

- **Action → motion model**
  - Map `v_linear`, `v_turn` to:
    - Target forward speed & yaw rate.
  - Limit:
    - Max speed.
    - Max acceleration / deceleration.
    - Max turn rate.
  - Add simple friction so bots naturally slow if no command.

- **2-wheel tilt model (front/back rocking)**
  - Update `tilt_angle` based on:
    - Change in forward speed (accel).
    - Collisions (impulses at front/back).
  - Add damping so `tilt_angle` decays back toward 0 over time.

- **Flipping state**
  - Define discrete orientation state:
    - `UPRIGHT`, `ON_SIDE`, `UPSIDE_DOWN`.
  - Rules for when tilt and impacts flip the bot between these states.

---

## 3. Sensor Emulation Details

We’ve defined *what* sensors exist — we still need exact math:

- **Coordinate system**
  - Bot local frame:
    - +x forward, +y left, +z up (for IMU).
  - Arena frame:
    - +X right, +Y up (top-down).

- **Per-sensor transform**
  - For each ToF / ground sensor:
    - Mount position in bot frame `(x_s, y_s, z_s)`.
    - Mount direction (angles).
  - Use yaw + tilt (+ possible roll later) to get world ray.

- **Noise models**
  - ToF:
    - Add small Gaussian noise.
    - Occasional outlier / missed readings.
  - Mouse:
    - Scale & small noise on `v_forward`, `v_sideways`.
  - IMU:
    - Bias + noise on accel and gyro.

- **Normalisation ranges**
  - Decide exact min/max for:
    - Distances → map to 0..1.
    - Speeds → map to -1..1.
    - Accel/gyro → sane [-1, 1] scaling.
  - These must match what you’ll use on the real robot.

---

## 4. Opponent Bot Behaviour

Need simple “AI” for the **other** bots during training:

- **Basic behaviours (pick randomly per episode)**
  - Charger:
    - Always moves toward our bot.
  - Circle & strike:
    - Orbits near center, dashes in when close.
  - Random walker:
    - Moves semi-randomly but avoids walls.
  - Camper:
    - Hangs near pit or center then rams when we get close.

- **Same physics/sensors**
  - Opponent uses the same dynamics & collision rules.
  - Can be:
    - Scripted (no neural net), or
    - Another copy of the same policy if you go self-play later.

---

## 5. Arena Randomisation & Compactor Schedule

We already said “random size + same for 2 rounds” — now specify:

- **Size ranges**
  - `width ∈ [W_min, W_max]`
  - `height ∈ [H_min, H_max]`

- **Pit size & location**
  - Exact corner (e.g. top-right).
  - Pit rectangle size `[pit_w, pit_h]` or radius if circular.

- **Compactor timing**
  - `t_compactor_start`:
    - Random in range (e.g. 5–20 s after Go).
  - `compactor_speed`:
    - How fast walls move inward.
  - Minimum safe arena size so bots don’t get instantly crushed.

- **Randomised spawns**
  - Offsets:
    - `x_offset, y_offset` ranges per corner.
  - Yaw variation:
    - Around “toward center” direction.

---

## 6. RL Environment Wrapper

Turn the sim into something your training code can call easily:

- **API**
  - `env.reset(seed=None)`:
    - Randomises arena config.
    - Prepares Round 1 or the pair (Round 1 & 2).
    - Returns initial observation(s) for our bot.
  - `env.step(action)`:
    - Applies our bot’s action; opponent acts via its scripted policy.
    - Runs one `dt` step.
    - Returns:
      - `obs`, `reward`, `done`, `info`.

- **Two-round structure**
  - Option 1: treat each round as a separate episode.
  - Option 2: treat the **pair** (corner swap) as one extended episode with a reset-in-the-middle and carry some meta in `info`.

- **Seeding & reproducibility**
  - Use a random seed for:
    - Arena size.
    - Start positions.
    - Opponent behaviour choice.
    - Noise.
  - So you can reproduce runs when debugging.

---

## 7. Logging & Debug Tools

- **Episode logging**
  - For debugging:
    - Store history of:
      - Observations
      - Actions
      - Rewards
      - Positions/orientation
      - Collisions/flip events
  - This is separate from the future *real robot* logs.

- **Visualisation**
  - Simple 2D viewer:
    - Bots as shapes.
    - Walls, pit, compactor.
    - Rays for ToFs (optional).
  - Great for:
    - Checking collisions.
    - Verifying sensors.
    - Watching learned behaviour.

- **Diagnostics**
  - Track and print:
    - Average episode length.
    - Win/draw/loss rates.
    - Average reward per episode.

---

## 8. Training / Model Pipeline Hooks

- **Input ordering**
  - Final fixed list of input indices:
    - e.g. index 0–N for each sensor, RC mode, scratch.
  - This must match:
    - Sim → training → model export → Pico firmware.

- **Network config**
  - Define:
    - Input size (sensors + scratch_in).
    - Output size (2 + scratch_out).
    - Hidden layer sizes.
  - This will be used:
    - In training code.
    - In your C inference code.

- **Export format**
  - Decide now:
    - `model.bin` header layout.
    - Weight order.
    - Quantisation scheme (e.g. int8 with per-layer scale).

---

## 9. Practical “First Implementation” Steps

When you start coding, do it in layers:

1. **Bare sim core**
   - Arena with pit (no compactor yet).
   - 2D rigid body for one bot.
   - Horizontal ToFs.
2. **Second bot + collisions**
   - Add opponent bot.
   - Handle bot–bot and bot–wall collision.
3. **Tilt + ground sensors**
   - Implement `tilt_angle`.
   - Ground ToFs reacting to tilt and pit.
4. **Compactor & flipping**
   - Moving walls.
   - Flip logic on strong impacts.
5. **Rewards & RC phases**
   - Implement Test / Get Ready / Go.
   - Add movement + combat rewards.
6. **RL wrapper**
   - `reset/step`, observation packing, action unpacking.
7. **Visual debug**
   - Simple viewer to see what’s actually happening.

---
