# Rates Week: Tuesday

## 1. QuantLib Dependency Framework

QuantLib should be understood as a dynamic dependency graph rather than a collection of isolated pricing functions.

```text
Market Quote
    ↓
SimpleQuote
    ↓
QuoteHandle
    ↓
YieldTermStructure
    ↓
YieldTermStructureHandle
    ↓
Index / Pricing Engine
    ↓
Instrument
    ↓
NPV / Fair Rate / Risk
```

A market-data update propagates through this graph, and dependent objects are recalculated only when a result is requested.

---

## 2. Quotes and Handles

`ql.SimpleQuote` stores a mutable market input.

```python
quote = ql.SimpleQuote(0.04)
quote.setValue(0.041)
```

`ql.QuoteHandle` passes the quote into curves and other dependent objects without copying the value.

```python
quote_handle = ql.QuoteHandle(quote)
```

A handle provides indirection and dependency propagation. A relinkable handle can be redirected to a different curve:

```python
curve_handle = ql.RelinkableYieldTermStructureHandle()
curve_handle.linkTo(curve)
```

The distinction is:

- `quote.setValue(...)`: change one market-data node;
- `handle.linkTo(...)`: replace the curve, model, or scenario.

---

## 3. Yield-Term-Structure Interface

Different curve classes share the same `YieldTermStructure` interface:

```python
curve.discount(date)
curve.zeroRate(...)
curve.forwardRate(...)
```

The curve's internal representation does not determine its use. A curve represented through discount factors can still be used for forecasting, and a zero curve can be used for discounting.

---

## 4. Discount, Zero, and Forward Representations

Discount factors, zero rates, and forward rates are equivalent representations of the same pricing information.

Under continuous compounding:

$$
D(0,T)=e^{-z(0,T)T}
$$

The instantaneous forward rate is:

$$
f(0,T)=-\frac{\partial \ln D(0,T)}{\partial T}
$$

The zero rate is the average of instantaneous forwards:

$$
z(0,T)=\frac{1}{T}\int_0^T f(0,u)\,du
$$

Because zero rates are averages, a smooth zero curve can hide unstable local forward rates.

---

## 5. `FlatForward`

A flat-forward curve assumes a constant forward rate:

$$
f(0,t)=c
$$

Therefore:

$$
D(0,t)=e^{-ct}
$$

Example:

```python
curve = ql.FlatForward(
    reference_date,
    ql.QuoteHandle(quote),
    ql.Actual365Fixed(),
    ql.Continuous,
    ql.NoFrequency,
)
```

When the underlying `SimpleQuote` changes, the curve updates automatically.

---

## 6. `DiscountCurve`

`ql.DiscountCurve` takes discount-factor nodes and uses log-linear interpolation.

```python
curve = ql.DiscountCurve(
    dates,
    discount_factors,
    day_count,
    calendar,
)
```

Between two nodes:

$$
\ln D(t)=(1-w)\ln D_i+w\ln D_{i+1}
$$

Since $\ln D(t)$ is piecewise linear, the instantaneous forward rate is piecewise constant:

$$
f(0,t)
=
-\frac{\ln D_{i+1}-\ln D_i}
{t_{i+1}-t_i}
$$

Forward rates may jump at curve nodes.

---

## 7. `ZeroCurve`

`ql.ZeroCurve` takes zero-rate nodes and interpolates the zero rates directly.

```python
curve = ql.ZeroCurve(
    dates,
    zero_rates,
    day_count,
    calendar,
    ql.Linear(),
    ql.Continuous,
    ql.Annual,
)
```

For piecewise-linear zero rates:

$$
z(t)=a_i+b_it
$$

The instantaneous forward rate is:

$$
f(0,t)=z(0,t)+t\frac{\partial z(0,t)}{\partial t}
$$

Hence:

$$
f(0,t)=a_i+2b_it
$$

A linear zero curve does not imply a flat or globally smooth forward curve.

---

## 8. Interpolation Is Part of the Curve

Curve nodes alone do not define the full term structure.

$$
\text{Curve Nodes}
+
\text{Interpolation Rule}
=
\text{Complete Curve}
$$

Two curves can match exactly at every node but differ between nodes in:

- discount factors;
- forward rates;
- forward-starting swap rates;
- intermediate cash-flow PVs;
- bucketed sensitivities.

An interpolation method must always specify the interpolated quantity:

- linear zero;
- log-linear discount;
- flat forward;
- cubic zero;
- cubic discount.

---

## 9. Curve Diagnostics

A curve should be inspected through several representations:

1. discount factors;
2. zero rates;
3. short-period forward rates;
4. approximate instantaneous forwards;
5. differences between interpolation methods.

Node matching is necessary but not sufficient. Curve validation must include:

$$
\text{Node Fit}
+
\text{Shape Diagnostics}
+
\text{Risk Stability}
$$

Forward curves are especially important because differentiation amplifies local interpolation noise.

---

## 10. Compounding and Day-Count Conversion

Different quoted zero rates can represent the same discount factor.

For continuous compounding:

$$
D(0,T)=e^{-z_cT}
$$

For annual compounding:

$$
D(0,T)=\frac{1}{(1+z_a)^T}
$$

Usually $z_c\neq z_a$, but both can imply the same price.

The curve day count defines the internal time coordinate:

$$
t(d)=\operatorname{YearFraction}(t_{\mathrm{ref}},d)
$$

The day count passed to `zeroRate()` or `forwardRate()` determines how the output rate is reported.

---

## 11. Evaluation Date and Reference Date

The evaluation date is the global valuation date:

```python
ql.Settings.instance().evaluationDate
```

The curve reference date is the curve's time origin:

```python
curve.referenceDate()
```

They need not be equal.

$$
D(t_{\mathrm{ref}},t_{\mathrm{ref}})=1
$$

A fixed-reference curve keeps the same reference date when the evaluation date changes.

A moving curve defines its reference date from the evaluation date and settlement-day offset:

```python
moving_curve = ql.FlatForward(
    settlement_days,
    calendar,
    quote_handle,
    day_count,
)
```

The three dates must remain conceptually separate:

$$
\text{Evaluation Date}
\neq
\text{Curve Reference Date}
\neq
\text{Instrument Effective Date}
$$

---

## 12. Date-Based and Time-Based Queries

The following queries have different meanings:

```python
curve.discount(5.0)
curve.discount(fixed_calendar_date)
```

`discount(5.0)` refers to a maturity five years after the current curve reference date.

`discount(fixed_calendar_date)` refers to a specific contractual date. If a moving curve's reference date advances, the remaining maturity to that fixed date shortens.

This distinction separates:

- constant-maturity analysis;
- valuation of fixed contractual cash flows.

---

## 13. Observer and Lazy Evaluation

When a quote, curve, or evaluation date changes, dependent objects are marked stale.

```text
Market update
    ↓
Dependency notification
    ↓
Object marked stale
    ↓
Result requested
    ↓
Recalculation
```

QuantLib does not necessarily perform a full recalculation immediately after each update. This lazy-evaluation mechanism is essential for efficient curve rebuilding and risk calculations.

---

## 14. Relinking and Automatic Repricing

A pricing engine should depend on a curve handle rather than a permanently hard-coded curve.

```python
curve_handle = ql.RelinkableYieldTermStructureHandle()
curve_handle.linkTo(base_curve)

engine = ql.DiscountingBondEngine(curve_handle)
instrument.setPricingEngine(engine)
```

A scenario can then be changed without rebuilding the instrument:

```python
curve_handle.linkTo(stress_curve)
stressed_npv = instrument.NPV()
```

This architecture supports:

- base and stress scenarios;
- interpolation comparison;
- historical curve switching;
- discount/projection curve separation;
- portfolio repricing.

---

## 15. Extrapolation

A curve normally supports dates only up to `curve.maxDate()`.

```python
curve.maxDate()
curve.enableExtrapolation()
```

Allowing extrapolation does not make the extrapolated result economically reliable. Diagnostics should record:

- maximum curve date;
- whether extrapolation is enabled;
- distance beyond the last liquid point;
- extrapolation rule.

---

## 16. Engineering Discipline

The evaluation date is global mutable state. Research and test code should restore it after use.

```python
from contextlib import contextmanager

@contextmanager
def quantlib_evaluation_date(as_of_date):
    settings = ql.Settings.instance()
    old_date = settings.evaluationDate
    settings.evaluationDate = as_of_date
    try:
        yield
    finally:
        settings.evaluationDate = old_date
```

For parallel work, each worker should preferably own an independent QuantLib object graph and evaluation-date context.

---

## 17. Practical Curve Definition

A reproducible curve is not defined by zero-rate nodes alone.

$$
\text{Curve}
=
\text{Reference Date}
+
\text{Nodes}
+
\text{Representation}
+
\text{Interpolation}
+
\text{Day Count}
+
\text{Extrapolation Rule}
$$

Useful metadata include:

- evaluation date;
- reference date;
- maximum date;
- curve class;
- curve day count;
- interpolation method;
- compounding convention;
- extrapolation status.

---

## 18. Tuesday Takeaway

The main QuantLib framework is:

$$
\text{Market Quotes}
\rightarrow
\text{Handles}
\rightarrow
\text{Yield Curve}
\rightarrow
\text{Index / Engine}
\rightarrow
\text{Instrument}
$$

The main curve-construction principle is:

$$
\text{Same Nodes}
\not\Rightarrow
\text{Same Curve Between Nodes}
$$

The main diagnostic principle is:

$$
\text{Smooth Zero Curve}
\not\Rightarrow
\text{Stable Forward Curve}
$$

The main engineering principle is:

$$
\text{Market Update}
\rightarrow
\text{Notification}
\rightarrow
\text{Lazy Recalculation}
$$

Tuesday establishes the object-model and curve-representation foundation required to bootstrap a USD SOFR curve from futures and OIS market quotes.