# Communication Protocols

## Overview

BlackHole Fund's infrastructure utilizes a combination of communication protocols optimized for different use cases: low-latency trading operations, reliable message delivery, and efficient data streaming.

## Protocol Matrix

```
┌────────────────────┬───────────────────┬──────────────────┬─────────────────┐
│     Service        │   Protocol        │   Pattern        │   Latency       │
├────────────────────┼───────────────────┼──────────────────┼─────────────────┤
│ mt5_tick           │ ZeroMQ (PUB/SUB)  │ Publish-Subscribe│ < 100μs         │
│ mt5_executor       │ ZeroMQ (REQ/REP)  │ Request-Reply    │ < 500μs         │
│ bh-risk            │ gRPC              │ Unary/Stream     │ < 5ms           │
│ bh-guardian        │ gRPC + ZeroMQ     │ Hybrid           │ < 1ms           │
│ bh-quant-engine    │ Redis Streams     │ Consumer Group   │ < 50ms          │
│ bh-core            │ gRPC              │ Bidirectional    │ < 10ms          │
│ bh-market-gateway  │ FIX + WebSocket   │ Various          │ < 20ms          │
└────────────────────┴───────────────────┴──────────────────┴─────────────────┘
```

## Service Communication Diagram

```
                           SYNCHRONOUS COMMUNICATION (gRPC)
═══════════════════════════════════════════════════════════════════════════════════

         ┌─────────────────────────────────────────────────────────────────┐
         │                                                                 │
         │    ┌──────────┐        gRPC          ┌──────────┐              │
         │    │ bh-core  │◄────────────────────►│ bh-risk  │              │
         │    └────┬─────┘                      └────┬─────┘              │
         │         │                                 │                     │
         │         │ gRPC                            │ gRPC                │
         │         │                                 │                     │
         │         ▼                                 ▼                     │
         │    ┌──────────┐        gRPC          ┌──────────┐              │
         │    │bh-guard  │◄────────────────────►│bh-quant  │              │
         │    └──────────┘                      └──────────┘              │
         │                                                                 │
         └─────────────────────────────────────────────────────────────────┘


                        ASYNCHRONOUS COMMUNICATION (ZeroMQ/Redis)
═══════════════════════════════════════════════════════════════════════════════════

    ┌─────────────┐                                          ┌─────────────┐
    │  mt5_tick   │──────────────────┐                       │ bh-quant    │
    │   [PUB]     │                  │                       │   [SUB]     │
    └─────────────┘                  │                       └──────▲──────┘
                                     │                              │
                                     ▼                              │
                              ┌─────────────┐                       │
                              │   ZeroMQ    │───────────────────────┤
                              │   Proxy     │                       │
                              └─────────────┘                       │
                                     │                              │
                                     │                              │
    ┌─────────────┐                  │                       ┌──────┴──────┐
    │  bh-risk    │◄─────────────────┘                       │  bh-core    │
    │   [SUB]     │                                          │   [SUB]     │
    └─────────────┘                                          └─────────────┘


                           EVENT STREAMING (Redis Streams)
═══════════════════════════════════════════════════════════════════════════════════

    ┌─────────────┐
    │bh-market-gw │─────┐                               ┌─────────────────────────┐
    └─────────────┘     │                               │    Consumer Groups      │
                        │                               │                         │
    ┌─────────────┐     │      ┌─────────────┐         │  ┌─────────────────┐    │
    │  mt5_tick   │─────┼─────►│   Redis     │────────►│  │ cg:risk-engine  │────┼───► bh-risk
    └─────────────┘     │      │   Stream    │         │  └─────────────────┘    │
                        │      │             │         │                         │
    ┌─────────────┐     │      │  market:    │         │  ┌─────────────────┐    │
    │  bh-quant   │─────┘      │  ticks      │────────►│  │ cg:quant-engine │────┼───► bh-quant
    └─────────────┘            └─────────────┘         │  └─────────────────┘    │
                                                       │                         │
                                                       │  ┌─────────────────┐    │
                                                       │  │ cg:analytics    │────┼───► Analytics
                                                       │  └─────────────────┘    │
                                                       └─────────────────────────┘
```

## Protocol Details

### 1. ZeroMQ (High-Performance Messaging)

Used for ultra-low-latency communication in the execution path.

#### Tick Data Distribution (PUB/SUB)

```
Publisher: mt5_tick
Socket Type: PUB
Endpoint: tcp://*:5555

Message Format:
┌──────────────────────────────────────────────────────────────┐
│ Header (8 bytes)  │ Symbol (6 bytes) │ Payload (Variable)    │
├───────────────────┼──────────────────┼───────────────────────┤
│ MSG_TYPE: TICK    │ XAUUSD           │ Bid, Ask, Timestamp   │
│ SEQ_NUM: uint64   │                  │ Volume, Spread        │
└───────────────────┴──────────────────┴───────────────────────┘

Topic Filtering:
- TICK.XAUUSD      (Gold spot ticks)
- TICK.XAUUSD.1M   (1-minute aggregates)
- TICK.XAUUSD.5M   (5-minute aggregates)
```

#### Order Execution (REQ/REP)

```
Service: mt5_executor
Socket Type: REP
Endpoint: tcp://*:5556

Request Message:
┌─────────────────────────────────────────────────────────────────────┐
│ Field           │ Type      │ Description                          │
├─────────────────┼───────────┼──────────────────────────────────────┤
│ request_id      │ uuid      │ Unique request identifier            │
│ action          │ enum      │ BUY, SELL, MODIFY, CANCEL            │
│ symbol          │ string    │ Trading symbol (e.g., XAUUSD)        │
│ volume          │ double    │ Position size in lots                │
│ price           │ double    │ Requested price (0 for market)       │
│ sl              │ double    │ Stop loss price                      │
│ tp              │ double    │ Take profit price                    │
│ magic           │ int64     │ Expert Advisor identifier            │
│ comment         │ string    │ Order comment                        │
└─────────────────┴───────────┴──────────────────────────────────────┘

Response Message:
┌─────────────────────────────────────────────────────────────────────┐
│ Field           │ Type      │ Description                          │
├─────────────────┼───────────┼──────────────────────────────────────┤
│ request_id      │ uuid      │ Original request identifier          │
│ status          │ enum      │ SUCCESS, REJECTED, PARTIAL, ERROR    │
│ order_id        │ int64     │ Broker order identifier              │
│ filled_price    │ double    │ Actual execution price               │
│ filled_volume   │ double    │ Actual filled volume                 │
│ latency_us      │ int64     │ Execution latency in microseconds    │
│ error_code      │ int32     │ Error code (if applicable)           │
│ error_message   │ string    │ Error description                    │
└─────────────────┴───────────┴──────────────────────────────────────┘
```

### 2. gRPC (Service-to-Service Communication)

Used for reliable, typed communication between core services.

#### Risk Service API

```protobuf
// risk_service.proto

service RiskService {
  // Evaluate order before execution
  rpc EvaluateOrder(OrderRequest) returns (RiskDecision);

  // Stream real-time position updates
  rpc StreamPositions(PositionFilter) returns (stream PositionUpdate);

  // Get current risk metrics
  rpc GetRiskMetrics(MetricsRequest) returns (RiskMetrics);

  // Update risk parameters
  rpc UpdateParameters(RiskParameters) returns (UpdateResponse);
}

message RiskDecision {
  bool approved = 1;
  double adjusted_volume = 2;
  double max_loss = 3;
  string reason = 4;
  RiskLevel risk_level = 5;
}

enum RiskLevel {
  LOW = 0;
  MEDIUM = 1;
  HIGH = 2;
  CRITICAL = 3;
}
```

#### Guardian Service API (Circuit Breaker)

```protobuf
// guardian_service.proto

service GuardianService {
  // Check if trading is allowed
  rpc CheckTradingStatus(StatusRequest) returns (TradingStatus);

  // Register for circuit breaker events
  rpc SubscribeEvents(EventFilter) returns (stream CircuitEvent);

  // Manual system control
  rpc SetSystemState(StateCommand) returns (StateResponse);
}

message TradingStatus {
  bool trading_allowed = 1;
  double current_daily_pnl = 2;
  double daily_loss_limit = 3;
  double remaining_capacity = 4;
  SystemState state = 5;
}

enum SystemState {
  ACTIVE = 0;
  REDUCED = 1;    // Reduced position sizes
  CLOSING = 2;    // Closing positions only
  HALTED = 3;     // All trading stopped
}
```

### 3. Redis Streams (Event Streaming)

Used for durable, distributed event streaming with consumer groups.

#### Stream Configuration

```
Streams:
├── market:ticks:xauusd          # Real-time tick data
├── market:bars:xauusd:1m        # 1-minute OHLCV bars
├── orders:requests              # Order requests
├── orders:executions            # Execution reports
├── risk:events                  # Risk events and alerts
├── system:health                # Health check events
└── analytics:signals            # Trading signals

Consumer Groups:
├── cg:risk-engine              # Risk processing
├── cg:quant-engine             # Quantitative analysis
├── cg:order-router             # Order routing
├── cg:analytics                # Analytics and reporting
└── cg:audit                    # Audit logging
```

#### Message Schema Example

```json
{
  "stream": "market:ticks:xauusd",
  "id": "1706745600000-0",
  "fields": {
    "timestamp": "1706745600000",
    "bid": "2035.45",
    "ask": "2035.65",
    "bid_volume": "100",
    "ask_volume": "150",
    "spread": "0.20",
    "source": "bloomberg"
  }
}
```

### 4. FIX Protocol (Exchange Connectivity)

Used for connecting to exchanges and market data providers.

```
FIX 4.4 Configuration:
├── Session: bh-gateway -> Exchange
├── HeartBtInt: 30 seconds
├── ResetOnLogon: Y
├── ResetOnLogout: Y
└── ResetOnDisconnect: Y

Message Types Supported:
├── D  - New Order Single
├── F  - Order Cancel Request
├── G  - Order Cancel/Replace Request
├── 8  - Execution Report
├── 9  - Order Cancel Reject
├── V  - Market Data Request
├── W  - Market Data Snapshot
└── X  - Market Data Incremental Refresh
```

## Message Serialization

### Protocol Buffers (gRPC Services)

All gRPC services use Protocol Buffers for efficient serialization:

- **Compact**: 3-10x smaller than JSON
- **Fast**: 20-100x faster parsing
- **Typed**: Strong schema enforcement
- **Versioned**: Backward compatible evolution

### MessagePack (ZeroMQ Messages)

High-frequency messages use MessagePack:

- **Binary**: Efficient encoding
- **Schema-less**: Flexible structure
- **Fast**: Minimal parsing overhead

## Error Handling

### Retry Policies

| Protocol | Retry Count | Backoff | Timeout |
|----------|-------------|---------|---------|
| gRPC | 3 | Exponential | 5s |
| ZeroMQ | 0 | N/A | 100ms |
| Redis | 5 | Linear | 1s |
| FIX | Session-level | N/A | 30s |

### Circuit Breaker Pattern

```
┌─────────────┐     Failure      ┌─────────────┐
│   CLOSED    │────────────────►│    OPEN     │
│  (Normal)   │                 │  (Blocking) │
└──────┬──────┘                 └──────┬──────┘
       │                               │
       │ Success                       │ Timeout
       │                               │
       │         ┌─────────────┐       │
       └─────────│ HALF-OPEN   │◄──────┘
                 │  (Testing)  │
                 └─────────────┘
                       │
                       │ Success/Failure
                       ▼
                 CLOSED/OPEN
```

## Security

### Transport Security

| Connection Type | Encryption | Authentication |
|-----------------|------------|----------------|
| Internal gRPC | mTLS | Service certificates |
| Internal ZeroMQ | CurveZMQ | Public key pairs |
| External FIX | TLS 1.3 | Client certificate |
| Redis | TLS | Password + ACL |

### Message Signing

Critical messages (orders, risk decisions) include cryptographic signatures:

```
┌─────────────────────────────────────────┐
│ Message Envelope                        │
├─────────────────────────────────────────┤
│ Header                                  │
│   ├── message_id: uuid                  │
│   ├── timestamp: int64                  │
│   ├── source_service: string            │
│   └── signature_algorithm: ED25519      │
├─────────────────────────────────────────┤
│ Payload (serialized message)            │
├─────────────────────────────────────────┤
│ Signature (64 bytes)                    │
└─────────────────────────────────────────┘
```

## Monitoring

### Latency Tracking

Every message includes timing information for end-to-end latency monitoring:

```
Trace Headers:
- x-trace-id: Distributed trace identifier
- x-span-id: Current span identifier
- x-parent-span-id: Parent span identifier
- x-timestamp-origin: Origin timestamp (nanoseconds)
```

### Metrics Collected

- Message throughput (messages/second)
- Latency percentiles (p50, p95, p99, p99.9)
- Error rates by type
- Queue depths and backlogs
