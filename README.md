# BlackHole Fund - Trading Infrastructure

## Overview

BlackHole Fund is a quantitative investment fund specializing in **Gold (XAU/USD)** trading within the forex market. Our infrastructure is designed for institutional-grade execution, combining cutting-edge quantitative analysis with robust risk management systems.

Our trading systems operate across **two AWS regions** (London & Manchester) ensuring high availability, disaster recovery, and optimal latency to major financial centers.

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              BLACKHOLE TRADING INFRASTRUCTURE                            │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│   ┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐            │
│   │  Market Data    │        │   Bloomberg     │        │   Exchange      │            │
│   │  Feeds (Tick)   │        │   Terminal API  │        │   Feeds         │            │
│   └────────┬────────┘        └────────┬────────┘        └────────┬────────┘            │
│            │                          │                          │                      │
│            └──────────────────────────┼──────────────────────────┘                      │
│                                       ▼                                                  │
│                        ┌──────────────────────────────┐                                 │
│                        │      MARKET CONNECTOR        │                                 │
│                        │      (bh-market-gateway)     │                                 │
│                        │         [Go + Rust]          │                                 │
│                        └──────────────┬───────────────┘                                 │
│                                       │                                                  │
│           ┌───────────────────────────┼───────────────────────────┐                     │
│           ▼                           ▼                           ▼                     │
│  ┌─────────────────┐      ┌─────────────────────┐      ┌─────────────────┐             │
│  │   MT5 TICK      │      │    QUANT ENGINE     │      │   NEWS/EVENTS   │             │
│  │   (mt5_tick)    │◄────►│  (bh-quant-engine)  │◄────►│   PROCESSOR     │             │
│  │     [C++]       │      │      [Python]       │      │                 │             │
│  └────────┬────────┘      └─────────┬───────────┘      └─────────────────┘             │
│           │                         │                                                   │
│           │                         │                                                   │
│           ▼                         ▼                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐           │
│  │                         ORCHESTRATOR (bh-core)                          │           │
│  │                              [Go]                                       │           │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │           │
│  │  │  Message    │  │  Service    │  │  Health     │  │  Config     │   │           │
│  │  │  Router     │  │  Discovery  │  │  Monitor    │  │  Manager    │   │           │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘   │           │
│  └─────────────────────────────────┬───────────────────────────────────────┘           │
│                                    │                                                    │
│           ┌────────────────────────┼────────────────────────┐                          │
│           ▼                        ▼                        ▼                          │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────┐            │
│  │  RISK ENGINE    │    │   CIRCUIT       │    │     MT5 EXECUTOR        │            │
│  │  (bh-risk)      │───►│   BREAKER       │───►│     (mt5_executor)      │            │
│  │     [Go]        │    │   (bh-guardian) │    │        [C++]            │            │
│  └─────────────────┘    │     [Rust]      │    └───────────┬─────────────┘            │
│                         └─────────────────┘                │                           │
│                                                            ▼                           │
│                                                   ┌─────────────────┐                  │
│                                                   │   MT5 BROKER    │                  │
│                                                   │   CONNECTION    │                  │
│                                                   └─────────────────┘                  │
│                                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────┐           │
│  │                         DATABASE LAYER                                   │           │
│  │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │           │
│  │   │ TimescaleDB │  │   Redis     │  │ PostgreSQL  │  │ InfluxDB    │   │           │
│  │   │ (Tick Data) │  │  (Cache)    │  │ (Analytics) │  │ (Metrics)   │   │           │
│  │   └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘   │           │
│  └─────────────────────────────────────────────────────────────────────────┘           │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘

                    AWS eu-west-2 (London) ◄──── Active-Active ────► AWS eu-west-1 (Manchester)
```

## Core Repositories

| Repository | Language | Description |
|------------|----------|-------------|
| [mt5_executor](docs/repositories/mt5_executor.md) | C++ | High-performance order execution engine |
| [mt5_tick](docs/repositories/mt5_tick.md) | C++ | Real-time tick data processor |
| [bh-risk](docs/repositories/bh-risk.md) | Go | Risk management and position sizing engine |
| [bh-guardian](docs/repositories/bh-guardian.md) | Rust | Daily circuit breaker and system protection |
| [bh-quant-engine](docs/repositories/bh-quant-engine.md) | Python | Quantitative analysis and volatility modeling |
| [bh-core](docs/repositories/bh-core.md) | Go | Central orchestration and service coordination |
| [bh-market-gateway](docs/repositories/bh-market-gateway.md) | Go/Rust | Market data connectors and feed handlers |

## Key Features

### Quantitative Analysis
- **Volatility Forecasting**: GARCH, EGARCH, TGARCH, FIGARCH models
- **Market Regime Detection**: Hidden Markov Models, Regime-Switching GARCH
- **Risk Metrics**: Monte Carlo VaR, CVaR/Expected Shortfall, Stress Testing
- **Statistical Indicators**: Hurst Exponent, Cointegration Analysis, Kalman Filtering

### Risk Management
- Real-time position monitoring and exposure limits
- Dynamic position sizing based on volatility regime
- **1% Daily Stop Loss** circuit breaker (hard limit)
- Multi-level risk controls (order, account, portfolio)

### Infrastructure
- **Dual-region AWS deployment** for high availability
- Sub-millisecond internal message latency
- Automated failover and disaster recovery
- Comprehensive monitoring and alerting

## Documentation

- [Architecture Overview](docs/architecture/overview.md)
- [Communication Protocols](docs/architecture/communication.md)
- [Quantitative Models](docs/quantitative/models.md)
- [Risk Framework](docs/risk-management/framework.md)
- [Market Connectors](docs/connectors/overview.md)

## Technology Stack

| Layer | Technologies |
|-------|-------------|
| **Execution** | C++ 20, MetaTrader 5 API |
| **Risk & Orchestration** | Go 1.22+, gRPC, Protocol Buffers |
| **Circuit Breaker** | Rust 1.75+, Tokio |
| **Quantitative** | Python 3.11+, NumPy, SciPy, Arch, Statsmodels |
| **Messaging** | ZeroMQ, Redis Streams, Apache Kafka |
| **Databases** | TimescaleDB, PostgreSQL, Redis, InfluxDB |
| **Infrastructure** | AWS EKS, Terraform, Prometheus, Grafana |

## Compliance & Security

All systems operate under strict compliance with financial regulations. Security measures include:
- End-to-end encryption for all inter-service communication
- Hardware Security Modules (HSM) for key management
- Comprehensive audit logging
- Role-based access control (RBAC)

---

*BlackHole Fund - Precision Trading Through Quantitative Excellence*
