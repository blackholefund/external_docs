# Architecture Overview

## System Design Philosophy

BlackHole Fund's trading infrastructure follows a **microservices architecture** optimized for:

- **Low Latency**: Single-digit millisecond targets under normal market conditions
- **High Availability**: Availability targets with active monitoring and failover
- **Scalability**: Horizontal scaling for market data processing
- **Resilience**: Multiple layers of failover protection

## Multi-Region Deployment

```mermaid
flowchart TB
    subgraph DNS["Global DNS Layer"]
        R53["Global Traffic Manager\nLatency-Based Routing"]
    end

    subgraph LD4["ld4 (Primary)"]
        subgraph L_EKS["Kubernetes Cluster"]
            L_Core["bh-core"]
            L_Risk["bh-risk"]
            L_Quant["bh-quant-engine"]
            L_Guard["bh-guardian"]
        end
        subgraph L_Data["Database Cluster"]
            L_TS["TimescaleDB\n(Primary)"]
            L_PG["PostgreSQL"]
            L_Redis["Redis"]
        end
        subgraph L_Exec["Execution Layer\n(Near Liquidity Provider)"]
            L_MT5E["mt5_executor"]
            L_MT5T["mt5_tick"]
        end
    end

    subgraph DR["DR Site (EU)"]
        subgraph M_EKS["Kubernetes Cluster"]
            M_Core["bh-core"]
            M_Risk["bh-risk"]
            M_Quant["bh-quant-engine"]
            M_Guard["bh-guardian"]
        end
        subgraph M_Data["Database Cluster"]
            M_TS["TimescaleDB\n(Replica)"]
            M_PG["PostgreSQL"]
            M_Redis["Redis"]
        end
        subgraph M_Exec["Execution Layer\n(Near Liquidity Provider)"]
            M_MT5E["mt5_executor"]
            M_MT5T["mt5_tick"]
        end
    end

    R53 --> L_EKS
    R53 --> M_EKS

    L_EKS <-->|Cross-Region Sync| M_EKS
    L_Data <-->|Async Replication| M_Data
    L_Exec <-->|Failover| M_Exec
```

## Service Topology

### Tier 1: Market Interface Layer
Services that directly interface with external markets and brokers.

| Service | Location | Latency Target |
|---------|----------|----------------|
| mt5_executor | Near LP | Single-digit ms (target) |
| mt5_tick | Near LP | Single-digit ms (target) |
| bh-market-gateway | Core DC | Low-latency target |

### Tier 2: Intelligence Layer
Services that process data and make trading decisions.

| Service | Location | Processing Window |
|---------|----------|-------------------|
| bh-quant-engine | Core DC | 100ms - 5min (model-dependent) |
| bh-risk | Core DC | Low-latency target |
| bh-guardian | Core DC | Low-latency target |

### Tier 3: Orchestration Layer
Services that coordinate system operations.

| Service | Location | Role |
|---------|----------|------|
| bh-core | Core DC | Central coordination |

## Data Flow Architecture

### Market Data Flow

```mermaid
flowchart LR
    subgraph Sources["External Sources"]
        Premium["Licensed Providers"]
        Exchanges["Exchanges"]
    end

    subgraph Gateway["Gateway"]
        MG["bh-market-gateway"]
    end

    subgraph Stream["Message Streams"]
        Redis["Redis Streams"]
        Kafka["Kafka"]
    end

    subgraph Consumers["Consumers"]
        Quant["bh-quant-engine"]
        Risk["bh-risk"]
        Core["bh-core"]
    end

    subgraph Storage["Storage"]
        TS["TimescaleDB"]
    end

    Premium --> MG
    Exchanges --> MG

    MG --> Redis
    MG --> Kafka
    MG --> TS

    Redis --> Quant
    Redis --> Risk
    Redis --> Core
```

### Order Flow

```mermaid
flowchart LR
    Signal["Trading\nSignal"] --> Core["bh-core"]
    Core --> Risk["bh-risk"]
    Risk --> Guardian["bh-guardian"]
    Guardian --> Executor["mt5_executor"]
    Executor --> Broker["MT5 Broker"]

    Core --> AuditLog["Audit Log"]
    Risk --> RiskLog["Risk Log"]
    Guardian --> CircuitLog["Circuit Log"]
    Executor --> ExecLog["Execution Log"]
```

### Risk Flow

```mermaid
flowchart TB
    subgraph Inputs["Real-Time Inputs"]
        Price["Market Price"]
        Position["Position Update"]
        PnL["P&L Update"]
    end

    subgraph RiskEngine["Risk Engine (bh-risk)"]
        Assessment["Risk Assessment"]
        Sizing["Position Sizing"]
        Decision["Order Decision"]
    end

    subgraph Circuit["Circuit Breaker (bh-guardian)"]
        Check["Daily Loss Check"]
        Halt["HALT TRADING\n(if loss > 1%)"]
    end

    Price --> Assessment
    Position --> Assessment
    PnL --> Assessment

    Assessment --> Sizing
    Sizing --> Decision
    Decision --> Check

    Check -->|Loss > 1%| Halt
    Check -->|Loss OK| Execute["Execute Order"]
```

## High Availability Design

### Active-Active Configuration

Both data centers operate in **active-active** mode:

1. **Traffic Distribution**: Global traffic manager with latency-based routing
2. **State Synchronization**: Redis Cluster with cross-region replication
3. **Database**: TimescaleDB with streaming replication
4. **Failover Time**: Targeted sub-minute automatic failover

### Failure Scenarios

| Scenario | Impact | Recovery |
|----------|--------|----------|
| Single service failure | None | Kubernetes auto-restart |
| AZ failure | Minimal | Traffic routed to other AZs |
| Region failure | Temporary | Automatic failover to DR region |
| Broker connection loss | Trading paused | Automatic reconnection |

## Network Architecture

### Internal Communication

```mermaid
flowchart TB
    subgraph VPC["Private Network (10.0.0.0/16)"]
        subgraph Private["Private Subnet (10.0.1.0/24)"]
            Core["bh-core"]
            Risk["bh-risk"]
            Guard["bh-guardian"]
            Mesh["Service Mesh\n(Istio)"]
        end

        subgraph Data["Data Subnet (10.0.2.0/24)"]
            TS["TimescaleDB"]
            PG["PostgreSQL"]
            Redis["Redis"]
        end

        Core <--> Mesh
        Risk <--> Mesh
        Guard <--> Mesh
        Mesh <--> Data
    end
```

### External Connectivity

| Connection | Protocol | Security |
|------------|----------|----------|
| Market Data APIs | REST/WebSocket | mTLS + API Key |
| Exchange Feeds | FIX 4.4 | VPN + mTLS |
| MT5 Broker | MT5 Protocol | Encrypted Channel |
| Cross-Region | Private Interconnect | Encrypted routing + TLS |

## Monitoring & Observability

```mermaid
flowchart LR
    subgraph Services["Services"]
        S1["bh-core"]
        S2["bh-risk"]
        S3["bh-quant"]
    end

    subgraph Monitoring["Monitoring Stack"]
        Prom["Prometheus"]
        Graf["Grafana"]
        Alert["AlertManager"]
    end

    subgraph Alerting["Alerting"]
        OnCall["On-Call System"]
        ChatOps["ChatOps"]
    end

    S1 --> Prom
    S2 --> Prom
    S3 --> Prom

    Prom --> Graf
    Prom --> Alert

    Alert --> OnCall
    Alert --> ChatOps
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
| Trading Core | Near-zero (synchronous) | Sub-minute target |
| Market Data | < 1s | < 60s |
| Analytics | < 5min | < 5min |
| Audit Logs | Near-zero | < 1min |

### Backup Strategy

- **Real-time**: Redis replication, PostgreSQL streaming
- **Hourly**: TimescaleDB continuous archiving
- **Daily**: Full database snapshots to S3
- **Weekly**: Cross-region backup verification

## Deployment Strategy

### Execution Layer Deployment

Our execution components (mt5_executor, mt5_tick) are deployed **as close as possible to our liquidity providers** to minimize latency:

```mermaid
flowchart LR
    subgraph DC["Core Data Center"]
        Core["Core Services"]
    end

    subgraph LP["Near Liquidity Provider"]
        Exec["mt5_executor\nmt5_tick"]
    end

    subgraph Broker["Broker Infrastructure"]
        MT5["MT5 Server"]
    end

    Core <-->|"Low single-digit ms (target)"| Exec
    Exec <-->|"Low single-digit ms (target)"| MT5
```

This architecture ensures:
- **Minimal network hops** between execution and broker
- **Single-digit ms total execution latency target** from signal to fill (venue dependent)
- **Geographic proximity** to gold market liquidity
