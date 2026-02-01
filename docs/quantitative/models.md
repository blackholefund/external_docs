# Quantitative Models

## Overview

BlackHole Fund employs a sophisticated suite of quantitative models for volatility forecasting, risk assessment, market regime detection, and signal generation. This document provides a high-level overview of the statistical methodologies employed.

## Volatility Forecasting Models

### GARCH Family

The Generalized Autoregressive Conditional Heteroskedasticity (GARCH) family of models forms the foundation of our volatility forecasting framework.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         GARCH Model Hierarchy                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│                              ┌─────────────┐                                   │
│                              │  GARCH(1,1) │                                   │
│                              │  (Baseline) │                                   │
│                              └──────┬──────┘                                   │
│                                     │                                           │
│           ┌─────────────────────────┼─────────────────────────┐                │
│           │                         │                         │                │
│           ▼                         ▼                         ▼                │
│    ┌─────────────┐          ┌─────────────┐          ┌─────────────┐          │
│    │   EGARCH    │          │   TGARCH    │          │   FIGARCH   │          │
│    │ (Leverage)  │          │(Asymmetric) │          │(Long Memory)│          │
│    └─────────────┘          └─────────────┘          └─────────────┘          │
│           │                         │                         │                │
│           └─────────────────────────┼─────────────────────────┘                │
│                                     │                                           │
│                                     ▼                                           │
│                          ┌───────────────────┐                                 │
│                          │  Regime-Switching │                                 │
│                          │      GARCH        │                                 │
│                          └───────────────────┘                                 │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### GARCH(1,1)

The standard GARCH(1,1) model captures volatility clustering:

$$\sigma_t^2 = \omega + \alpha \epsilon_{t-1}^2 + \beta \sigma_{t-1}^2$$

Where:
- $\sigma_t^2$ is the conditional variance at time t
- $\omega$ is the constant term
- $\alpha$ captures shock impact
- $\beta$ captures persistence
- $\alpha + \beta < 1$ ensures stationarity

**Use Case**: Baseline volatility estimation, model comparison benchmark

#### EGARCH (Exponential GARCH)

Captures the leverage effect (negative returns increase volatility more than positive returns):

$$\ln(\sigma_t^2) = \omega + \alpha \left( \frac{|\epsilon_{t-1}|}{\sigma_{t-1}} - \sqrt{\frac{2}{\pi}} \right) + \gamma \frac{\epsilon_{t-1}}{\sigma_{t-1}} + \beta \ln(\sigma_{t-1}^2)$$

**Use Case**: Gold markets where negative shocks (price drops) often increase volatility

#### TGARCH / GJR-GARCH

Threshold GARCH for asymmetric volatility response:

$$\sigma_t^2 = \omega + (\alpha + \gamma I_{t-1}) \epsilon_{t-1}^2 + \beta \sigma_{t-1}^2$$

Where $I_{t-1} = 1$ if $\epsilon_{t-1} < 0$ (negative shock indicator)

**Use Case**: Capturing news impact asymmetry

#### FIGARCH (Fractionally Integrated GARCH)

For long-memory volatility processes:

$$\sigma_t^2 = \omega + [1 - \beta(L) - \phi(L)(1-L)^d]\epsilon_t^2 + \beta(L)\sigma_t^2$$

Where $d \in (0, 0.5)$ is the fractional integration parameter

**Use Case**: Long-horizon forecasting, capturing persistent volatility regimes

### Realized Volatility

High-frequency estimators for intraday volatility measurement:

#### Standard Realized Volatility

$$RV_t = \sum_{i=1}^{n} r_{t,i}^2$$

#### Two-Scale Realized Volatility (TSRV)

Corrects for microstructure noise:

$$TSRV = RV^{(slow)} - \frac{\bar{n}}{n} RV^{(fast)}$$

#### Realized Kernel

Robust to noise and non-synchronous trading:

$$RK = \sum_{h=-H}^{H} k\left(\frac{h}{H}\right) \gamma_h$$

Where $k(\cdot)$ is a kernel function and $\gamma_h$ are autocovariances

## Time Series Models

### ARIMA Framework

Autoregressive Integrated Moving Average for price/return forecasting:

$$\phi(L)(1-L)^d X_t = \theta(L)\epsilon_t$$

**Model Selection**: Automated using AIC/BIC with stepwise search

### Vector Autoregression (VAR)

Multivariate time series for cross-market analysis:

$$Y_t = c + A_1 Y_{t-1} + A_2 Y_{t-2} + ... + A_p Y_{t-p} + \epsilon_t$$

**Variables Modeled**:
- Gold spot (XAU/USD)
- US Dollar Index (DXY)
- US Treasury yields (10Y)
- S&P 500 VIX
- Oil prices (WTI)

**Applications**:
- Granger causality testing
- Impulse response analysis
- Variance decomposition

### State Space Models

Kalman filtering for signal extraction:

**State Equation**: $\alpha_{t+1} = T_t \alpha_t + R_t \eta_t$

**Observation Equation**: $y_t = Z_t \alpha_t + \epsilon_t$

**Applications**:
- Trend/cycle decomposition
- Noise filtering
- Missing data handling

## Statistical Indicators

### Hurst Exponent

Measures long-term memory and mean-reversion tendency:

| Hurst Value | Interpretation | Trading Implication |
|-------------|----------------|---------------------|
| H < 0.5 | Mean-reverting | Counter-trend strategies |
| H = 0.5 | Random walk | No predictability |
| H > 0.5 | Trending | Momentum strategies |

**Calculation Method**: R/S (Rescaled Range) analysis with detrended fluctuation analysis (DFA) validation

### Cointegration Analysis

Tests for long-run equilibrium relationships:

#### Engle-Granger Two-Step Method

1. Estimate cointegrating regression: $y_t = \alpha + \beta x_t + u_t$
2. Test residuals for stationarity (ADF test)

#### Johansen Procedure

For multiple cointegrating relationships in multivariate systems

**Applications**:
- Gold vs. Silver spread trading
- Gold vs. mining stocks
- Cross-currency relationships

### Half-Life of Mean Reversion

Estimates time for spread to revert halfway to mean:

$$\text{Half-Life} = -\frac{\ln(2)}{\lambda}$$

Where $\lambda$ is estimated from: $\Delta S_t = \lambda S_{t-1} + \epsilon_t$

## Monte Carlo Simulation

### Geometric Brownian Motion (GBM)

Standard price path simulation:

$$dS_t = \mu S_t dt + \sigma S_t dW_t$$

**Discrete approximation**:
$$S_{t+\Delta t} = S_t \exp\left[\left(\mu - \frac{\sigma^2}{2}\right)\Delta t + \sigma\sqrt{\Delta t}Z\right]$$

### Jump-Diffusion (Merton Model)

Incorporating sudden price jumps:

$$dS_t = \mu S_t dt + \sigma S_t dW_t + S_t dJ_t$$

Where $J_t$ is a compound Poisson process

**Parameters**:
- Jump intensity ($\lambda$): Average jumps per year
- Jump size distribution: Log-normal with mean $\mu_J$ and std $\sigma_J$

### Stochastic Volatility

Heston model for realistic volatility dynamics:

$$dS_t = \mu S_t dt + \sqrt{v_t} S_t dW_t^S$$
$$dv_t = \kappa(\theta - v_t)dt + \xi\sqrt{v_t}dW_t^v$$

With correlation $\rho$ between $W^S$ and $W^v$

### Simulation Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| Simulations | 10,000 | Number of paths |
| Horizon | 22 days | ~1 trading month |
| Time step | 1 day | Daily granularity |
| Seed | Fixed | Reproducibility |

## Regime Detection

### Hidden Markov Model (HMM)

Probabilistic regime identification:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         Market Regime States                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│    ┌─────────────────┐                              ┌─────────────────┐        │
│    │   LOW           │         0.05                 │   HIGH          │        │
│    │   VOLATILITY    │◄────────────────────────────►│   VOLATILITY    │        │
│    │                 │                              │                 │        │
│    │   μ = 0.0002    │         0.15                 │   μ = -0.0001   │        │
│    │   σ = 0.008     │                              │   σ = 0.025     │        │
│    │                 │                              │                 │        │
│    │   Persistence:  │                              │   Persistence:  │        │
│    │   0.95          │                              │   0.85          │        │
│    └────────┬────────┘                              └────────┬────────┘        │
│             │                                                │                  │
│             │                   0.10                         │                  │
│             │         ┌─────────────────────┐               │                  │
│             └────────►│     TRANSITION      │◄──────────────┘                  │
│                       │                     │                                   │
│                       │   μ = 0.0000        │                                   │
│                       │   σ = 0.015         │                                   │
│                       │                     │                                   │
│                       │   Persistence: 0.70 │                                   │
│                       └─────────────────────┘                                   │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Regime Characteristics

| Regime | Volatility | Avg Return | Duration | Trading Approach |
|--------|------------|------------|----------|------------------|
| Low Vol | < 10% ann | +0.02%/day | ~50 days | Full position |
| Normal | 10-20% ann | 0%/day | ~30 days | Normal position |
| High Vol | > 20% ann | -0.01%/day | ~15 days | Reduced position |
| Crisis | > 35% ann | Variable | ~5 days | Minimal/hedge |

### Regime-Based Adjustments

```
Position Sizing by Regime:
├── Low Volatility:     1.25x base position
├── Normal:             1.00x base position
├── High Volatility:    0.50x base position
└── Crisis:             0.25x base position (or flat)

Stop-Loss Adjustment:
├── Low Volatility:     1.5 ATR
├── Normal:             2.0 ATR
├── High Volatility:    3.0 ATR
└── Crisis:             4.0 ATR or time-based exit
```

## Risk Metrics

### Value at Risk (VaR)

| Method | Confidence | Horizon | Use Case |
|--------|------------|---------|----------|
| Historical | 95%, 99% | 1-day | Regulatory |
| Parametric | 95%, 99% | 1-day | Quick estimate |
| Monte Carlo | 95%, 99% | 1-22 days | Comprehensive |

### Conditional VaR (CVaR) / Expected Shortfall

$$CVaR_\alpha = E[Loss | Loss > VaR_\alpha]$$

**Advantages over VaR**:
- Coherent risk measure (subadditive)
- Captures tail risk magnitude
- Basel III/FRTB compliant

### Stress Testing Scenarios

| Scenario | Gold Move | VIX | DXY | Probability |
|----------|-----------|-----|-----|-------------|
| Flash Crash | -5% | +50% | +2% | Tail event |
| Fed Surprise | ±3% | +30% | ±3% | Low |
| Geopolitical | +4% | +40% | -1% | Medium |
| Liquidity Crisis | -8% | +100% | +5% | Tail event |

## Model Validation

### Backtesting

All models undergo rigorous backtesting:

1. **In-sample fit**: Parameter estimation on training data
2. **Out-of-sample testing**: Forecast evaluation on held-out data
3. **Rolling window**: Expanding/rolling estimation windows
4. **Statistical tests**: Kupiec, Christoffersen, DQ tests for VaR

### Model Confidence

| Metric | Target | Current |
|--------|--------|---------|
| VaR violations (95%) | 5% | 4.8% |
| VaR violations (99%) | 1% | 1.1% |
| Volatility forecast MSE | < 0.0001 | 0.00008 |
| Regime accuracy | > 70% | 73% |

## Update Frequency

| Model | Refit Frequency | Forecast Update |
|-------|-----------------|-----------------|
| GARCH | Daily | Every 5 minutes |
| ARIMA | 4 hours | Hourly |
| HMM Regime | Daily | Hourly |
| Monte Carlo | 4 hours | 4 hours |
| Cointegration | Weekly | Daily |

## References

This quantitative framework draws from established academic and industry research:

- Bollerslev, T. (1986). Generalized Autoregressive Conditional Heteroskedasticity
- Engle, R. F., & Granger, C. W. (1987). Co-integration and Error Correction
- Heston, S. L. (1993). A Closed-Form Solution for Options with Stochastic Volatility
- Mandelbrot, B. B. (1963). The Variation of Certain Speculative Prices
- Merton, R. C. (1976). Option Pricing When Underlying Stock Returns Are Discontinuous
