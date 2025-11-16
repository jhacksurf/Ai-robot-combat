# AI Battle Bot – Updated Sensor & Network I/O Spec

This is the **latest version** of the design based on your last message.

---

## 1. MCU & High-Level Design

- MCU: **Raspberry Pi Pico 2**
- Storage: Internal flash (and later SD if needed).
- **No explicit history** in inputs.
- **Memory is entirely via scratch channels** (network-controlled state).
- Model is trained in simulation first, then refined with real fight logs.

---

## 2. Sensors (Per Timestep)

### 2.1 Horizontal ToF Distance Sensors (Walls / Bots / Compactor)

**Purpose:** See objects around the bot while on wheels **or** flipped.

Layout idea (exact count tweakable):

- **Front** (e.g. 3–5 sensors):
  - `tof_front_left`
  - `tof_front_center`
  - `tof_front_right`
  - (optionally `tof_front_mid_left`, `tof_front_mid_right`)

- **Sides** (e.g. 2 sensors):
  - `tof_side_left`
  - `tof_side_right`

- **Rear** (e.g. 2 sensors):
  - `tof_rear_left`
  - `tof_rear_right` (or `tof_rear_center`)

All values are **normalized** (0..1):
- 0 = very close
- 1 = far / max measurable distance

The exact number and names can be finalized when you know how many ToFs you physically mount, but the network expects:
- A block of “horizontal ToFs” that cover front, sides, rear.

---

### 2.2 Ground / Pit ToF Sensors

**Purpose:** Detect floor vs pit, both upright and upside-down.

Total: **6 ground/pit-oriented ToFs**:

- **4 downward / angled near the base** (for normal upright pit detection):
  - `tof_ground_fl_down`
  - `tof_ground_fr_down`
  - `tof_ground_rl_down`
  - `tof_ground_rr_down`

- **2 angled “upwards” / opposite direction** (useful when upside down or tilted):
  - `tof_ground_front_up`
  - `tof_ground_rear_up`

All normalized 0..1 as distances.

> The network will see all 6 values every tick.  
> IMU orientation lets it learn when the “down” ones are trustworthy and when the “up” ones become useful (e.g. upside down).

---

### 2.3 Mouse / Motion Sensor

**Purpose:** Estimate robot motion along floor, detect slip.

Per timestep, convert raw optical Δx, Δy to robot-frame velocities:

- `v_forward`   – forward/backward speed (normalized -1..1)
- `v_sideways`  – lateral slip speed (normalized -1..1)

No manual history; the network will use scratch memory to track anything longer-term it cares about.

---

### 2.4 IMU – Full Raw Inputs

You want **full raw IMU**, so we include all accel + gyro axes:

- Accelerometer:
  - `accel_x`
  - `accel_y`
  - `accel_z`

- Gyroscope:
  - `gyro_x`
  - `gyro_y`
  - `gyro_z`

From these, the network can infer:
- Upright vs upside-down vs on side (via gravity direction).
- Spin rate, impacts, etc.

> No separate orientation flags are strictly required now, because the model sees full accel/gyro and can decode orientation itself.

---

### 2.5 RC Inputs (Mode Signals for the Model)

Instead of feeding sticks, the model just gets **mode/state info** from RC / game state:

- `rc_test_check`  – “test/check” mode
- `rc_get_ready`   – “get ready” mode
- `rc_go`          – “fight/go” mode

You can feed these as **three separate inputs**, typically one-hot encoded:
- Example:
  - Test/check: 1,0,0
  - Get ready:  0,1,0
  - Go:         0,0,1

If you decide to add an “idle” or “disabled” state later, you can add a fourth mode input.

These tell the network *where in the match flow* you are (e.g. testing sensors vs actually fighting).

---

### 2.6 Scratch Memory (Network-Controlled State)

All “Pi memory” / internal state is via **scratch channels**.

- Inputs:
  - `scratch_in[0..N-1]`  (from previous timestep)
- Outputs:
  - `scratch_out[0..N-1]` (network’s new memory state)

On the firmware side:

```text
scratch_in(t) = scratch_out(t-1)
scratch_in(0) = all zeros at match start/reset
