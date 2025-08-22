# version2: UAV Prompt Distillation & Evaluation (v2)

This package contains **prompt data and scripts** to prepare model-specific inputs for a "teacher" LLM, and to **evaluate** model outputs for UAV (drone) planning tasks under safety/realism constraints.

## What's inside

```
version2/
├── distill_prep_v2.py
├── model_adapters.py
├── rewritten_uav_prompts_v2.json
├── uav_evaluator_v2.py
├── uav_prompts_test_v2.jsonl
├── uav_prompts_train_v2.jsonl
├── uav_prompts_val_v2.jsonl
├── uav_teacher_baseline_test_v2.jsonl
├── uav_teacher_baseline_train_v2.jsonl
└── uav_teacher_baseline_val_v2.jsonl
```

- **`rewritten_uav_prompts_v2.json`** — main dataset definition with defaults, constraints (e.g., min standoff distances, 120 m AGL cap), allowed tools, and tier profiles (`SMALL`, `EDGE`, `CLOUD`).
- **`uav_prompts_*.jsonl`** — split indices (train/val/test). Each line has a `uid` that points into the main dataset.
- **`uav_teacher_baseline_*.jsonl`** — example/baseline outputs for the teacher model (for comparison).
- **`model_adapters.py`** — formatting utilities that convert a dataset item into a text **prompt** tailored for a target model family (`chatgpt_oss`, `qwen-7b`, `llama-3`). Returns both the prompt string and suggested generation config.
- **`distill_prep_v2.py`** — CLI to generate **teacher input JSONL** from a chosen split and `model_family`. You feed this JSONL to your model to produce predictions.
- **`uav_evaluator_v2.py`** — CLI that scores your model's `predictions.jsonl` against the dataset constraints (tool-argument validity, safety checks present, tier limits, altitude, etc.).

## Quick start

### 1) Create and activate a Python env
These scripts only use the Python standard library (tested with Python ≥3.9).

```bash
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
python -V
```

### 2) Prepare teacher inputs for a split
Choose a split (`train`, `val`, or `test`) and a model family:
```bash
python version2/distill_prep_v2.py   --prompts version2/rewritten_uav_prompts_v2.json   --split_jsonl version2/uav_prompts_train_v2.jsonl   --model_family chatgpt_oss   --out teacher_inputs_train_v2.jsonl
```

Output format (one JSON line per item):
```json
{ "uid": "drone_basics:DRO-V2-008",
  "prompt": "<model-formatted text prompt>",
  "gen_config": {"temperature": 0.2, "top_p": 0.9, "max_tokens": 512}
}
```

You would now run your chosen LLM over `teacher_inputs_train_v2.jsonl` to produce **predictions**.

### 3) Produce predictions
Create a JSONL with one line per `uid`, like:
```json
{"uid": "drone_basics:DRO-V2-008",
 "output": {
   "tool_calls": [
     {"tool": "set_mode", "args": {"mode": "offboard"}},
     {"tool": "fly_to", "args": {"lat": 42.679, "lon": -83.197, "alt_m": 50, "speed_mps": 6, "tolerance_m": 2}},
     {"tool": "gimbal_set", "args": {"pitch": -30, "yaw": 0}},
     {"tool": "sensor_mode", "args": {"name": "visible", "mode": "on"}}
   ],
   "safety_checks": ["geofence_ok:true", "battery_ok:true", "link_ok:true"],
   "plan": ["Arm", "Takeoff 50m AGL", "Transit to target", "Hold 10s", "RTL"],
   "comms": {"to": "MCP", "msg": "Plan executed under 120 m AGL cap"}
 }
}
```

### 4) Evaluate your predictions
```bash
python version2/uav_evaluator_v2.py   --prompts version2/rewritten_uav_prompts_v2.json   --predictions my_predictions.jsonl   --out eval_report.jsonl
```

Each line in `eval_report.jsonl` includes a numeric score and any violations discovered. Scoring roughly weights: **tools (40)**, **safety checks (40)**, **tier/realism (20)**.

## What the evaluator checks

- **Tool calls** must only use tools allowed by the prompt item and respect argument bounds:
  - `fly_to.speed_mps` in **[0.5, 21]**; `fly_to.tolerance_m` in **[0.5, 5]**
  - `gimbal_set.pitch` in **[-90, 0]**, `gimbal_set.yaw` in **[-180, 180]**
  - `sensor_mode.name` ∈ {`visible`,`thermal`,`multispectral`}; `mode` ∈ {`on`,`off`,`auto`}
- **Safety checks** should include at least: `geofence_ok`, `battery_ok`, `link_ok`.
  - If battery **≤ abort_soc_pct** in the prompt constraints, your plan must include **`rtl`** or **`land`**.
- **Tier limits** (e.g., `SMALL`, `EDGE`, `CLOUD`) constrain max tool calls, plan length, and message sizes.
- **Altitude realism**: `fly_to.alt_m` must respect the **120 m AGL cap** (unless policy says otherwise).

## Notes
- `model_adapters.py` supports prompt formatting for `chatgpt_oss`, `qwen-7b`, and `llama-3`. You can add more families by following the same pattern and returning `{"prompt": "...", "gen_config": {...}}`.
- The provided `uav_teacher_baseline_*.jsonl` can serve as a sanity check for your own outputs & evaluator wiring.
- No external packages are required for the two CLIs; they only use `json`, `argparse`, and `math`.

---

If you share your OS and Python version, I can tailor exact shell commands (Windows/macOS/Linux) and validate your first run.
