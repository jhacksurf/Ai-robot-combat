# AI Battle Bot – Sensing & Network I/O Spec

This doc captures the current design so we don’t lose it.

---

## 1. Overall Approach

- MCU: Raspberry Pi Pico 2
- No explicit “history” in the inputs (no prev/delta).
- **Many sensors + orientation + RC + internal memory**.
- Neural network has:
  - Normal outputs for motion.
  - Extra “scratch” outputs that are fed back as inputs next tick = **learned memory**.
- Model trained in simulation first, then refined with real fight logs.

---

## 2. Sensors & Signals (Per Timestep)

### 2.1 Time-of-Flight (ToF) Distance Sensors

**A. Horizontal ToFs (walls / bots / compactor)**  
Exact count & layout can be tuned, but conceptually:

- Front arc: e.g. 5 sensors
  - `tof_front_left`
  - `tof_front_mid_left`
  - `tof_front_center`
  - `tof_front_mid_right`
  - `tof_front_right`
- Side sensors: e.g. 2 sensors
  - `tof_side_left`
  - `tof_side_right`
- Rear sensors: e.g. 2 sensors
  - `tof_rear_left`
  - `tof_rear_center` / `tof_rear_right`

All values **normalized** 0..1 (0 = very close, 1 = far / max range).

**B. Ground / Pit ToFs (downward)**

- 4 downward/angled sensors near the front edge:
  - `tof_ground_fl`
  - `tof_ground_fr`
  - `tof_ground_rl`
  - `tof_ground_rr`
- Detect normal floor vs pit (sudden “no floor” distance).

Values normalized similarly; policy also sees orientation flags so it can learn when these are unreliable (e.g. upside down).

---

### 2.2 Mouse / Optical Motion Sensor

Converted into velocities in robot frame:

- `v_forward`   – forward/backwards speed estimate
- `v_sideways`  – lateral slip

Values normalized to a reasonable range (e.g. -1..1).

---

### 2.3 IMU (Accel + Gyro) & Orientation

Raw-ish IMU:

- `accel_x`
- `accel_y`
- `accel_z`
- `gyro_z` (yaw rate)

Plus **orientation flags** derived from accel (gravity direction):

- `is_upright`      (0 or 1)
- `is_on_side`      (0 or 1)
- `is_upside_down`  (0 or 1)

Only one of these should be 1 at a time under normal conditions.

---

### 2.4 RC Inputs

To mix human control and AI:

- `rc_throttle`   (stick forward/back, normalized -1..1)
- `rc_turn`       (stick left/right, -1..1)
- `rc_ai_enable`  (0 or 1)
- `rc_mode`       (0..1 or -1..1; extra 3-pos switch like safe/normal/aggressive)

---

### 2.5 Pit Memory (Optional Hard-Coded Memory)

The firmware can maintain an estimate of pit direction when seen:

- `pit_known`  (0 or 1, or a decaying confidence 0..1)
- `pit_vec_x`  (cos of pit direction from robot)
- `pit_vec_y`  (sin of pit direction from robot)

These are updated whenever pit sensors + orientation indicate the pit has been detected.

---

### 2.6 Scratch Memory (Learned State)

The network is given N “scratch” memory channels:

- **Inputs**: `scratch_in[0..N-1]`
- **Outputs**: `scratch_out[0..N-1]`

Firmware wiring:

```text
scratch_in(t) = scratch_out(t-1)
scratch_in(0) = all zeros at match start/reset
