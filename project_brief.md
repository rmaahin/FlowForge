# Project Brief: OpenROAD PPA Predictor

**Status:** Currently on Phase 1 (environment setup)
**Last updated:** 2026-11-11

---

## Goal

Build a Python pipeline that runs the open-source OpenROAD-flow-scripts (ORFS) chip design flow in Docker across a sweep of clock periods and utilization settings, turns every run into a labeled dataset, and trains ML models to predict timing slack and power from the design settings.

## Target outcomes (resume bullets, hypotheses, not guaranteed)

- Built a Python pipeline that runs the OpenROAD flow (synthesis, place and route, timing and power analysis) in Docker across **110+ clock and utilization settings on 2 designs**
- Turned each tool run into a labeled dataset of timing slack, power, and area, and tracked failed runs, finding that **~10% failed routing once utilization passed 70%**
- Trained XGBoost models to predict timing slack and power from design settings, reaching **~0.88 R² on slack and ~8% error on power for unseen clock values** vs. **~0.76 R² and ~14%** for a linear baseline

> Final bullets will use the numbers I actually measure.

## Working style (mentor mode)

I write all the code myself. Claude provides file/function structure, pseudocode, edge cases, docs to look up, and code reviews (hints first). Full solutions only if I say "just show me."

---

## My setup

| Item | Value |
|---|---|
| Machine | Lenovo Legion: CPU `24 Cores`, RAM `32 GB` |
| OS | Windows `11 Home` + WSL2 (Ubuntu `24.04`) |
| Docker | Docker Desktop `4.99.0`, WSL integration on |
| ORFS image | `openroad/orfs:sha256:688272c2cca337c25e98437ae0e2dc919f1bb659d8e488427c52d97ae903e507` |
| Python | 3.__ (pandas, xgboost, scikit-learn, matplotlib, pyyaml) |
| Project path | `C:\Users\rmaah\Documents\Projects\FlowForge` |

## Daily startup / shutdown

### Start up
1. **Windows:** start Docker Desktop and wait until it shows "Engine running".
2. **Open Ubuntu:** Windows Terminal → ⌄ → Ubuntu-24.04.
3. **Check Docker is reachable:** `docker info > /dev/null && echo docker ok`
4. **(Python work only)** go to the repo and activate the venv:
   - `cd /mnt/c/Users/rmaah/Documents/Projects/FlowForge`
   - `source ~/.venvs/flowforge/bin/activate` → prompt shows `(flowforge)`
5. **(ORFS work)** start the container from the ORFS clone:
   - `cd ~/OpenROAD-flow-scripts`
   - `docker run --rm -it -u $(id -u):$(id -g) -v $(pwd)/flow:/OpenROAD-flow-scripts/flow openroad/orfs:latest`
6. **Inside the container:** set up tool paths, then go to the flow folder:
   - `source ./env.sh`
   - `cd flow`
7. **Verify:** `yosys -help` and `openroad -help` both print help.

### Shut down
1. Make sure no `make` run is in progress. If you Ctrl+C a run, clean that variant before reusing it (`make DESIGN_CONFIG=... FLOW_VARIANT=... clean_all`).
2. **Container:** `exit` (`--rm` deletes the container; outputs in `flow/` stay on disk).
3. **venv:** `deactivate`
4. **Ubuntu:** `exit`
5. **PowerShell (optional, frees RAM):** `wsl --shutdown`
6. **Docker Desktop:** right-click the whale icon in the system tray → Quit Docker Desktop.

---

## Designs & sweep plan

| | Design 1 | Design 2 |
|---|---|---|
| Name | `gcd` (nangate45) | `aes` or `ibex` (nangate45), TBD |
| Why | Small, fast, sanity check | Larger, more likely to hit routing congestion |
| Clock periods | ~11 values, `____` to `____` ns | ~11 values, `____` to `____` ns |
| Core utilization | 30, 40, 50, 60, 70, 75, 80, 85 % (TBD) | same |
| Planned runs | ≥55 | ≥55 |

**Target labels per run:** setup WNS, TNS, total power, design area, run status (ok / failed / timeout), failing stage, runtime.

**Features:** design, clock period, core utilization (+ maybe place density, derived features).

**Evaluation rule:** hold out by clock period (unseen clock values), never a random split. Metrics: R² for slack, MAPE for power.

---

## Phases

- [ ] **Phase 1: Environment.** WSL2 + Docker + ORFS image; run `gcd` by hand; find the metrics file, slack/power/area keys, where clock and utilization are set; force a failure and inspect the log.
- [ ] **Phase 2: Sweep runner.** `sweep.py` + `run_flow.py`: generate per-run SDC/config, launch the container, timeout, log status.
- [ ] **Phase 3: Extraction.** `extract.py` + `build_dataset.py`: parse metrics JSON + logs into one CSV, including failed runs.
- [ ] **Phase 4: Failure analysis.** `analyze_failures.py`: failure rate by utilization bin, plots.
- [ ] **Phase 5: Modeling.** `train.py`: linear baseline vs. XGBoost, GroupKFold by clock, R² / MAPE, feature importance.
- [ ] **Phase 6: Write-up.** README, figures, final resume bullets with real numbers.

## Planned repo layout

```
FlowForge/
├── configs/sweep.yaml
├── src/
│   ├── sweep.py
│   ├── run_flow.py
│   ├── extract.py
│   ├── build_dataset.py
│   ├── analyze_failures.py
│   └── train.py
├── data/          # raw run dirs + dataset CSV
├── results/       # plots, metrics, model outputs
└── notebooks/
```

---

## Decisions log

| Date | Decision | Why |
|---|---|---|
| 2026-11-11 | Start with `gcd` on nangate45 | Small and fast for validating the pipeline |
| | | |

## Results log

| Date | What | Result |
|---|---|---|
| 2026-11-11 | gcd nangate45, default settings, first run | real 0m24.950s, peak mem 561 MB (from docker stats) |

## Open questions

- Which second design: `aes` or `ibex`? (Depends on runtime per run.)
- Exact clock range per design (find the default period first, then sweep around it).

## Notes

- Everytime I run the ORFS container, I need to run "source ./env.sh" so the PATH is changed.