# Rates Week: Wednesday

## 1. Core Objective

Wednesday focuses on constructing and validating a USD SOFR term structure from tradable market instruments.

$$
\text{Market Quotes}
\rightarrow
\text{Rate Helpers}
\rightarrow
\text{Bootstrap}
\rightarrow
\text{SOFR Curve}
\rightarrow
\text{Repricing and Diagnostics}
$$

The curve is a calibrated pricing system, not a line fitted directly through quoted rates.

---

## 2. Bootstrap Framework

Given market instruments $I_1,\ldots,I_n$ with quotes $q_i^{mkt}$, bootstrap solves for curve nodes such that:

$$
q_i^{model}(\mathcal C)=q_i^{mkt}
$$

The solution is normally obtained sequentially from short maturities to long maturities.

Bootstrap differs from statistical curve fitting:

- bootstrap normally reprices each input instrument exactly;
- fitting may allow residual errors in exchange for a smoother or more parsimonious curve;
- interpolation remains part of the bootstrap because intermediate cash-flow dates require interpolated discount factors.

---

## 3. SOFR Instrument Ladder

A typical USD SOFR curve uses:

```text
Historical SOFR Fixings
        ↓
SR1 Monthly SOFR Futures
        ↓
SR3 Quarterly SOFR Futures
        ↓
SOFR OIS
        ↓
Complete SOFR Term Structure
```

### SR1 Futures

SR1 is based on the weighted arithmetic average of daily SOFR over a calendar month.

$$
P_{\mathrm{SR1}}=100-R_{\mathrm{SR1}}^{(\%)}
$$

### SR3 Futures

SR3 is based on compounded SOFR over an IMM-to-IMM reference quarter.

$$
G=\prod_j(1+r_j\delta_j)
$$

$$
R_{\mathrm{SR3}}=\frac{G-1}{\alpha}
$$

$$
P_{\mathrm{SR3}}=100-R_{\mathrm{SR3}}^{(\%)}
$$

Futures prices rise when implied rates fall.

---

## 4. Futures Convexity

A futures-implied rate is not necessarily identical to the corresponding forward rate because futures are marked to market daily.

$$
R_{\mathrm{futures}}
=
R_{\mathrm{forward}}
+
\text{Convexity Adjustment}
$$

A zero convexity adjustment is acceptable for a teaching implementation, but the assumption must be recorded explicitly.

---

## 5. OIS Bootstrap Equation

For a simplified spot-starting OIS:

$$
PV_{\mathrm{fixed}}
=
NK\sum_{i=1}^{n}\alpha_iD(0,T_i)
$$

$$
PV_{\mathrm{float}}
=
N\left[1-D(0,T_n)\right]
$$

At the par rate:

$$
K\sum_{i=1}^{n}\alpha_iD(0,T_i)
=
1-D(0,T_n)
$$

Each OIS quote therefore adds a constraint on the discount curve.

Production pricing must use actual schedules, payment lags, calendars, fixings, stubs, lookbacks and observation conventions instead of relying on the simplified formula.

---

## 6. QuantLib Rate Helpers

A `RateHelper` translates a market quote into an instrument-pricing equation.

```text
Market Quote
    ↓
RateHelper
    ├── Dates
    ├── Calendar
    ├── Day Count
    ├── Payment Rules
    ├── Cash-Flow Model
    └── Implied Quote
```

Main helpers:

```python
ql.SofrFutureRateHelper
ql.OISRateHelper
```

The core repricing condition is:

$$
\text{helper.impliedQuote()}
\approx
\text{helper.quote().value()}
$$

For futures, the implied quote is a futures price. For OIS, it is a decimal par rate.

---

## 7. Quote Units

Quote normalization must be explicit.

```text
SOFR futures price: 96.50
OIS market rate:    3.50% → 0.0350
1 bp:               0.0001
```

For a futures contract:

$$
P=100-100r
$$

Therefore a $0.01$ futures-price move corresponds approximately to a $1$ bp move in the opposite direction.

---

## 8. Curve Pillars and Instrument Selection

Important helper dates include:

```python
helper.earliestDate()
helper.maturityDate()
helper.latestRelevantDate()
helper.pillarDate()
helper.latestDate()
```

The contractual maturity, final payment date, latest fixing-relevant date and pillar date may differ.

Curve construction must reject duplicate pillars and review near-duplicate pillars.

A practical futures–OIS transition policy is:

```text
Use liquid SOFR futures through a chosen contract
        ↓
Apply a maturity buffer
        ↓
Use OIS beyond the futures section
```

More instruments do not necessarily produce a better curve. Overlapping or inconsistent instruments can create bootstrap failures or local forward spikes.

---

## 9. QuantLib Construction Pipeline

```text
Raw Market Data
    ↓
Quote Normalization
    ↓
SimpleQuote Registry
    ↓
SofrFutureRateHelper / OISRateHelper
    ↓
Pillar and Overlap Checks
    ↓
Piecewise Yield Curve
    ↓
Force Bootstrap
    ↓
Curve Handle
    ↓
Diagnostics
```

Typical curve classes:

```python
ql.PiecewiseLogCubicDiscount
ql.PiecewiseLinearZero
ql.PiecewiseFlatForward
```

Quotes should be stored in `ql.SimpleQuote` objects so that market updates and bucketed risk can use `setValue()` without rebuilding the complete object graph.

---

## 10. Lazy Bootstrap

Constructing a piecewise curve does not guarantee that bootstrap has succeeded.

```python
curve = ql.PiecewiseLogCubicDiscount(...)
```

QuantLib usually performs the actual calculation only when a result is requested.

```python
curve.discount(curve.maxDate())
```

A production-quality builder must force materialization and catch bootstrap exceptions before returning the curve.

---

## 11. Helper and Independent Repricing

### Helper Repricing

Check:

$$
q_i^{helper}-q_i^{market}\approx0
$$

This verifies consistency inside the bootstrap object graph.

### Independent Repricing

Rebuild each OIS independently using the bootstrapped curve and check:

$$
S_i^{fair}-S_i^{market}\approx0
$$

A swap struck at the market par rate should also satisfy:

$$
NPV_i\approx0
$$

If helper repricing passes but independent repricing fails, the likely cause is inconsistent conventions between the curve builder and the instrument builder.

---

## 12. Curve Representations

A curve should be inspected through:

- discount factors;
- continuous zero rates;
- short-period forwards;
- 3M forwards;
- 6M forwards.

The instantaneous forward is:

$$
f(0,T)
=
-\frac{\partial\ln D(0,T)}{\partial T}
$$

For a continuous zero curve:

$$
f(0,T)
=
z(0,T)
+
T\frac{\partial z(0,T)}{\partial T}
$$

A visually smooth zero curve can therefore conceal unstable local forwards.

---

## 13. Interpolation Comparison

### Piecewise Flat Forward

$$
f(t)=f_i,
\qquad t\in(T_{i-1},T_i]
$$

- transparent and local;
- forward rates jump at pillars;
- useful as a diagnostic benchmark.

### Piecewise Linear Zero

$$
z(t)=a_i+b_it
$$

$$
f(t)=a_i+2b_it
$$

- zero rates are continuous and piecewise linear;
- forwards can kink or jump at nodes.

### Log-Cubic Discount

Cubic interpolation is applied to $\ln D(0,t)$.

- usually produces smoother forwards;
- may overshoot between sparse or noisy nodes;
- quote bumps may affect a wider maturity region.

The correct criterion is not visual smoothness alone:

$$
\text{Curve Quality}
=
\text{Repricing}
+
\text{Forward Plausibility}
+
\text{Risk Stability}
$$

---

## 14. Forward Diagnostics

Local forwards are especially sensitive to interpolation and node placement.

Useful diagnostics include:

- adjacent 1D forward jumps;
- futures–OIS transition behavior;
- proximity to curve pillars;
- 3M and 6M forward differences across interpolation methods;
- sparse long-end nodes;
- sensitivity to small quote bumps.

A detected forward spike is a review flag, not automatically an error. It may reflect bad data, a pillar issue, interpolation overshoot, a policy-meeting expectation or a genuine turn effect.

---

## 15. Hard and Soft Validation Checks

### Hard Failures

- bootstrap exception;
- missing historical fixing;
- duplicate pillar;
- non-finite or non-positive discount factor;
- failed helper repricing;
- failed independent repricing;
- accidental extrapolation.

### Review Flags

- large local forward jump;
- strong interpolation divergence;
- sparse pillar spacing;
- locally increasing discount factors;
- unstable response to small quote bumps.

Discount factors must remain positive, but they need not always decrease in a negative-rate environment.

---

## 16. Recommended Diagnostic Order

When a curve looks wrong, check:

1. quote timestamp and units;
2. evaluation and reference dates;
3. calendars, day counts and payment conventions;
4. helper maturity and pillar dates;
5. futures–OIS overlap;
6. helper repricing;
7. independent OIS repricing;
8. discount, zero and forward shapes;
9. interpolation sensitivity;
10. bucketed quote risk.

Do not change interpolation before checking market data and conventions.

---

## 17. Wednesday Takeaway

The complete Wednesday framework is:

$$
\text{Tradable Quotes}
\rightarrow
\text{Instrument Equations}
\rightarrow
\text{Bootstrapped Curve}
$$

$$
\text{Bootstrap Success}
\not\Rightarrow
\text{Good Curve}
$$

A trustworthy SOFR curve must satisfy:

$$
\boxed{
\text{Correct Conventions}
+
\text{Exact Repricing}
+
\text{Reasonable Forwards}
+
\text{Stable Risk}
}
$$

The central lesson is:

> A rates curve is a calibrated pricing and risk system, not a plotted collection of yields.