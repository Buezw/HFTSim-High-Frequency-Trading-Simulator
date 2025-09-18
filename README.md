# HFTSim-High-Frequency-Trading-Simulator

**FPGA-inspired Verilog (sim), C++ low-latency matching engine, Python strategy + viz**
*Historical tick/order-book replay with documented execution assumptions*

---

## 1) What this project is

A **high-frequency trading (HFT) style** simulator that proves you understand:

* **Market microstructure**: order book, bid/ask, fills, cancels
* **Low-latency execution**: C++ matching engine with μs-level telemetry
* **Hardware thinking**: Verilog modules (simulated) for ns-pipeline parsing & pre-trade checks
* **Research→Execution handoff**: Python strategy layer; optional model trained in Python, run in C++

> We do **not** connect to live exchanges. We **replay historical tick/LOB data** as the counterparty market (“insert your strategy back into the past”) and compute fills at the book level under clear rules.

---

## 2) Repo layout

```
HybridHFT/
├─ data/                    # historical ticks/order book snapshots (CSV or binary)
├─ verilog/                 # Verilog modules: market-data parser + pre-trade risk (sim/Verilator)
├─ engine_cpp/              # C++ order book & matching engine (+ telemetry)
├─ py_strategy/             # Python strategies, PnL eval, charts, (optional) model training/export
├─ bridges/                 # pybind11 wrapper or ZeroMQ bridge (choose one)
├─ configs/                 # YAML/JSON configs for instruments, fills, fees
├─ docs/                    # design notes, latency budget, assumptions, diagrams
└─ README.md                # this file
```

---

## 3) System diagram (data flow)

```
        Historical Market Data (tick / L2-L3)
                      │
          (ns, simulated pipeline)
                      ▼
        [Verilog Parser + Pre-Trade Risk]
                      │  (Verilator eval or sim feed)
          (μs, low-latency matching)
                      ▼
         [C++ Order Book & Matching Engine]
                      │
           orders                 exec reports
         (bridge IPC)  ◀────────▶ (bridge IPC)
                      │
                 [Python Strategy]
         (research, viz, model training)
```

---

## 4) Quick start (minimum viable run)

### 4.1 Requirements

* **C++17/20**, CMake ≥ 3.20
* **Python 3.10+**, `numpy`, `pandas`, `matplotlib` (or `plotly`), `pyyaml`
* **pybind11** (if using in-process binding) or **ZeroMQ** (if using IPC)
* **Verilator** *(optional but recommended)* for integrating Verilog sim with C++

  * Alternative: run Verilog in a simulator (ModelSim/Vivado) and feed files/pipe

### 4.2 Build (choose ONE bridge)

**A) pybind11 (simplest, same process)**

```bash
# From project root
cmake -B build -D BRIDGE=PYBIND11 -D ENABLE_VERILATOR=ON
cmake --build build --config Release
# Produces: engine_cpp/libhft_engine.so (or .dll) and py_strategy/hft_engine*.so (Python module)
```

**B) ZeroMQ (more “prod-like”, separate processes)**

```bash
cmake -B build -D BRIDGE=ZEROMQ -D ENABLE_VERILATOR=ON
cmake --build build --config Release
# Produces: engine_cpp/hft_engine_server and py_strategy client scripts use ZMQ sockets
```

### 4.3 Run a demo

```bash
# 1) Prepare a dataset (see §6): e.g., data/binance_btcusdt_topbook_2023-06-01.csv
# 2) Configure assumptions
cp configs/example.yaml configs/run.yaml
# 3) Launch engine (and verilog sim if separate)
./build/engine_cpp/hft_engine_server --config configs/run.yaml --data data/...
# 4) Run a Python strategy
python py_strategy/run_mm_strategy.py --config configs/run.yaml
# 5) View charts & metrics in py_strategy/out/
```

---

## 5) Features checklist

* **Tick-level replay** (ns/us timestamps) with throughput stats
* **Verilog modules (simulated)**

  * Market-data “packet” parser (pipeline: ≥1 event/clk)
  * **Pre-trade risk** (size caps, price bands, rate limits)
  * Integrate via **Verilator** or simulator file/pipe
* **C++ matching engine**

  * Price-indexed order book (FIFO per level)
  * **Active (cross) and passive (posted) fills**
  * Deterministic backtest; μs-level telemetry (p50/p95/p99)
* **Python strategy**

  * Market-making (inventory control, adaptive spread)
  * Mean-reversion (EMA/microprice deviation)
  * PnL, fill ratio, drawdown, latency histograms
* **Research→Execution handoff (optional)**

  * Train lightweight model in Python (e.g., logistic/XGBoost)
  * Export weights/ONNX; run inference in C++

---

## 6) Data format & ingestion

### 6.1 Unified CSV (example)

```
ts_ns,event_type,side,price,qty,best_bid,best_ask,best_bid_qty,best_ask_qty
1685596800000000000,BOOK_UPDATE,B,27000.00,1.2,26999.90,27000.10,5.1,4.7
1685596800001000000,TRADE,S,27000.10,0.3,26999.90,27000.10,5.1,4.4
...
```

* `event_type ∈ {BOOK_UPDATE, TRADE}`
* If you have full L2/L3, pre-aggregate to top-of-book or feed depth deltas into the engine’s book updater.
* **Timestamps** are **ns** or **μs** integers. Prices **tick-aligned** (e.g., 0.01 USD).

### 6.2 Pre-run dataset probe

Run `py_strategy/tools/probe_data.py` to print:

* Total events, events/sec (avg/peak), spread distribution, trade size histogram

---

## 7) Execution assumptions (documented)

Backtests become credible when assumptions are explicit:

### 7.1 Fill model (choose & configure in `configs/run.yaml`)

* **Top-of-Book Fill (default, simple)**

  * Passive quotes fill **when** market **touches/crosses** your price; filled **up to** counter-flow size observed in history (pro-rata or capped).
* **Queue model (advanced, optional)**

  * Maintain a notional queue position per level; decrement with observed opposite-side traded volume to decide your turn.

### 7.2 Order semantics

* **Active orders** (marketable): fill at resting levels, may sweep multiple levels.
* **Passive orders**: rest in book; subject to min quote life `min_quote_us` before cancel.
* **Cancels/replace**: throttled by `cancel_cooldown_us`.

### 7.3 Fees, slippage, anomalies

* Fees: maker/taker in bps (configurable).
* Slippage: can add per-trade slippage bps for conservatism.
* Anomalies: specify policies for missing ticks, time regressions, negative sizes (drop/repair).

> All assumptions live in `configs/run.yaml` and are printed at start of run for reproducibility.

---

## 8) Verilog layer (simulated FPGA)

### 8.1 Modules

* `md_parser.v`

  * **In**: 64-bit event words (serialized CSV rows or binary feed) + `clk`
  * **Out**: structured record (ts, type, side, price, qty, best\_bid/ask, …)
  * **Pipeline**: target ≥ 1 event/clk; measure **cycles/event** and **latency(ns)** in sim
* `pretrade_risk.v`

  * **In**: order req (price, qty, side), policy (size cap, price band, rate limit)
  * **Out**: pass/deny + reason code

### 8.2 Integration

* **Verilator** path (recommended): build to C++; engine calls `eval()` each tick to get structured events.
* **Sim-file path**: write parser outputs to a FIFO file/pipe the C++ engine consumes.

### 8.3 Deliverables

* Waveforms (or logs) demonstrating **per-stage latency** & **max throughput**.
* Unit tests feeding malformed packets to verify robust parse & risk denials.

---

## 9) C++ engine (order book & matching)

### 9.1 Core responsibilities

* Maintain price-indexed book with FIFO queues per level
* Apply market events (from Verilog parser)
* Accept orders (`submit`, `cancel`) from strategy bridge
* Compute fills under chosen model; emit exec reports
* **Telemetry**: timestamps at each stage to produce latency histograms

### 9.2 Telemetry points (emit to CSV)

* `t_feed_in` → `t_book_upd` → `t_order_recv` → `t_match_done` → `t_report_out`
* Produce p50/p95/p99 for: **book update**, **order→match**, **match→report**, **end-to-end**

### 9.3 Determinism

* Single-threaded matching critical path (or fixed scheduling) for reproducible fills
* Seeded RNG for tie-breaks if any pro-rata logic is used

---

## 10) Bridge options

### A) pybind11 (simplest, in-process)

* Expose `Engine` class to Python: `on_market_event`, `submit`, `cancel`, `poll_exec_reports`
* **Pros**: easy debugging, zero copy potential
* **Cons**: Python in same process—separate your **core latency** vs **Py invocation** metrics in reports

### B) ZeroMQ IPC (prod-like, two processes)

* Engine runs as server (`REQ/REP` or `PUB/SUB` duplex); Python client submits orders and receives exec reports
* **Pros**: isolates Python overhead, resembles production topology
* **Cons**: serialization cost; include it in a separate “bridge latency” section

---

## 11) Python strategy & evaluation

### 11.1 Built-ins

* **Market making**: quote at `mid ± k*spread`; inventory skew; adaptive spread with volatility
* **Mean reversion**: EMA/microprice deviations; stop loss/take profit; cool-down

### 11.2 Research→Execution demo (optional)

* Train a small model: logistic/XGBoost predicting short-horizon up/down
* Export:

  * Linear/logistic → JSON weights `{ "w": [...], "b": ... }`
  * XGBoost → binary model; C++ predicts via XGBoost C API
  * PyTorch/TF → ONNX; C++ runs **ONNX Runtime** inference

### 11.3 Reports (saved to `py_strategy/out/`)

* **PnL curve**, **drawdown**, **Sharpe (simple)**
* **Fill ratio** (orders filled / posted), **avg fill size**, **cancel rate**
* **Latency histograms** p50/p95/p99 (core vs bridge vs full path)
* **Microstructure stats**: spread distribution, book depth snapshots, trade size hist

---

## 12) Configuration (`configs/run.yaml` example)

```yaml
instrument: "BTCUSDT"
price_tick: 0.10
time_unit: "ns"

data:
  path: "data/binance_btcusdt_topbook_2023-06-01.csv"
  schema: "csv_topbook_v1"

fills:
  model: "top_of_book"      # or "queue"
  pro_rata: true
  min_quote_life_us: 500
  cancel_cooldown_us: 200

fees_bps:
  maker: 0.5
  taker: 1.0
slippage_bps: 0.0

risk:
  max_order_qty: 5.0
  price_band_bps: 50
  max_orders_per_sec: 200

bridge:
  type: "pybind11"          # or "zeromq"
```

---

## 13) Reproducibility & validation

* **Deterministic**: engine produces identical results given same data + config
* **Unit tests**: book invariants, boundary price matches, multi-level sweeps, partial fills
* **Stress tests**: ≥ 1e6 events replay; report throughput & memory footprint
* **Sanity checks**: PnL sanity under zero-edge (MM with zero spread edge should ≈0 after fees)

---

## 14) Interpreting “HFT-style” (what makes this high-frequency)

* **Tick-level** (not minute/day bars)
* **Order-book-based fills** (not “buy at close”)
* **Latency telemetry** at μs for engine, ns pipeline in Verilog sim
* **Cancel/replace mechanics** & posting behavior
* **Documented execution assumptions** (fill model, fees, queueing)

Include this statement (use in README/LinkedIn/Resume):

> *Execution core (order book & matching) is implemented in C++ with microsecond-level latency; FPGA-inspired Verilog modules (simulated via Verilator) handle market data parsing and pre-trade checks at nanosecond-level pipeline latency. Python is used only for strategy research and visualization. Historical tick data acts as the counterparty market, and fills are computed at the order-book level under documented assumptions.*

---

## 15) FAQ (short)

**Q: If Python is involved, is it still “high-frequency”?**
A: Yes—**core execution is C++** and measured separately. Python is the **research layer**. We publish latency split: core vs bridge vs end-to-end.

**Q: No real FPGA board—does Verilog still help?**
A: We simulate in ns via Verilator/ModelSim. The point is **hardware-pipeline thinking** and a **pre-trade risk block** with measured cycles/event.

**Q: Top-of-book only—is that realistic?**
A: For education it’s enough. You can extend to multi-level L2 depth deltas; engine already supports price-indexed levels.

---

## 16) Roadmap (optional extensions)

* Queue-position estimator for passive fills
* Multi-instrument / cross-venue arbitrage backtest
* ONNX Runtime inference in C++ for DL models
* Streamlit dashboard for real-time replay monitoring

---

## 17) License & data

* Code: MIT (or your choice).
* Data: ensure you have rights to use any third-party datasets; keep raw feeds out of repo if licenses forbid redistribution.

---

## 18) What reviewers should look at first

1. `docs/arch_diagram.png` (or ASCII in README)
2. `configs/run.yaml` (assumptions)
3. `py_strategy/out/*` (PnL, latency hist, fill ratio)
4. `engine_cpp/metrics/*.csv` (p50/p95/p99 latency, throughput)
5. `verilog/sim/` waveforms or logs (pipeline/throughput proof)

---

**This README is the contract**: clone, build, run, and you’ll see a complete **HFT-style research→execution loop** with **hardware-inspired preprocessing**, **μs-level C++ engine**, and **book-level fills** on historical tick data.
