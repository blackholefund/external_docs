# bh-risk

## Overview

**bh-risk** is the risk and position-sizing engine of the Genese Capital (formerly BlackHole Capital) infrastructure, written in Go. It evaluates every order before it reaches the `bh-guardian` circuit breaker and the execution layer.

## Technical Specifications

| Attribute | Value |
|-----------|-------|
| **Language** | Go 1.22+ |
| **Framework** | gRPC, Redis, PostgreSQL |
| **Role** | Risk and position sizing |

## Responsibilities

| Area | Rule |
|------|------|
| Position sizing | GARCH volatility model: higher volatility reduces lot size, lower volatility allows larger size within limits |
| Size per order | Approx. 7.5 lots maximum on a NAV of approx. 4.2 million; limits scale in lots per million of NAV |
| Per-position stop | 25.00 move in the gold price (2,500 per lot) |
| Margin | Internal limit of 10% of NAV |
| Spread filter | No new entries above a defined spread threshold; protective closure if spreads widen abruptly after entry |
| News filter | Blocks entries around high-impact economic events (calendar and news via API) |
| Liquidity | Size checked against liquidity-provider top-of-book depth before entry |
| Lot adjustments | System-generated; require management approval |

The daily (1% of NAV, since October 2025) and cumulative (5% of NAV) loss limits are enforced by [bh-guardian](bh-guardian.md).

## Architecture

```mermaid
flowchart TB
    subgraph bhRisk["bh-risk"]
        subgraph grpc["gRPC Service Layer"]
            EvaluateOrd["Order evaluation"]
            GetMetrics["Risk metrics"]
        end

        subgraph core["Risk Engine Core"]
            Sizer["Position Sizer\n(GARCH volatility)"]
            Limits["Limit Checker\n(size per order, margin, stop)"]
            Filters["Entry Filters\n(spread, news, top-of-book)"]
        end

        subgraph data["Data Layer"]
            Redis["Redis (Cache)"]
            PostgreSQL["PostgreSQL (Persist)"]
            QuantFeed["bh-quant-engine (Volatility)"]
            Market["Market (Prices)"]
        end

        grpc --> core
        core --> data
    end
```

## Risk Evaluation Flow

```mermaid
flowchart TD
    Input["Input: OrderRequest"] --> Filters{"News / spread /\ntop-of-book OK?"}
    Filters -->|NO| RejectFilter["REJECT: Filter"]
    Filters -->|YES| Size["Size from GARCH volatility\n(scaled to NAV)"]
    Size --> Limits{"Within size-per-order\nand margin limits?"}
    Limits -->|NO| RejectLimit["REJECT / reduce"]
    Limits -->|YES| Output["Output: RiskDecision\n(approved, volume, stop)"]
    Output --> Guardian["bh-guardian"]
```

## Integration Points

```mermaid
flowchart TD
    bhQuant["bh-quant-engine (Volatility)"] --> bhRisk["bh-risk"]
    bhCore["bh-core (Orders)"] --> bhRisk
    bhRisk --> bhGuardian["bh-guardian (Circuit breaker)"]
    bhGuardian --> mt5Executor["mt5_executor (Execution)"]
```
