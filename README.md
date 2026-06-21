# Financial Reality Engine (FRE)

[![Tests](https://github.com/gharteykeziah-hub/financial-reality-engine/actions/workflows/tests.yml/badge.svg)](https://github.com/gharteykeziah-hub/financial-reality-engine/actions/workflows/tests.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue.svg)](pyproject.toml)

A production-grade financial decision system that models how time allocation drives income, risk, and financial stability for variable-income workers.

Built for gig workers, shift employees, freelancers, and hourly workers whose income is not fixed — but derived from time worked.

---

## Screenshots

| Dashboard | Schedule (Week View) |
|---|---|
| ![Dashboard](screenshots/dashboard.png) | ![Schedule](screenshots/schedule_week.png) |

| Income Analytics | Monte Carlo Forecast |
|---|---|
| ![Analytics](screenshots/analytics_income.png) | ![Forecast](screenshots/forecasting.png) |

The dashboard surfaces balance, risk score, and weekly flow at a glance. The schedule view is the actual income source — every Work block on the calendar is what `financial_state.py` reads to compute everything else. The analytics tab and Monte Carlo forecast below pull directly from those scheduled hours, not from manually entered numbers.

---

## Why This Project Exists

Traditional budgeting tools assume income is constant.

That assumption breaks for millions of people.

FRE inverts the model:

> Income is not an input. It is computed from time.

```
Income = Σ (shift hours × hourly rate)
```

This enables the system to simulate financial outcomes from schedule changes in real time.

---

## What Makes FRE Different

FRE is not a budgeting app.

It is a financial simulation engine.

It allows users to:

- Simulate income directly from scheduled work shifts
- Measure the financial impact of adding/removing shifts instantly
- Rank jobs by effective hourly value ($/hr efficiency)
- Compute deterministic financial risk (0–100 scoring model)
- Run Monte Carlo simulations (500+ futures) for financial stability
- Project income and balance from real time allocation

---

## Core Insight

Most financial tools assume:

> Income → Budgeting → Outcome

FRE reverses this:

> Time → Income → Financial Stability

This shift enables questions like:

- What happens if I drop this shift?
- Which job actually pays the most per hour worked?
- What is my probability of running out of money in 12 weeks?
- How does my schedule affect long-term financial stability?

---

## System Architecture

```
Schedule Layer
    ↓
Time Engine (conflicts, free blocks, shift logic)
    ↓
Schedule Analytics (pure functions)
    ├── Job Efficiency Ranking
    └── Shift Optimizer (0/1 knapsack DP)
    ↓
Financial State Engine (single source of truth)
    ↓
Decision Systems
    ├── Insight Engine (interpretation layer)
    ├── Scenario Engine (what-if comparison)
    └── Monte Carlo Engine (500+ simulations, NumPy-vectorized)
    ↓
┌─────────────────────────┬─────────────────────────┐
│  UI Layer               │  API Layer (optional)    │
│  Tkinter desktop app    │  FastAPI service (api.py) │
└─────────────────────────┴─────────────────────────┘
    ↓
Export Layer (PDF + CSV)
```

This is the same UI / business-logic / data-layer split enforced throughout the codebase: the Tkinter desktop app and the FastAPI service (see [Web Service](#web-service-optional) below) are two different front ends sitting on top of the *same* unmodified engine layer — neither one contains any financial logic itself.

---

## Core Engineering Design

### 1. Financial State Engine (Single Source of Truth)

All financial metrics are computed in one place:

- income
- expenses
- net flow
- savings rate
- risk score (0–100)
- health score (0–100)

No other module recalculates financial logic.

---

### 2. Pure Analytics Layer

All schedule analytics are pure functions:

- no database access
- no UI dependency
- fully unit testable

---

### 3. Decision Engine

Computes real-time impact of schedule changes:

- income loss / gain
- savings rate change
- risk score delta
- natural language recommendations

---

### 4. Monte Carlo Simulation (500+ Runs)

Simulates financial futures using probabilistic life events.

**Methodology.** Each simulated week, 10 independent background events can fire — extra shifts, tip windfalls, sick days, car repairs, and similar real-life swings — each with its own probability and dollar range (e.g. "Car repair: 6% chance, −$80 to −$400/week"). A run's ending balance is the starting balance plus N weeks of base net flow plus whichever events happened to fire along the way. Doing this 500 times produces 500 independently plausible futures; sorting their ending balances gives the percentile spread (p25/median/p75), the best/worst case, and the fraction that ended in deficit — that fraction is reported directly as `deficit_probability`.

**Performance.** The simulation core is vectorized with NumPy: all `n` runs and all `weeks` of event draws are generated in one batched array operation instead of `n × weeks × 10` individual Python-level dice rolls. Measured on this machine (`scripts/benchmark_monte_carlo.py`, best-of-5):

| Scenario | Pure Python (original) | NumPy vectorized | Speedup |
|---|---|---|---|
| 500 runs / 52 weeks | 27.7 ms | 5.5 ms | **5.0×** |
| 500 runs / 12 weeks | 6.5 ms | 1.7 ms | 3.8× |
| 5,000 runs / 52 weeks | 282.2 ms | 44.8 ms | **6.3×** |

Run it yourself: `python3 scripts/benchmark_monte_carlo.py`

Outputs:

- probability of deficit
- median / worst / best case outcomes
- balance distribution
- financial stability risk

---

### 5. Schedule-Driven Income Model

Every shift is a financial event:

- start time
- end time
- hourly rate
- date

Income is derived entirely from schedule structure, not user input.

---

### 6. Shift Optimizer (constrained selection, not sorting)

`job_efficiency_report()` ranks jobs by effective $/hr — that's just sorting, and sorting doesn't answer the actual question a worker with limited hours has: *"I only have 25 hours free this week — which combination of available shifts earns the most?"*

Greedy "take the highest-rate shift first" is not guaranteed optimal once an hour budget limits which shifts can coexist — a worse-paying shift that leaves room for two more can beat a better-paying one that eats the whole budget. `optimizer.py` solves this exactly as a 0/1 knapsack problem via dynamic programming (hours discretized to quarter-hour units, O(n × capacity) time/space):

```python
from optimizer import ShiftCandidate, optimize_shift_selection

candidates = [
    ShiftCandidate("x", "Job X", hours=9, hourly_rate=20),  # $180 — highest rate
    ShiftCandidate("y", "Job Y", hours=5, hourly_rate=19),  # $95
    ShiftCandidate("z", "Job Z", hours=5, hourly_rate=19),  # $95
]
result = optimize_shift_selection(candidates, max_hours=10)
# greedy-by-rate would grab X alone ($180, 1h wasted)
# optimizer correctly picks Y + Z instead: $190, full 10h used
```

Covered by 10 dedicated tests in `test_fre.py`, including a regression test that asserts the optimizer beats the naive greedy-by-rate selection on a constructed counterexample.

---

## Key Features

- Real-time financial projections from schedule changes
- Shift impact analysis (what-if decision modeling)
- Job efficiency ranking ($/hour comparison across roles)
- Constrained shift optimization (0/1 knapsack DP — not greedy sorting)
- Deterministic financial risk scoring system
- NumPy-vectorized Monte Carlo forecasting engine (500+ runs)
- SQLite persistence with schema migration + deduplication
- Optional FastAPI service exposing the same engine over HTTP
- Fully offline desktop application by default
- Continuous integration — full test suite runs on every push

---

## Engineering Principles

- Single source of truth for financial state
- Pure functions for all analytics
- Schedule-driven income model (not manual input)
- Dependency injection (testable architecture)
- Deterministic outputs over heuristics
- UI fully separated from business logic

---

## Testing

- 166 automated pytest tests across 19 test classes
- No GUI required for testing
- Runs automatically on every push via GitHub Actions (see badge at the top of this README)
- Covers:
  - financial calculations
  - schedule system
  - simulation engine
  - shift optimizer (including a knapsack-vs-greedy regression test)
  - edge cases (overnight shifts, zero income, conflicts)
  - Monte Carlo stability
  - database integrity

```bash
python3 -m pytest test_fre.py -v
```

---

## Project Structure

```
├── Core
│   ├── financial_state.py      # All financial calculations — single source of truth
│   ├── model.py                # Job + Expense with frequency-aware weekly conversion
│   ├── database.py             # SQLite persistence, migration, backup, dedup
│   ├── utils.py                # canon_name() — shared name normalisation
│   └── config.py               # All constants and thresholds
│
├── Engines
│   ├── schedule_analytics.py   # Pure analytics: income by job, shift impact, efficiency
│   ├── optimizer.py            # 0/1 knapsack shift-selection optimizer
│   ├── schedule_service.py     # Schedule → financial sync (testable, no GUI)
│   ├── time_engine.py          # Free-block analysis, conflict detection, opportunity cost
│   ├── insight_engine.py       # Score interpretation and insight generation
│   ├── scenario_engine.py      # Side-by-side scenario projection
│   └── simulation.py           # What-If simulator + NumPy-vectorized 500-run Monte Carlo
│
├── Schedule
│   ├── schedule_event.py       # ScheduleEvent dataclass + time helpers
│   ├── schedule_core.py        # Schedule backend — DB ops, week navigation
│   ├── date_parser.py          # Natural language schedule import parser
│   └── shift_parser.py         # Shift input parsing
│
├── Pages
│   ├── app.py                  # App shell, DI container, navigation, exports
│   ├── page_dashboard.py       # Hero balance, stats, insights
│   ├── page_schedule.py        # Weekly calendar, conflict detection, free time
│   ├── page_analytics.py       # 5-tab analytics with decision engine output
│   ├── page_forecast.py        # Projection, scenario comparison, simulation
│   ├── page_goals.py           # Goal tracking, weeks-to-goal, emergency fund
│   └── page_settings.py        # App configuration
│
├── UI
│   ├── theme.py                # Dark/light palettes, ThemeManager, font constants
│   ├── widgets.py              # ScrollFrame, TabBar, card, kv_row, labeled_entry
│   └── charts.py               # matplotlib chart types embedded in tkinter
│
├── API (optional)
│   ├── api.py                  # FastAPI service — same engine, HTTP transport
│   └── static/index.html       # Thin frontend (vanilla JS, no build step)
│
├── .github/workflows/tests.yml # CI — runs the full suite on every push
├── scripts/benchmark_monte_carlo.py  # Reproducible before/after vectorization benchmark
└── test_fre.py                 # 166 pytest tests — no GUI instantiation required
```

---

## Tech Stack

- Python 3.10+
- Tkinter (GUI)
- SQLite (persistence)
- NumPy (vectorized Monte Carlo simulation)
- matplotlib (visualization)
- reportlab (PDF export)
- pytest (testing)
- FastAPI + Pydantic (optional HTTP API layer)

No ORM. No frontend build step. No UI toolkit beyond what ships with Python — the FastAPI layer is opt-in and adds zero dependencies to the desktop app.

---

## Installation

```bash
git clone https://github.com/gharteykeziah-hub/financial-reality-engine.git
cd financial-reality-engine
pip install -r requirements.txt
python3 main.py
```

`finance.db` is created on first launch and listed in `.gitignore`.

> **macOS:** if the window appears blank on first launch, run `brew install python-tk`.

---

## Web Service (Optional)

The Tkinter app is the primary interface, but the entire engine layer (financial state, schedule analytics, the optimizer, Monte Carlo) is GUI-agnostic — `api.py` exposes it over HTTP with zero changes to any engine module. Same source of truth, two front ends.

```bash
pip install -r requirements-api.txt
uvicorn api:app --reload
```

Then open `http://127.0.0.1:8000` for the built-in thin frontend (vanilla JS, no build step — run the financial snapshot, the shift optimizer, and a Monte Carlo simulation right in the browser), or `http://127.0.0.1:8000/docs` for interactive Swagger docs generated automatically from the type hints in `api.py`.

**Deploying it:** `render.yaml` and `Procfile` are included for Render (or any Procfile-compatible host like Railway or Heroku). Connect the repo on the host's dashboard and it builds from `requirements-api.txt` — no code changes needed. (Deployment itself requires an account on whichever host you pick; the config files here just remove all the setup work.)

---

## Project Philosophy

This system prioritizes:

- correctness over features
- architecture over UI polish
- deterministic computation over estimation
- simulation over static budgeting
- modeling real-world complexity instead of simplifying it

---

## Future Improvements

- PostgreSQL migration for multi-device sync (no business-logic changes required — `database.py` is the only file that would change)
- Predictive income modeling (time-series forecasting on the `history` table)
- Recurring shift engine
- Risk-minimizing mode for the optimizer (currently maximizes income; minimizing risk score under the same hour constraint is the natural dual problem)
- Richer web frontend (React) built against the existing FastAPI endpoints — the API and its schemas already exist, this is a pure frontend project
- External calendar integration (Google Calendar / .ics)

---

## Author

Aba

Software engineer focused on building systems that model real-world complexity through clean architecture, simulation, and deterministic design.

---

> "The goal is not to track money. The goal is to understand how time creates money."
