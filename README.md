# k3s Quant MVP – Paper-Trading Platform

## Overview

This project is a small, **paper-trading quant platform** running on a local k3s cluster.  
It does **not** trade real money. What it actually does is the following:

- Pulls live (or stubbed) market price and volume data
- Computes simple features (price return + volume spike)
- Runs one rule-based strategy that generates BUY/SELL signals
- Simulates trades and tracks positions/PnL in Postgres
- Runs each component as its own service on k3s

High-level data flow:

> **market data → features → signals → execution → trades/positions → PnL**

---

## Architecture

The system is split into small services:

1. **market-data-service**
   - Periodically fetches price & volume from an API 
   - Writes raw ticks into a Postgres table `ticks`

2. **feature-engine-service**
   - Reads recent ticks from `ticks`
   - Computes simple features, e.g.:
     - 5-minute price return
     - 5-minute total volume
     - Volume spike vs typical 5-minute volume over the last hour
   - Writes into `features`

3. **strategy-service**
   - Reads latest features per symbol
   - Applies a simple threshold rule, for example:
     - `return_5m > 1%`
     - `volume_ratio > 3x`
   - Emits `BUY` (and later optional `SELL`) signals into `signals`

4. **execution-service**
   - Reads unprocessed signals
   - For a `BUY` signal:
     - Opens a paper position if none is open
     - Inserts a trade into `trades`
     - Inserts/updates position in `positions`
   - Optional later: handle `SELL`, close positions, compute PnL

5. **(Optional) monitor-service**
   - Simple CLI or script that queries Postgres
   - Prints out open positions, trades, and basic PnL summaries

All services are deployed as **Kubernetes Deployments** into a dedicated namespace on k3s, and share a **Postgres** database (and **Redis** later as a message bus).

---

## Tech Stack

- **Language:** Python (service code)
- **Orchestration:** k3s (lightweight Kubernetes)
- **Database:** Postgres (ticks, features, signals, positions, trades)
- **Containerization:** Docker images per service
- **Optional:** Redis, Prometheus, Grafana, etc. for later extensions

---

## Rough Folder Structure (Planned)

This structure will be filled in as the project progresses:

```text
quant-platform-mvp/
  README.md

  services/
    market_data/
      main.py
      Dockerfile
    feature_engine/
      main.py
      Dockerfile
    strategy/
      main.py
      Dockerfile
    execution/
      main.py
      Dockerfile

  k8s/
    namespace.yaml
    postgres.yaml
    redis.yaml              
    market-data-deploy.yaml
    feature-engine-deploy.yaml
    strategy-deploy.yaml
    execution-deploy.yaml

  sql/
    01_ticks.sql
    02_features.sql
    03_signals.sql
    04_positions.sql
    05_trades.sql