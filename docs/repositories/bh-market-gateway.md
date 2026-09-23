# bh-market-gateway

## Overview

**bh-market-gateway** is the market data connector service that interfaces with external data providers, exchanges, and news feeds. Written in Go with performance-critical components in Rust, it normalizes data from multiple sources into a unified format for consumption by the other services of the Genese Capital (formerly BlackHole Capital) infrastructure. The specific data sources used by the decision engine are proprietary.

**Note:** Provider names are placeholders (Provider A/Provider B). Replace with contracted vendors and adjust specs/SLAs accordingly.

## Technical Specifications

| Attribute | Value |
|-----------|-------|
| **Language** | Go 1.22+ (Core), Rust (FIX Engine) |
| **Protocols** | FIX 4.4, WebSocket, REST, gRPC |
| **Dependencies** | quickfix, tokio, redis, kafka |
| **Supported Sources** | Provider A, Provider B, Exchange feeds |

## Architecture

```mermaid
flowchart TB
    subgraph gateway["bh-market-gateway"]
        subgraph connectors["Connector Layer"]
            ProviderA["Provider A Market Data API"]
            ProviderB["Provider B Market Data API"]
            Exchange["Exchange FIX Feeds"]
            NewsAPI["News API"]
        end

        subgraph normalization["Normalization Engine"]
            subgraph priceNorm["Price Normalizer"]
                decimal["Decimal prec"]
                currency["Currency conv"]
                unit["Unit standard"]
            end
            subgraph symbolMap["Symbol Mapper"]
                crossSource["Cross-source"]
                aliasMapping["Alias mapping"]
                validation["Validation"]
            end
            subgraph timeSync["Time Sync"]
                ntp["NTP sync"]
                latencyComp["Latency comp"]
                timestamp["Timestamp"]
            end
        end

        subgraph aggregation["Aggregation & Distribution"]
            subgraph bestPrice["Best Price Aggregator"]
                multiSource["Multi-source"]
                weightedAvg["Weighted avg"]
                arbitrageDet["Arbitrage det"]
            end
            subgraph conflation["Conflation Engine"]
                rateLimiting["Rate limiting"]
                sampling["Sampling"]
                batching["Batching"]
            end
            subgraph fanout["Fan-out Publisher"]
                ZeroMQ["ZeroMQ"]
                RedisStreams["Redis Streams"]
                Kafka["Kafka"]
            end
        end

        connectors --> normalization
        normalization --> aggregation
    end
```

## Data Sources

### 1. Provider A Market Data API

Primary institutional data feed for real-time and reference data.

| Data Type | Update Frequency | Latency |
|-----------|------------------|---------|
| Spot prices | Real-time | Low-latency target |
| Reference data | Daily | N/A |
| Corporate actions | Event-driven | < 1min |
| Economic indicators | Event-driven | < 1s |

### 2. Provider B Market Data API

Backup data feed and additional market coverage.

### 3. Economic Calendar & News API

The economic calendar and news are received via API and drive the news filter, which blocks entries around high-impact events.

## Connector Implementations

### Provider A Connector

```go
type ProviderAConnector struct {
    session    *providerapi.Session
    subscriptions map[string]*Subscription
    normalizer *PriceNormalizer
}

func (c *ProviderAConnector) Subscribe(symbols []string) error {
    for _, symbol := range symbols {
        sub := &providerapi.Subscription{
            Security: symbol,
            Fields:   []string{"BID", "ASK", "LAST_PRICE", "VOLUME"},
        }
        c.session.Subscribe(sub)
    }
    return nil
}

func (c *ProviderAConnector) handleEvent(event *providerapi.Event) {
    switch event.EventType {
    case providerapi.SUBSCRIPTION_DATA:
        c.processMarketData(event)
    case providerapi.SUBSCRIPTION_STATUS:
        c.handleStatusChange(event)
    }
}
```

### FIX Engine (Rust)

High-performance FIX protocol handler for exchange connectivity.

```rust
pub struct FixEngine {
    sessions: HashMap<String, FixSession>,
    message_store: Arc<MessageStore>,
    router: MessageRouter,
}

impl FixEngine {
    pub async fn connect(&mut self, config: &SessionConfig) -> Result<()> {
        let session = FixSession::new(config)?;
        session.logon().await?;
        self.sessions.insert(config.session_id.clone(), session);
        Ok(())
    }

    pub async fn handle_message(&self, msg: FixMessage) -> Result<()> {
        match msg.msg_type() {
            MsgType::MarketDataSnapshotFullRefresh => {
                self.process_snapshot(msg).await
            }
            MsgType::MarketDataIncrementalRefresh => {
                self.process_incremental(msg).await
            }
            _ => Ok(())
        }
    }
}
```

## Symbol Mapping

Unified symbol mapping across all data sources:

```yaml
# symbol_mapping.yaml
XAUUSD:
  provider_a: "XAUUSD-SPOT"
  provider_b: "XAUUSD"
  internal: "XAUUSD"
  description: "Gold Spot USD"
  tick_size: 0.01
  lot_size: 1.0
  currency: "USD"
```

## Data Normalization

### Price Normalization

```go
type NormalizedPrice struct {
    Symbol      string    `json:"symbol"`
    Bid         float64   `json:"bid"`
    Ask         float64   `json:"ask"`
    Mid         float64   `json:"mid"`
    Spread      float64   `json:"spread"`
    BidSize     float64   `json:"bid_size"`
    AskSize     float64   `json:"ask_size"`
    Timestamp   time.Time `json:"timestamp"`
    Source      string    `json:"source"`
    SourceTime  time.Time `json:"source_time"`
    Latency     int64     `json:"latency_us"`
}

func (n *PriceNormalizer) Normalize(raw *RawPrice) *NormalizedPrice {
    return &NormalizedPrice{
        Symbol:     n.symbolMapper.ToInternal(raw.Symbol, raw.Source),
        Bid:        n.roundPrice(raw.Bid),
        Ask:        n.roundPrice(raw.Ask),
        Mid:        (raw.Bid + raw.Ask) / 2,
        Spread:     raw.Ask - raw.Bid,
        BidSize:    raw.BidSize,
        AskSize:    raw.AskSize,
        Timestamp:  time.Now().UTC(),
        Source:     raw.Source,
        SourceTime: raw.SourceTime,
        Latency:    time.Since(raw.SourceTime).Microseconds(),
    }
}
```

### Best Price Aggregation

```go
type BestPriceAggregator struct {
    prices    map[string]map[string]*NormalizedPrice // symbol -> source -> price
    weights   map[string]float64                      // source weights
    staleTime time.Duration
}

func (a *BestPriceAggregator) GetBestPrice(symbol string) *AggregatedPrice {
    sources := a.prices[symbol]

    var bestBid, bestAsk float64
    var totalWeight float64

    for source, price := range sources {
        if time.Since(price.Timestamp) > a.staleTime {
            continue
        }

        weight := a.weights[source]

        // Best bid is highest, best ask is lowest
        if price.Bid > bestBid {
            bestBid = price.Bid
        }
        if bestAsk == 0 || price.Ask < bestAsk {
            bestAsk = price.Ask
        }

        totalWeight += weight
    }

    return &AggregatedPrice{
        Symbol:   symbol,
        BestBid:  bestBid,
        BestAsk:  bestAsk,
        Sources:  len(sources),
    }
}
```

## Configuration

```yaml
# bh-market-gateway.yaml
server:
  grpc_port: 50054
  metrics_port: 9095

connectors:
  provider_a:
    enabled: true
    host: "provider-a-api.internal"
    port: 8194
    app_name: "blackhole_gateway"
    symbols:
      - "XAUUSD-SPOT"
    reconnect_interval: 5s

  provider_b:
    enabled: true
    host: "provider-b-feed.internal"
    port: 14002
    app_id: ${PROVIDER_B_APP_ID}
    symbols:
      - "XAUUSD"

  news:
    enabled: true
    economic_calendar:
      enabled: true

normalization:
  decimal_precision: 5
  stale_threshold: 5s
  symbol_mapping_file: "/etc/gateway/symbol_mapping.yaml"

aggregation:
  source_weights:
    provider_a: 1.0
    provider_b: 0.8
  update_interval: 10ms

distribution:
  zeromq:
    enabled: true
    endpoint: "tcp://*:5560"

  redis:
    enabled: true
    host: "redis.internal"
    stream_prefix: "market:"

  kafka:
    enabled: true
    brokers: ["kafka:9092"]
    topic: "market-data"

monitoring:
  latency_buckets: [0.001, 0.005, 0.01, 0.05, 0.1]
```

## Output Streams

### Real-Time Prices (ZeroMQ)

```
Topics:
├── PRICE.XAUUSD           Raw prices from all sources
├── PRICE.XAUUSD.BEST      Aggregated best price
└── PRICE.XAUUSD.PROVIDER_A Single source
```

### Redis Streams

```
Streams:
├── market:prices:xauusd     Real-time price updates
├── market:depth:xauusd      Order book depth
├── market:news:gold         News events
└── market:calendar          Economic calendar
```

### Event Schema

```json
{
  "stream": "market:prices:xauusd",
  "event": {
    "type": "price_update",
    "symbol": "XAUUSD",
    "bid": 2035.45,
    "ask": 2035.65,
    "mid": 2035.55,
    "spread": 0.20,
    "timestamp": "2024-01-15T10:30:00.123456Z",
    "source": "provider_a",
    "source_latency_us": 1234,
    "sequence": 123456789
  }
}
```

## Monitoring

### Prometheus Metrics

```
# Connector metrics
bh_gateway_connected{source="provider_a|provider_b"}
bh_gateway_messages_received_total{source="...",type="price|news|status"}
bh_gateway_latency_seconds{source="...",quantile="0.5|0.95|0.99"}

# Data quality metrics
bh_gateway_stale_prices_total{source="...",symbol="..."}
bh_gateway_gaps_detected_total{source="..."}
bh_gateway_price_variance{symbol="..."}  # Cross-source variance

# Distribution metrics
bh_gateway_published_total{channel="zmq|redis|kafka"}
bh_gateway_publish_latency_seconds{channel="..."}
```

## Failover & Anomaly Handling

Market data is automatically cross-validated between sources. Depending on the anomaly, the system discards the tick, freezes execution or pauses trading. **A pause requires manual release.**

```mermaid
flowchart TD
    Tick["Incoming tick"] --> Validate{"Cross-source\nvalidation"}
    Validate -->|"OK"| Publish["Publish"]
    Validate -->|"Anomaly"| Response{"Severity"}
    Response --> Discard["Discard tick"]
    Response --> Freeze["Freeze execution"]
    Response --> Pause["Pause trading\n(manual release required)"]
```

## Security

### API Authentication

```yaml
# Per-source authentication
provider_a:
  auth_type: "certificate"
  cert_file: "/etc/ssl/provider-a.pem"
  key_file: "/etc/ssl/provider-a.key"

provider_b:
  auth_type: "oauth2"
  token_url: "https://auth.provider-b.com/token"
  client_id: ${PROVIDER_B_CLIENT_ID}
  client_secret: ${PROVIDER_B_CLIENT_SECRET}

fix:
  auth_type: "fix_login"
  username: ${FIX_USERNAME}
  password: ${FIX_PASSWORD}
```

### Data Validation

All incoming data is validated before distribution:
- Price sanity checks (within expected range)
- Timestamp validation (not in future, not too stale)
- Sequence number tracking (gap detection)
- Cross-source consistency checks
