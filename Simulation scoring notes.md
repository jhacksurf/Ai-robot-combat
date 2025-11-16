# AI Battle Bot – Scoring & Reward Design

These notes define how **scoring / rewards** work in the simulator, and how the **start sequence** using RC inputs ties into that.

---

## 1. Match Phases & RC Start Logic

We use the RC-mode inputs as a simple **state machine**:

- `rc_test_check` – pre-fight testing / sensor check.
- `rc_get_ready` – bots in starting corners, ready but not fighting.
- `rc_go` – fight is live: scoring, rewards, and win/lose conditions active.

### 1.1 Start Sequence

1. **Test / Check Phase**
   - `rc_test_check = 1`, others 0.
   - Used for bench tests and system checks.
   - No scoring; episode not yet “live”.

2. **Get Ready Phase**
   - `rc_get_ready = 1`, others 0.
   - Bots are placed in corners (with random offsets).
   - They should mostly hold position (or make only small movements).
   - No major scoring; optionally small penalty for big movement to encourage being ready.

3. **Go Phase**
   - `rc_go = 1`, others 0.
   - Match timer starts here (e.g. up to 3 minutes).
   - All scoring / rewards are active.
   - Compactor timing is referenced from the start of `Go`.

For RL, you can either:
- Include `Test/Check` + `Get Ready` in the episode with near-zero reward, or
- Consider the episode “starts” at `Go` while still feeding mode bits into the net.

---

## 2. Movement-Based Scoring (Controlled Movement)

We want to encourage **controlled movement**, especially forward motion, and not just spinning around.

Let (from sim/mouse):

- `v_forward` = forward speed (can be negative for backward).
- `v_turn` = turning speed / yaw rate.

### 2.1 Basic Movement Reward

- Reward any movement a little:
  - `R_move_any = k_any * |v_forward|`
- Reward forward movement more (2× like you suggested):
  - `R_move_fwd = k_fwd * max(v_forward, 0)`  
    with `k_fwd ≈ 2 * k_any`
- Reward controlled turning when moving:
  - `R_turn = k_turn * |v_turn| * step(|v_forward| > v_min)`

Total per-step movement reward:

```text
R_movement = k_any  * |v_forward|
           + k_fwd  * max(v_forward, 0)
           + k_turn * |v_turn| * step(|v_forward| > v_min)
