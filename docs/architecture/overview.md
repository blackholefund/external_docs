# Architecture Overview

## System Design Philosophy

BlackHole Fund's trading infrastructure follows a **microservices architecture** optimized for:

- **Low Latency**: Sub-millisecond message processing
- **High Availability**: 99.99% uptime SLA
- **Scalability**: Horizontal scaling for market data processing
- **Resilience**: Multiple layers of failover protection

## Multi-Region Deployment

```
                                    ┌──────────────────────────┐
                                    │     Global DNS          │
                                    │   (Route 53 Latency)    │
                                    └───────────┬─────────────┘
                                                │
                    ┌───────────────────────────┼───────────────────────────┐
                    │                           │                           │
                    ▼                           │                           ▼
    ┌───────────────────────────────┐          │          ┌───────────────────────────────┐
    │      AWS eu-west-2            │          │          │      AWS eu-west-1            │
    │        (LONDON)               │          │          │      (MANCHESTER)             │
    │                               │          │          │                               │
    │  ┌─────────────────────────┐  │          │          │  ┌─────────────────────────┐  │
    │  │       EKS Cluster       │  │          │          │  │       EKS Cluster       │  │
    │  │                         │  │          │          │  │                         │  │
    │  │  ┌─────┐ ┌─────┐       │  │          │          │  │  ┌─────┐ ┌─────┐       │  │
    │  │  │Core │ │Risk │       │  │◄─────────┼─────────►│  │  │Core │ │Risk │       │  │
    │  │  └─────┘ └─────┘       │  │   Cross  │          │  │  └─────┘ └─────┘       │  │
    │  │  ┌─────┐ ┌─────┐       │  │  Region  │          │  │  ┌─────┐ ┌─────┐       │  │
    │  │  │Quant│ │Guard│       │  │   Sync   │          │  │  │Quant│ │Guard│       │  │
    │  │  └─────┘ └─────┘       │  │          │          │  │  └─────┘ └─────┘       │  │
    │  └─────────────────────────┘  │          │          │  └─────────────────────────┘  │
    │                               │          │          │                               │
    │  ┌─────────────────────────┐  │          │          │  ┌─────────────────────────┐  │
    │  │     Database Cluster    │  │          │          │  │     Database Cluster    │  │
    │  │  TimescaleDB + Postgres │◄─┼──────────┼─────────►│  │  TimescaleDB + Postgres │  │
    │  │      (Primary)          │  │  Async   │          │  │      (Replica)          │  │
    │  └─────────────────────────┘  │  Repl    │          │  └─────────────────────────┘  │
    │                               │          │          │                               │
    └───────────────────────────────┘          │          └───────────────────────────────┘
                                               │
                                               │
                              ┌────────────────┴────────────────┐
                              │                                 │
                              ▼                                 ▼
                    ┌──────────────────┐              ┌──────────────────┐
                    │   Colocation     │              │   Colocation     │
                    │   Equinix LD4    │              │   Equinix LD5    │
                    │                  │              │                  │
                    │  ┌────────────┐  │              │  ┌────────────┐  │
                    │  │mt5_executor│  │              │  │mt5_executor│  │
                    │  │  mt5_tick  │  │              │  │  mt5_tick  │  │
                    │  └────────────┘  │              │  └────────────┘  │
                    └──────────────────┘              └──────────────────┘
```

## Service Topology

### Tier 1: Market Interface Layer
Services that directly interface with external markets and brokers.

| Service | Location | Latency Requirement |
|---------|----------|---------------------|
| mt5_executor | Colocation | < 1ms |
| mt5_tick | Colocation | < 500μs |
| bh-market-gateway | AWS/Colo | < 5ms |

### Tier 2: Intelligence Layer
Services that process data and make trading decisions.

| Service | Location | Processing Window |
|---------|----------|-------------------|
| bh-quant-engine | AWS | 100ms - 5min |
| bh-risk | AWS | < 10ms |
| bh-guardian | AWS | < 1ms |

### Tier 3: Orchestration Layer
Services that coordinate system operations.

| Service | Location | Role |
|---------|----------|------|
| bh-core | AWS | Central coordination |

## Data Flow Architecture

```
                                  MARKET DATA FLOW
═══════════════════════════════════════════════════════════════════════════════

[Exchange Feeds] ──► [bh-market-gateway] ──► [Redis Streams] ──► [Consumers]
                            │                                         │
                            │                                         ├── bh-quant-engine
                            │                                         ├── bh-risk
                            │                                         └── bh-core
                            │
                            └──► [TimescaleDB] (Historical Storage)


                                  ORDER FLOW
═══════════════════════════════════════════════════════════════════════════════

[Signal] ──► [bh-core] ──► [bh-risk] ──► [bh-guardian] ──► [mt5_executor] ──► [Broker]
                │              │              │                  │
                │              │              │                  │
                ▼              ▼              ▼                  ▼
           [Audit Log]   [Risk Log]    [Circuit Log]      [Execution Log]


                                  RISK FLOW
═══════════════════════════════════════════════════════════════════════════════

[Position Update] ──────────────────────────────────────────────────────────┐
                                                                            │
[Market Price] ──► [bh-risk] ──► [Risk Assessment] ──► [Position Sizing]    │
                       │                                      │             │
                       │                                      ▼             │
                       │                               [Order Decision]     │
                       │                                      │             │
                       ▼                                      │             │
                 [bh-guardian] ◄──────────────────────────────┘             │
                       │                                                    │
                       │ (If Daily Loss > 1%)                               │
                       ▼                                                    │
                 [HALT ALL SYSTEMS]◄────────────────────────────────────────┘
```

## High Availability Design

### Active-Active Configuration

Both AWS regions operate in **active-active** mode:

1. **Traffic Distribution**: Route 53 latency-based routing
2. **State Synchronization**: Redis Cluster with cross-region replication
3. **Database**: TimescaleDB with streaming replication
4. **Failover Time**: < 30 seconds automatic failover

### Failure Scenarios

| Scenario | Impact | Recovery |
|----------|--------|----------|
| Single service failure | None | Kubernetes auto-restart |
| AZ failure | Minimal | Traffic routed to other AZs |
| Region failure | Temporary | Automatic failover to DR region |
| Broker connection loss | Trading paused | Automatic reconnection |

## Network Architecture

### Internal Communication

```
┌─────────────────────────────────────────────────────────────────┐
│                        VPC (10.0.0.0/16)                        │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Private Subnet (10.0.1.0/24)                │   │
│  │                                                          │   │
│  │   ┌─────────┐    ┌─────────┐    ┌─────────┐             │   │
│  │   │ bh-core │◄──►│ bh-risk │◄──►│bh-guard │             │   │
│  │   └─────────┘    └─────────┘    └─────────┘             │   │
│  │        ▲              ▲              ▲                   │   │
│  │        │              │              │                   │   │
│  │        └──────────────┼──────────────┘                   │   │
│  │                       │                                  │   │
│  │                       ▼                                  │   │
│  │              ┌─────────────────┐                         │   │
│  │              │  Service Mesh   │                         │   │
│  │              │    (Istio)      │                         │   │
│  │              └─────────────────┘                         │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Data Subnet (10.0.2.0/24)                   │   │
│  │                                                          │   │
│  │   ┌──────────┐  ┌──────────┐  ┌──────────┐              │   │
│  │   │TimescaleDB│  │PostgreSQL│  │  Redis   │              │   │
│  │   └──────────┘  └──────────┘  └──────────┘              │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### External Connectivity

| Connection | Protocol | Security |
|------------|----------|----------|
| Bloomberg API | REST/WebSocket | mTLS + API Key |
| Exchange Feeds | FIX 4.4 | VPN + mTLS |
| MT5 Broker | MT5 Protocol | Encrypted Channel |
| Cross-Region | AWS PrivateLink | VPC Peering + TLS |

## Monitoring & Observability

### Metrics Collection

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Services   │────►│ Prometheus  │────►│   Grafana   │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │ AlertManager│────► PagerDuty
                    └─────────────┘
```

### Key Metrics Monitored

- **Latency**: Order execution time, tick-to-trade latency
- **Throughput**: Messages per second, orders per second
- **Risk**: Current drawdown, position exposure, VaR
- **System**: CPU, memory, network, disk I/O

## Disaster Recovery

### RPO/RTO Targets

| System | RPO | RTO |
|--------|-----|-----|
| Trading Core | 0 (synchronous) | < 30s |
| Market Data | < 1s | < 60s |
| Analytics | < 5min | < 5min |
| Audit Logs | 0 | < 1min |

### Backup Strategy

- **Real-time**: Redis replication, PostgreSQL streaming
- **Hourly**: TimescaleDB continuous archiving
- **Daily**: Full database snapshots to S3
- **Weekly**: Cross-region backup verification
