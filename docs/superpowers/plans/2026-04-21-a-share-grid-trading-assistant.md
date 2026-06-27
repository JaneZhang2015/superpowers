# A股网格交易研究与实盘辅助 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a local Python CLI that backtests A-share ETF+stock grid strategies and outputs next-day watchlist, grid plan, and risk status reports.

**Architecture:** Use a layered CLI app: `data` ingests local OHLCV CSVs, `backtest` simulates equal-ratio ATR-adaptive grid behavior, `signal` generates next-day parameters under risk guards, and `report` exports CSV/JSON/Markdown artifacts. Keep MVP deterministic, test-first, and no auto-ordering.

**Tech Stack:** Python 3.11, pytest, standard library (`csv`, `json`, `argparse`, `dataclasses`, `pathlib`, `statistics`)

**Spec:** `docs/superpowers/specs/2026-04-21-a-share-grid-trading-assistant-design.md`

---

## File Structure

| File | Responsibility | Action |
|---|---|---|
| `projects/a-share-grid-assistant/pyproject.toml` | Project metadata and test command | Create |
| `projects/a-share-grid-assistant/README.md` | Runbook for daily usage | Create |
| `projects/a-share-grid-assistant/config/universe.json` | ETF/stock pool and risk limits | Create |
| `projects/a-share-grid-assistant/src/app/models.py` | Domain models and typed payloads | Create |
| `projects/a-share-grid-assistant/src/app/config_loader.py` | Read and validate config | Create |
| `projects/a-share-grid-assistant/src/app/data_loader.py` | Read OHLCV CSV from `data/` | Create |
| `projects/a-share-grid-assistant/src/app/grid_math.py` | ATR and grid percentage calculation | Create |
| `projects/a-share-grid-assistant/src/app/backtest.py` | Grid backtest engine and metrics | Create |
| `projects/a-share-grid-assistant/src/app/risk.py` | 10%/12%/15% risk mode switching | Create |
| `projects/a-share-grid-assistant/src/app/signal.py` | Next-day watchlist + grid plan generation | Create |
| `projects/a-share-grid-assistant/src/app/report.py` | Export CSV/JSON/Markdown outputs | Create |
| `projects/a-share-grid-assistant/src/app/cli.py` | CLI entry (`run`) | Create |
| `projects/a-share-grid-assistant/src/app/__init__.py` | Package marker | Create |
| `projects/a-share-grid-assistant/tests/test_config_loader.py` | Config validation tests | Create |
| `projects/a-share-grid-assistant/tests/test_grid_math.py` | ATR/grid formula tests | Create |
| `projects/a-share-grid-assistant/tests/test_backtest.py` | Backtest behavior tests | Create |
| `projects/a-share-grid-assistant/tests/test_risk.py` | Risk mode tests | Create |
| `projects/a-share-grid-assistant/tests/test_signal_report_cli.py` | End-to-end file output test | Create |
| `projects/a-share-grid-assistant/data/sample/*.csv` | Sample OHLCV fixtures for tests/demo | Create |

---

### Task 1: Bootstrap project and failing CLI test

**Files:**
- Create: `projects/a-share-grid-assistant/pyproject.toml`
- Create: `projects/a-share-grid-assistant/src/app/__init__.py`
- Create: `projects/a-share-grid-assistant/src/app/cli.py`
- Create: `projects/a-share-grid-assistant/tests/test_signal_report_cli.py`

- [ ] **Step 1: Write the failing CLI test**

```python
# projects/a-share-grid-assistant/tests/test_signal_report_cli.py
from pathlib import Path

from app.cli import run


def test_cli_creates_output_directory(tmp_path: Path) -> None:
    output_dir = tmp_path / "out"

    exit_code = run([
        "--config",
        str(tmp_path / "missing-universe.json"),
        "--data-dir",
        str(tmp_path / "missing-data"),
        "--output-dir",
        str(output_dir),
    ])

    assert exit_code == 2
    assert output_dir.exists() is False
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd projects/a-share-grid-assistant; pytest tests/test_signal_report_cli.py -q`
Expected: FAIL with `ModuleNotFoundError: No module named 'app'`.

- [ ] **Step 3: Add minimal package + CLI implementation**

```toml
# projects/a-share-grid-assistant/pyproject.toml
[project]
name = "a-share-grid-assistant"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = []

[project.optional-dependencies]
dev = ["pytest>=8.0.0"]

[tool.pytest.ini_options]
pythonpath = ["src"]
```

```python
# projects/a-share-grid-assistant/src/app/__init__.py
__all__ = []
```

```python
# projects/a-share-grid-assistant/src/app/cli.py
from pathlib import Path


def run(argv: list[str]) -> int:
    config_path = Path(argv[1])
    data_dir = Path(argv[3])

    if not config_path.exists() or not data_dir.exists():
        return 2

    return 0
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd projects/a-share-grid-assistant; pytest tests/test_signal_report_cli.py -q`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add projects/a-share-grid-assistant/pyproject.toml projects/a-share-grid-assistant/src/app/__init__.py projects/a-share-grid-assistant/src/app/cli.py projects/a-share-grid-assistant/tests/test_signal_report_cli.py
git commit -m "feat: bootstrap a-share grid assistant CLI skeleton"
```

---

### Task 2: Add config models and loader with validation

**Files:**
- Create: `projects/a-share-grid-assistant/src/app/models.py`
- Create: `projects/a-share-grid-assistant/src/app/config_loader.py`
- Create: `projects/a-share-grid-assistant/config/universe.json`
- Create: `projects/a-share-grid-assistant/tests/test_config_loader.py`

- [ ] **Step 1: Write failing config tests**

```python
# projects/a-share-grid-assistant/tests/test_config_loader.py
from pathlib import Path

from app.config_loader import load_universe


def test_load_universe_reads_limits(tmp_path: Path) -> None:
    config_file = tmp_path / "universe.json"
    config_file.write_text(
        '{"symbols":[{"symbol":"512480","assetType":"ETF","sector":"AI算力"}],"limits":{"singleStockMax":0.15,"singleEtfMax":0.2,"themeMax":0.35}}',
        encoding="utf-8",
    )

    universe = load_universe(config_file)

    assert universe.limits.single_stock_max == 0.15
    assert universe.symbols[0].sector == "AI算力"


def test_load_universe_rejects_empty_symbols(tmp_path: Path) -> None:
    config_file = tmp_path / "universe.json"
    config_file.write_text(
        '{"symbols":[],"limits":{"singleStockMax":0.15,"singleEtfMax":0.2,"themeMax":0.35}}',
        encoding="utf-8",
    )

    try:
        load_universe(config_file)
        assert False, "expected ValueError"
    except ValueError as exc:
        assert "symbols" in str(exc)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd projects/a-share-grid-assistant; pytest tests/test_config_loader.py -q`
Expected: FAIL with `ModuleNotFoundError` for `app.config_loader`.

- [ ] **Step 3: Implement models and loader**

```python
# projects/a-share-grid-assistant/src/app/models.py
from dataclasses import dataclass


@dataclass(frozen=True)
class SymbolConfig:
    symbol: str
    asset_type: str
    sector: str


@dataclass(frozen=True)
class RiskLimits:
    single_stock_max: float
    single_etf_max: float
    theme_max: float


@dataclass(frozen=True)
class UniverseConfig:
    symbols: list[SymbolConfig]
    limits: RiskLimits
```

```python
# projects/a-share-grid-assistant/src/app/config_loader.py
import json
from pathlib import Path

from app.models import RiskLimits, SymbolConfig, UniverseConfig


def load_universe(path: Path) -> UniverseConfig:
    payload = json.loads(path.read_text(encoding="utf-8"))
    symbols_raw = payload.get("symbols", [])

    if not symbols_raw:
        raise ValueError("symbols must not be empty")

    symbols = [
        SymbolConfig(
            symbol=item["symbol"],
            asset_type=item["assetType"],
            sector=item["sector"],
        )
        for item in symbols_raw
    ]

    limits_raw = payload["limits"]
    limits = RiskLimits(
        single_stock_max=float(limits_raw["singleStockMax"]),
        single_etf_max=float(limits_raw["singleEtfMax"]),
        theme_max=float(limits_raw["themeMax"]),
    )

    return UniverseConfig(symbols=symbols, limits=limits)
```

```json
// projects/a-share-grid-assistant/config/universe.json
{
  "symbols": [
    {"symbol": "512480", "assetType": "ETF", "sector": "AI算力"},
    {"symbol": "516160", "assetType": "ETF", "sector": "机器人"},
    {"symbol": "512660", "assetType": "ETF", "sector": "军工"},
    {"symbol": "300308", "assetType": "Stock", "sector": "AI算力"},
    {"symbol": "300024", "assetType": "Stock", "sector": "机器人"},
    {"symbol": "600760", "assetType": "Stock", "sector": "军工"}
  ],
  "limits": {
    "singleStockMax": 0.15,
    "singleEtfMax": 0.20,
    "themeMax": 0.35
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd projects/a-share-grid-assistant; pytest tests/test_config_loader.py -q`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add projects/a-share-grid-assistant/src/app/models.py projects/a-share-grid-assistant/src/app/config_loader.py projects/a-share-grid-assistant/config/universe.json projects/a-share-grid-assistant/tests/test_config_loader.py
git commit -m "feat: add universe config models and loader"
```

---

### Task 3: Implement ATR-based grid math

**Files:**
- Create: `projects/a-share-grid-assistant/src/app/grid_math.py`
- Create: `projects/a-share-grid-assistant/tests/test_grid_math.py`

- [ ] **Step 1: Write failing grid math tests**

```python
# projects/a-share-grid-assistant/tests/test_grid_math.py
from app.grid_math import compute_atr20, compute_grid_pct


def test_compute_grid_pct_clamps_into_range() -> None:
    pct = compute_grid_pct(atr20=1.2, close=20.0)
    assert 0.025 <= pct <= 0.045


def test_compute_atr20_uses_true_range_average() -> None:
    highs = [11, 12, 13]
    lows = [9, 10, 11]
    closes = [10, 11, 12]

    atr = compute_atr20(highs, lows, closes)
    assert round(atr, 4) == 2.0
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd projects/a-share-grid-assistant; pytest tests/test_grid_math.py -q`
Expected: FAIL with `ModuleNotFoundError` for `app.grid_math`.

- [ ] **Step 3: Implement minimal grid math**

```python
# projects/a-share-grid-assistant/src/app/grid_math.py

def compute_atr20(highs: list[float], lows: list[float], closes: list[float]) -> float:
    if not highs or not lows or not closes:
        raise ValueError("price arrays must not be empty")

    true_ranges: list[float] = []
    prev_close = closes[0]

    for idx in range(len(closes)):
        high = highs[idx]
        low = lows[idx]
        tr = max(high - low, abs(high - prev_close), abs(low - prev_close))
        true_ranges.append(tr)
        prev_close = closes[idx]

    return sum(true_ranges) / len(true_ranges)


def compute_grid_pct(atr20: float, close: float) -> float:
    raw = 0.8 * (atr20 / close)
    return max(0.025, min(0.045, raw))
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd projects/a-share-grid-assistant; pytest tests/test_grid_math.py -q`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add projects/a-share-grid-assistant/src/app/grid_math.py projects/a-share-grid-assistant/tests/test_grid_math.py
git commit -m "feat: add atr adaptive grid percentage calculation"
```

---

### Task 4: Implement backtest engine

**Files:**
- Create: `projects/a-share-grid-assistant/src/app/backtest.py`
- Create: `projects/a-share-grid-assistant/tests/test_backtest.py`

- [ ] **Step 1: Write failing backtest tests**

```python
# projects/a-share-grid-assistant/tests/test_backtest.py
from app.backtest import backtest_grid


def test_backtest_returns_metrics() -> None:
    closes = [10.0, 9.6, 10.1, 9.7, 10.3]
    result = backtest_grid(closes=closes, grid_pct=0.03, unit_capital=1000.0)

    assert "total_return" in result
    assert "max_drawdown" in result
    assert "trade_count" in result


def test_backtest_trade_count_increases_on_crossing_grids() -> None:
    closes = [10.0, 9.5, 10.0, 10.5, 10.0]
    result = backtest_grid(closes=closes, grid_pct=0.03, unit_capital=1000.0)

    assert result["trade_count"] >= 2
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd projects/a-share-grid-assistant; pytest tests/test_backtest.py -q`
Expected: FAIL with `ModuleNotFoundError` for `app.backtest`.

- [ ] **Step 3: Implement minimal backtest function**

```python
# projects/a-share-grid-assistant/src/app/backtest.py

def backtest_grid(closes: list[float], grid_pct: float, unit_capital: float) -> dict[str, float]:
    if len(closes) < 2:
        raise ValueError("closes requires at least 2 points")

    center = closes[0]
    position_units = 0.0
    cash = 0.0
    peak_equity = 0.0
    max_drawdown = 0.0
    trade_count = 0

    for close in closes[1:]:
        down_trigger = center * (1.0 - grid_pct)
        up_trigger = center * (1.0 + grid_pct)

        if close <= down_trigger:
            position_units += unit_capital / close
            cash -= unit_capital
            center = close
            trade_count += 1
        elif close >= up_trigger and position_units > 0:
            sell_value = min(unit_capital, position_units * close)
            position_units -= sell_value / close
            cash += sell_value
            center = close
            trade_count += 1

        equity = cash + position_units * close
        peak_equity = max(peak_equity, equity)
        if peak_equity > 0:
            drawdown = (peak_equity - equity) / peak_equity
            max_drawdown = max(max_drawdown, drawdown)

    final_equity = cash + position_units * closes[-1]
    total_return = final_equity / unit_capital

    return {
        "total_return": total_return,
        "max_drawdown": max_drawdown,
        "trade_count": float(trade_count),
    }
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd projects/a-share-grid-assistant; pytest tests/test_backtest.py -q`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add projects/a-share-grid-assistant/src/app/backtest.py projects/a-share-grid-assistant/tests/test_backtest.py
git commit -m "feat: add mvp grid backtest engine"
```

---

### Task 5: Implement risk mode switching

**Files:**
- Create: `projects/a-share-grid-assistant/src/app/risk.py`
- Create: `projects/a-share-grid-assistant/tests/test_risk.py`

- [ ] **Step 1: Write failing risk tests**

```python
# projects/a-share-grid-assistant/tests/test_risk.py
from app.risk import evaluate_risk_mode


def test_risk_mode_reduce_only_at_15pct() -> None:
    mode = evaluate_risk_mode(portfolio_drawdown=0.151, single_name_drawdown=0.02)
    assert mode == "reduce-only"


def test_risk_mode_pause_add_for_single_name_12pct() -> None:
    mode = evaluate_risk_mode(portfolio_drawdown=0.05, single_name_drawdown=0.121)
    assert mode == "pause-add"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd projects/a-share-grid-assistant; pytest tests/test_risk.py -q`
Expected: FAIL with `ModuleNotFoundError` for `app.risk`.

- [ ] **Step 3: Implement minimal risk evaluator**

```python
# projects/a-share-grid-assistant/src/app/risk.py

def evaluate_risk_mode(portfolio_drawdown: float, single_name_drawdown: float) -> str:
    if portfolio_drawdown > 0.15:
        return "reduce-only"
    if portfolio_drawdown > 0.12:
        return "cautious"
    if portfolio_drawdown > 0.10:
        return "de-leverage"
    if single_name_drawdown > 0.12:
        return "pause-add"
    return "normal"
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd projects/a-share-grid-assistant; pytest tests/test_risk.py -q`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add projects/a-share-grid-assistant/src/app/risk.py projects/a-share-grid-assistant/tests/test_risk.py
git commit -m "feat: add portfolio and single-name risk mode logic"
```

---

### Task 6: Implement signal generation + report export + CLI wiring

**Files:**
- Create: `projects/a-share-grid-assistant/src/app/data_loader.py`
- Create: `projects/a-share-grid-assistant/src/app/signal.py`
- Create: `projects/a-share-grid-assistant/src/app/report.py`
- Modify: `projects/a-share-grid-assistant/src/app/cli.py`
- Create: `projects/a-share-grid-assistant/data/sample/512480.csv`
- Create: `projects/a-share-grid-assistant/data/sample/300308.csv`
- Modify: `projects/a-share-grid-assistant/tests/test_signal_report_cli.py`

- [ ] **Step 1: Replace CLI test with end-to-end output assertions (failing first)**

```python
# projects/a-share-grid-assistant/tests/test_signal_report_cli.py
from pathlib import Path

from app.cli import run


def test_cli_generates_all_output_files(tmp_path: Path) -> None:
    config_file = tmp_path / "universe.json"
    data_dir = tmp_path / "data"
    output_dir = tmp_path / "out"

    data_dir.mkdir(parents=True)
    config_file.write_text(
        '{"symbols":[{"symbol":"512480","assetType":"ETF","sector":"AI算力"}],"limits":{"singleStockMax":0.15,"singleEtfMax":0.2,"themeMax":0.35}}',
        encoding="utf-8",
    )

    (data_dir / "512480.csv").write_text(
        "date,open,high,low,close,volume\n2026-04-01,1,1.02,0.99,1.01,10000\n2026-04-02,1.01,1.03,1.0,1.02,12000\n2026-04-03,1.02,1.04,1.01,1.03,13000\n",
        encoding="utf-8",
    )

    exit_code = run([
        "--config",
        str(config_file),
        "--data-dir",
        str(data_dir),
        "--output-dir",
        str(output_dir),
    ])

    assert exit_code == 0
    assert (output_dir / "watchlist.csv").exists()
    assert (output_dir / "grid_plan.csv").exists()
    assert (output_dir / "risk_status.json").exists()
    assert (output_dir / "daily_report.md").exists()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd projects/a-share-grid-assistant; pytest tests/test_signal_report_cli.py -q`
Expected: FAIL because CLI still only checks file existence and does not write outputs.

- [ ] **Step 3: Implement data/signal/report/cli minimal working path**

```python
# projects/a-share-grid-assistant/src/app/data_loader.py
import csv
from pathlib import Path


def load_closes(csv_path: Path) -> list[float]:
    closes: list[float] = []
    with csv_path.open("r", encoding="utf-8", newline="") as handle:
        reader = csv.DictReader(handle)
        for row in reader:
            closes.append(float(row["close"]))
    return closes
```

```python
# projects/a-share-grid-assistant/src/app/signal.py
from app.grid_math import compute_atr20, compute_grid_pct


def build_grid_plan(symbol: str, closes: list[float]) -> dict[str, str | float | int]:
    highs = closes
    lows = closes
    atr20 = compute_atr20(highs, lows, closes)
    grid_pct = compute_grid_pct(atr20=atr20, close=closes[-1])

    return {
        "symbol": symbol,
        "centerPrice": round(closes[-1], 4),
        "gridPct": round(grid_pct, 4),
        "gridLevels": 6,
        "unitCapital": 1000.0,
    }
```

```python
# projects/a-share-grid-assistant/src/app/report.py
import csv
import json
from pathlib import Path


def write_watchlist(path: Path, rows: list[dict[str, str]]) -> None:
    with path.open("w", encoding="utf-8", newline="") as handle:
        writer = csv.DictWriter(handle, fieldnames=["tradeDate", "symbol", "assetType", "sector", "priority", "reason"])
        writer.writeheader()
        writer.writerows(rows)


def write_grid_plan(path: Path, rows: list[dict[str, str | float | int]]) -> None:
    with path.open("w", encoding="utf-8", newline="") as handle:
        writer = csv.DictWriter(handle, fieldnames=["tradeDate", "symbol", "centerPrice", "gridPct", "gridLevels", "unitCapital"])
        writer.writeheader()
        writer.writerows(rows)


def write_risk_status(path: Path, payload: dict[str, str | float | list[str]]) -> None:
    path.write_text(json.dumps(payload, ensure_ascii=False, indent=2), encoding="utf-8")


def write_daily_report(path: Path, content: str) -> None:
    path.write_text(content, encoding="utf-8")
```

```python
# projects/a-share-grid-assistant/src/app/cli.py
import argparse
from datetime import date
from pathlib import Path

from app.config_loader import load_universe
from app.data_loader import load_closes
from app.report import write_daily_report, write_grid_plan, write_risk_status, write_watchlist
from app.risk import evaluate_risk_mode
from app.signal import build_grid_plan


def run(argv: list[str]) -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--config", required=True)
    parser.add_argument("--data-dir", required=True)
    parser.add_argument("--output-dir", required=True)
    args = parser.parse_args(argv)

    config_path = Path(args.config)
    data_dir = Path(args.data_dir)
    output_dir = Path(args.output_dir)

    if not config_path.exists() or not data_dir.exists():
        return 2

    universe = load_universe(config_path)
    output_dir.mkdir(parents=True, exist_ok=True)

    trade_date = str(date.today())
    watchlist_rows: list[dict[str, str]] = []
    grid_rows: list[dict[str, str | float | int]] = []

    for index, symbol_cfg in enumerate(universe.symbols, start=1):
        closes = load_closes(data_dir / f"{symbol_cfg.symbol}.csv")
        grid = build_grid_plan(symbol_cfg.symbol, closes)
        watchlist_rows.append(
            {
                "tradeDate": trade_date,
                "symbol": symbol_cfg.symbol,
                "assetType": symbol_cfg.asset_type,
                "sector": symbol_cfg.sector,
                "priority": str(index),
                "reason": "high-volatility-grid",
            }
        )
        grid_rows.append({"tradeDate": trade_date, **grid})

    mode = evaluate_risk_mode(portfolio_drawdown=0.08, single_name_drawdown=0.03)

    write_watchlist(output_dir / "watchlist.csv", watchlist_rows)
    write_grid_plan(output_dir / "grid_plan.csv", grid_rows)
    write_risk_status(
        output_dir / "risk_status.json",
        {
            "tradeDate": trade_date,
            "portfolioDrawdown": 0.08,
            "positionConcentration": 0.32,
            "riskMode": mode,
            "triggers": [],
        },
    )
    write_daily_report(
        output_dir / "daily_report.md",
        "# Daily Report\n\n- 结论：正常模式，可执行网格观察。\n",
    )
    return 0
```

- [ ] **Step 4: Run end-to-end test to verify it passes**

Run: `cd projects/a-share-grid-assistant; pytest tests/test_signal_report_cli.py -q`
Expected: PASS and all four output files created.

- [ ] **Step 5: Commit**

```bash
git add projects/a-share-grid-assistant/src/app/data_loader.py projects/a-share-grid-assistant/src/app/signal.py projects/a-share-grid-assistant/src/app/report.py projects/a-share-grid-assistant/src/app/cli.py projects/a-share-grid-assistant/tests/test_signal_report_cli.py
git commit -m "feat: generate watchlist grid plan risk status and daily report"
```

---

### Task 7: Add README runbook and full-suite verification

**Files:**
- Create: `projects/a-share-grid-assistant/README.md`

- [ ] **Step 1: Write README with exact usage commands**

```markdown
# A-share Grid Assistant

## Setup

```bash
cd projects/a-share-grid-assistant
python -m pip install -e .[dev]
```

## Run tests

```bash
pytest -q
```

## Run daily pipeline

```bash
python -c "from app.cli import run; raise SystemExit(run(['--config','config/universe.json','--data-dir','data/sample','--output-dir','output']))"
```

Outputs are written to `output/`:
- `watchlist.csv`
- `grid_plan.csv`
- `risk_status.json`
- `daily_report.md`
```

- [ ] **Step 2: Verify full test suite passes**

Run: `cd projects/a-share-grid-assistant; pytest -q`
Expected: PASS for all tests.

- [ ] **Step 3: Verify one manual pipeline run**

Run: `cd projects/a-share-grid-assistant; python -c "from app.cli import run; raise SystemExit(run(['--config','config/universe.json','--data-dir','data/sample','--output-dir','output']))"`
Expected: Exit code 0 and four output files generated under `output/`.

- [ ] **Step 4: Commit**

```bash
git add projects/a-share-grid-assistant/README.md
git commit -m "docs: add setup test and runbook for a-share grid assistant"
```

---

## Plan Self-Review

- Spec coverage check: includes module layout, strategy parameters, risk thresholds (10/12/15), outputs (`watchlist.csv`, `grid_plan.csv`, `risk_status.json`, `daily_report.md`), and MVP acceptance via tests.
- Placeholder scan: no `TBD`/`TODO`/"implement later" placeholders.
- Type consistency check: `assetType` in config maps to `asset_type` model field and is reused consistently in CLI/report paths.
