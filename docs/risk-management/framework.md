# Risk Management Framework

## Overview

BlackHole Fund employs a comprehensive, multi-layered risk management framework designed to protect capital while enabling systematic trading in the gold market. Our approach combines quantitative risk metrics with hard operational limits.

## Risk Philosophy

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         RISK MANAGEMENT PRINCIPLES                              │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   1. CAPITAL PRESERVATION FIRST                                                │
│      The primary objective is protecting capital. Returns are secondary        │
│      to survival.                                                               │
│                                                                                 │
│   2. MULTIPLE INDEPENDENT SAFEGUARDS                                           │
│      No single point of failure. Risk controls operate at multiple levels     │
│      with independent systems.                                                  │
│                                                                                 │
│   3. HARD LIMITS ARE ABSOLUTE                                                  │
│      Circuit breakers cannot be overridden during market hours. Period.        │
│                                                                                 │
│   4. ADAPT TO REGIME                                                           │
│      Risk parameters dynamically adjust based on market conditions.            │
│                                                                                 │
│   5. TRANSPARENCY AND AUDITABILITY                                             │
│      Every risk decision is logged and auditable.                              │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Risk Control Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         FOUR LINES OF DEFENSE                                   │
└─────────────────────────────────────────────────────────────────────────────────┘

LEVEL 1: ORDER-LEVEL CONTROLS
├── Volume limits per order
├── Price sanity checks
├── Symbol validation
└── Duplicate detection

         │
         ▼

LEVEL 2: POSITION-LEVEL CONTROLS (bh-risk)
├── Position sizing based on volatility
├── Single position exposure limits
├── Correlated exposure limits
└── VaR contribution limits

         │
         ▼

LEVEL 3: PORTFOLIO-LEVEL CONTROLS (bh-risk)
├── Gross exposure limits
├── Net exposure limits
├── Sector concentration limits
└── Drawdown monitoring

         │
         ▼

LEVEL 4: SYSTEM-LEVEL CONTROLS (bh-guardian)
├── Daily loss limit (1%)
├── Emergency halt capability
├── Automatic position closeout
└── Manual override (authorized only)
```

## Quantitative Risk Metrics

### Value at Risk (VaR)

Daily VaR calculations at multiple confidence levels:

| Confidence | Interpretation | Limit |
|------------|----------------|-------|
| 95% VaR | 1 in 20 days exceeded | 1.5% of NAV |
| 99% VaR | 1 in 100 days exceeded | 2.5% of NAV |
| 99.9% VaR | 1 in 1000 days exceeded | 4.0% of NAV |

**Calculation Methods**:
- Historical simulation (252-day rolling window)
- Parametric (assuming Student-t distribution)
- Monte Carlo (10,000 simulations)

**Final VaR**: Conservative estimate using maximum of all methods

### Expected Shortfall (CVaR)

Average loss in tail scenarios:

| Metric | Target | Hard Limit |
|--------|--------|------------|
| CVaR 95% | < 1.8% | 2.5% |
| CVaR 99% | < 3.0% | 4.0% |

CVaR is used for:
- Capital allocation
- Position sizing optimization
- Regulatory reporting (FRTB compliance)

### Drawdown Limits

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         DRAWDOWN CONTROL FRAMEWORK                              │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   DAILY DRAWDOWN                                                               │
│   ═══════════════                                                              │
│   0.00% ────────────────────────────────────────────────────── 0% (Start)     │
│          │                                                                      │
│   0.50% ─┼───────────────────────────────── WARNING (Position size -25%)      │
│          │                                                                      │
│   0.75% ─┼─────────────────────── ALERT (No new positions, reduce exposure)   │
│          │                                                                      │
│   1.00% ─┼─────────── HALT (Circuit breaker - close all positions)            │
│          │                                                                      │
│          ▼                                                                      │
│                                                                                 │
│   WEEKLY DRAWDOWN                                                              │
│   ════════════════                                                             │
│   2.0%  → Warning level                                                        │
│   3.0%  → No new positions                                                     │
│   4.0%  → Strategy review required                                             │
│                                                                                 │
│   MONTHLY DRAWDOWN                                                             │
│   ═════════════════                                                            │
│   5.0%  → Warning level                                                        │
│   8.0%  → Formal review required                                               │
│                                                                                 │
│   PEAK-TO-TROUGH (Any Period)                                                  │
│   ════════════════════════════                                                 │
│   10.0% → Mandatory strategy review                                            │
│   15.0% → Trading suspension pending review                                    │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Position Sizing

### Volatility-Adjusted Sizing

Position size is inversely proportional to current volatility:

$$\text{Position Size} = \frac{\text{Risk Budget} \times \text{Account Equity}}{\text{ATR} \times \text{ATR Multiplier}}$$

**Parameters**:
- Risk Budget: 1% of equity per trade (max)
- ATR Period: 14-day Average True Range
- ATR Multiplier: 2.0 (adjustable by regime)

### Kelly Criterion (Modified)

Optimal sizing based on edge and win rate:

$$f^* = \frac{p \cdot b - q}{b}$$

Where:
- $p$ = Win probability
- $q$ = Loss probability (1-p)
- $b$ = Win/Loss ratio

**Implementation**: Fractional Kelly (0.25x) for variance reduction

### Regime-Based Adjustments

| Volatility Regime | Size Multiplier | Max Position |
|-------------------|-----------------|--------------|
| Low (< 10% ann) | 1.25x | 30% of NAV |
| Normal (10-20%) | 1.00x | 25% of NAV |
| High (20-35%) | 0.50x | 15% of NAV |
| Extreme (> 35%) | 0.25x | 10% of NAV |

## Exposure Limits

### Single Position Limits

| Limit Type | Threshold | Action |
|------------|-----------|--------|
| Notional value | 25% NAV | Reject new orders |
| Margin utilization | 50% | Warning |
| Margin utilization | 70% | Reduce exposure |
| Margin utilization | 80% | Close positions |

### Portfolio Exposure

| Exposure Type | Limit | Calculation |
|---------------|-------|-------------|
| Gross Exposure | 200% NAV | Σ|position| / NAV |
| Net Exposure | ±100% NAV | Σposition / NAV |
| Correlation-Adjusted | 150% NAV | Risk-weighted sum |

## Stress Testing

### Standard Scenarios

| Scenario | Description | Expected Impact |
|----------|-------------|-----------------|
| Gold -5% | Flash crash | -1.25% to -2.5% |
| Gold +5% | Sharp rally | Variable |
| VIX +50% | Volatility spike | Widen stops |
| DXY +3% | Dollar strength | -0.5% to -1.5% |
| Liquidity shock | Spread widening | -0.3% slippage |

### Tail Risk Scenarios

| Scenario | Gold | VIX | Probability | Max Loss |
|----------|------|-----|-------------|----------|
| 2008-style crisis | -15% | +200% | 0.1% | -4% (limited) |
| Flash crash | -10% | +100% | 0.5% | -2.5% (circuit) |
| Geopolitical event | +8% | +80% | 1% | Gains likely |

### Stress Test Frequency

- Daily: Standard scenarios
- Weekly: Extended historical scenarios
- Monthly: Comprehensive tail risk analysis
- Quarterly: Full system stress test

## Circuit Breaker System

### bh-guardian Operation

The circuit breaker (bh-guardian) operates independently of all other systems:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         CIRCUIT BREAKER LOGIC                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   INPUTS:                                                                       │
│   ├── Real-time position values                                                │
│   ├── Current market prices (from mt5_tick)                                    │
│   ├── Day-start equity (locked at market open)                                 │
│   └── Realized P&L (from execution reports)                                    │
│                                                                                 │
│   CALCULATION (Every 100ms):                                                   │
│   ├── Unrealized P&L = Σ(position_value - entry_value)                        │
│   ├── Total P&L = Realized P&L + Unrealized P&L                               │
│   └── Drawdown % = -Total P&L / Day-Start Equity × 100                        │
│                                                                                 │
│   ACTIONS:                                                                      │
│   ├── DD ≥ 0.5%:  Set state = REDUCED (position sizes halved)                 │
│   ├── DD ≥ 0.75%: Set state = CLOSING (close-only mode)                       │
│   └── DD ≥ 1.0%:  Set state = HALTED (emergency closeout)                     │
│                                                                                 │
│   RECOVERY:                                                                     │
│   ├── Automatic reset at day start (00:00 UTC)                                │
│   ├── Manual unlock requires: 30-min cooldown + authorized operator            │
│   └── All unlocks logged with full audit trail                                 │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Emergency Closeout Procedure

When 1% daily loss is reached:

1. **Immediate**: All pending orders cancelled
2. **T+0s**: State set to HALTED
3. **T+0s**: Alert sent to all channels (PagerDuty, Slack, SMS)
4. **T+1s**: Begin market order closeout of all positions
5. **T+5s**: Verify all positions closed
6. **T+10s**: Final P&L reconciliation
7. **T+30min**: Earliest possible manual unlock

## Operational Risk Controls

### System Redundancy

| Component | Primary | Backup | Failover Time |
|-----------|---------|--------|---------------|
| Execution | mt5_executor (LD4) | mt5_executor (LD5) | < 5s |
| Risk Engine | bh-risk (eu-west-2a) | bh-risk (eu-west-2b) | < 10s |
| Circuit Breaker | bh-guardian (eu-west-2a) | bh-guardian (eu-west-2b) | < 5s |
| Market Data | Bloomberg | Reuters | < 5s |

### Fail-Safe Defaults

If any critical system becomes unavailable:

| System Unavailable | Default Behavior |
|--------------------|------------------|
| bh-risk | Reject all new orders |
| bh-guardian | Reject all new orders |
| mt5_tick | Pause trading |
| bh-quant-engine | Use last known signals |
| Database | Continue with cached data |

## Monitoring & Reporting

### Real-Time Dashboard

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         RISK MONITORING DASHBOARD                               │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   CURRENT STATUS: ████ ACTIVE                                                  │
│                                                                                 │
│   Daily P&L:        +$12,450  (+0.25%)    ██████████░░░░░░░░░░  25% of limit  │
│   Daily Drawdown:   -0.15%                ███░░░░░░░░░░░░░░░░░  15% of limit  │
│   VaR (95%):        $45,000   (0.90%)     █████████░░░░░░░░░░░  60% of limit  │
│   CVaR (95%):       $62,000   (1.24%)     ████████████░░░░░░░░  69% of limit  │
│                                                                                 │
│   POSITIONS                                                                     │
│   ───────────────────────────────────────────────────────────────────────────  │
│   XAUUSD Long    2.5 lots    Entry: 2032.50    Current: 2035.00    +$6,250    │
│   XAUUSD Short   1.0 lots    Entry: 2038.00    Current: 2035.00    +$3,000    │
│                                                                                 │
│   EXPOSURE                                                                      │
│   ───────────────────────────────────────────────────────────────────────────  │
│   Gross: $875,000 (17.5% NAV)                                                  │
│   Net:   $375,000 (7.5% NAV)                                                   │
│                                                                                 │
│   ALERTS: None                                                                  │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Reporting Schedule

| Report | Frequency | Recipients |
|--------|-----------|------------|
| Real-time dashboard | Continuous | Trading desk |
| Daily risk summary | EOD | Risk committee |
| Weekly risk review | Weekly | Management |
| Monthly risk report | Monthly | Investors (summary) |
| Stress test results | Monthly | Risk committee |

## Governance

### Risk Committee

- **Chair**: Chief Risk Officer
- **Members**: CIO, Head of Trading, Head of Technology
- **Meeting Frequency**: Weekly (or as needed)
- **Authority**: Can modify risk parameters, suspend trading

### Parameter Changes

All risk parameter changes require:
1. Written proposal with justification
2. Risk committee approval
3. Backtesting/simulation results
4. Implementation during non-trading hours
5. Full audit trail

### Audit Trail

Every risk decision is logged with:
- Timestamp (nanosecond precision)
- Input parameters
- Calculation results
- Final decision
- System state
- Operator (if manual)
