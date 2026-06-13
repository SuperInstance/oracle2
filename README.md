# Oracle2 — Predictive Analytics and Decision Engine

**Oracle2** is a Rust library for **probabilistic forecasting** — producing reasoned estimates of future events given structured input. Where a classifier answers "what is this?", an oracle answers "what will happen?" It transforms historical observations into probability distributions over future outcomes, enabling agents to make decisions under uncertainty rather than reacting to what already occurred.

## Why It Matters

Every agent in a fleet makes decisions: should I process this task now or defer? Route to GPU A or GPU B? Scale up or stay flat? Each decision depends on a prediction: what will the workload be in 5 minutes? Will GPU A be free? Is this a temporary spike or a sustained trend?

Without a forecasting engine, agents default to **reactive control**: they see a spike, they scale up; they see a lull, they scale down. This is always late — by the time the scale-up takes effect, the spike is over. Oracle2 provides **predictive control**: it forecasts the spike 5 minutes ahead, initiates the scale-up early, and the capacity arrives just in time.

The engine is designed as a library, not a service. Downstream agents call into it synchronously: `oracle.forecast(&observations, horizon=Duration::minutes(5))`. No network hop, no serialization overhead — the prediction happens in-process, in microseconds.

## How It Works

### Forecasting Methods

Oracle2 is designed to support multiple forecasting methods, each with different trade-offs:

**1. Moving Average (MA):**

```
ŷ(t+h) = (1/w) · Σᵢ₌₀^{w-1} y(t-i)
```

Weight w controls smoothness. O(w) per forecast. Best for stationary signals.

**2. Exponential Smoothing (ETS):**

```
s(t) = α · y(t) + (1-α) · s(t-1)
ŷ(t+h) = s(t)
```

Single parameter α ∈ (0,1) controls how quickly old observations are forgotten. O(1) per update. Equivalent to an IIR filter with pole at (1-α). The half-life of an observation is `log(0.5)/log(1-α)`.

**3. Bayesian Update:**

```
posterior ∝ likelihood × prior
```

For conjugate distributions (Beta-Binomial, Gaussian-Gaussian), updates are closed-form:

```
μ_posterior = (μ_prior/σ²_prior + Σyᵢ/σ²_likelihood) / (1/σ²_prior + n/σ²_likelihood)
σ²_posterior = 1 / (1/σ²_prior + n/σ²_likelihood)
```

O(1) per update. Provides **calibrated uncertainty** — not just a point estimate, but a confidence interval.

**4. ARIMA (Autoregressive Integrated Moving Average):**

```
φ(B)(1-B)^d y(t) = θ(B)ε(t)
```

Where B is the backshift operator, φ is the AR polynomial, θ is the MA polynomial. The model captures trend (d), autocorrelation (φ), and shock decay (θ). Fitting requires O(p+q) parameters via maximum likelihood.

### Uncertainty Quantification

A point forecast is useless without a confidence interval. Oracle2 represents forecasts as distributions:

```
Forecast {
    point: 0.73,           // expected value
    p05: 0.61,             // 5th percentile
    p95: 0.85,             // 95th percentile
    method: "bayesian",    // source
}
```

The 90% credible interval [0.61, 0.85] tells the caller not just "0.73" but "probably between 0.61 and 0.85." An agent can then make risk-aware decisions: if the lower bound is still above threshold, act confidently; if the interval straddles threshold, hedge.

### Decision Theory Integration

Forecasts feed into **expected value calculations**:

```
E[action] = Σ_outcome P(outcome) · Utility(outcome, action)
```

The optimal action maximizes expected utility. Oracle2 provides the P(outcome) — the probability distribution over future states. The calling agent provides the Utility function.

**Big-O:** For K possible actions and O outcomes, the decision is O(K × O). Oracle2's contribution is making O tractable by producing a compact distribution rather than enumerating all possible futures.

### Comparison to Alternatives

| Approach | Latency | Uncertainty | Trend | Seasonality |
|---|---|---|---|---|
| Moving average | O(1) | No | No | No |
| Exponential smoothing | O(1) | No | No | No |
| ARIMA | O(n) fit | Via residuals | Yes | Yes |
| Bayesian conjugate | O(1) | **Yes (calibrated)** | Partial | No |
| Neural (Prophet/N-BEATS) | O(n) train | Partial | Yes | Yes |

Oracle2 targets the **microsecond regime**: forecasts that are "good enough" to inform real-time agent decisions, produced faster than a network round-trip.

## Quick Start

```rust
use oracle2::stub;

fn main() {
    println!("{}", stub::hello());
    // "hello from oracle2"
}
```

The crate is scaffolded. Planned API:

```rust
pub struct Oracle { /* ... */ }

impl Oracle {
    /// Forecast the next value given recent observations
    pub fn forecast(&self, observations: &[f64], horizon: usize) -> Forecast;

    /// Update the model with a new observation (online learning)
    pub fn observe(&mut self, value: f64);

    /// Compute expected utility for each action
    pub fn decide(&self, actions: &[Action], utility: impl Fn(&State, &Action) -> f64) -> usize;
}

pub struct Forecast {
    pub point: f64,
    pub p05: f64,
    pub p95: f64,
    pub method: ForecastMethod,
}
```

## API

### `stub::hello() -> &'static str`

Placeholder returning `"hello from oracle2"`. The forecasting engine is under development.

### Planned: `Oracle`

The main handle. `forecast()` produces a distribution over future values. `observe()` updates the internal model online (streaming). `decide()` integrates forecasting with decision theory to select optimal actions.

## Architecture Notes

Oracle2 provides the **predictive cognition layer** for the SuperInstance constellation. In the conservation law **γ + η = C**, prediction enables **proactive γ allocation**: if Oracle2 forecasts a spike in demand, the fleet pre-allocates γ (compute capacity) before the spike arrives, rather than reactively scaling after. This transforms the fleet from reactive to anticipatory.

Oracle2 is the successor to Oracle1 (the first-generation forecasting prototype). Where Oracle1 ran as a PLATO room service, Oracle2 is an in-process library — forecasts happen in the agent's own memory space, eliminating the network hop. See the [SuperInstance Architecture](https://github.com/SuperInstance/SuperInstance/blob/main/ARCHITECTURE.md).

## References

1. Hyndman, R. J. & Athanasopoulos, G. *Forecasting: Principles and Practice* (3rd ed.) — comprehensive textbook, [otexts.com/fpp3](https://otexts.com/fpp3/)
2. Bishop, C. M. *Pattern Recognition and Machine Learning* — Bayesian decision theory, Chapter 1
3. Box, G. E. P. et al. *Time Series Analysis: Forecasting and Control* — ARIMA foundations

## License

MIT
