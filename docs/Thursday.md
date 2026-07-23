# Rates Week: Thursday

## 1. Interest Rate Swap Framework

An interest rate swap exchanges two sets of future cash flows.

For a payer swap:

* pay fixed;
* receive floating.

$$
V_{\text{payer}}
=

PV_{\text{floating}}
-
PV_{\text{fixed}}
$$

For a receiver swap:

$$
V_{\text{receiver}}
=

PV_{\text{fixed}}
-
PV_{\text{floating}}
$$

For otherwise identical swaps:

$$
V_{\text{receiver}}
=

-V_{\text{payer}}
$$

---

## 2. Fixed Leg and Swap Annuity

For notional $N$, fixed rate $K$, accrual fractions $\alpha_i$, and payment dates $p_i$:

$$
PV_{\text{fixed}}
=

NK\sum_{i=1}^{n}\alpha_iD(0,p_i)
$$

Define the swap annuity:

$$
A
=

\sum_{i=1}^{n}\alpha_iD(0,p_i)
$$

Therefore:

$$
PV_{\text{fixed}}=NKA
$$

The accrual dates determine $\alpha_i$, while the actual payment date determines the discount factor.

---

## 3. Overnight Floating Leg

For an overnight coupon period:

$$
G_i
=

\prod_j(1+r_j\delta_j)
$$

$$
CF_i^{\text{ON}}
=

N(G_i-1)
$$

The floating-leg PV is:

$$
PV_{\text{floating}}
=

N\sum_i(G_i-1)D(0,p_i)
$$

For a simplified single-curve OIS without payment lag or special observation conventions:

$$
PV_{\text{floating}}
=

N\left[D(0,T_0)-D(0,T_n)\right]
$$

This telescoping identity is a sanity check. Production pricing should use the actual generated cash flows.

---

## 4. Fair Swap Rate and NPV

The fair rate $S$ is the fixed rate that makes the swap NPV equal to zero:

$$
S
=

\frac{PV_{\text{floating}}}{NA}
$$

For a payer swap with contractual fixed rate $K$:

$$
\boxed{
V_{\text{payer}}
=

NA(S-K)
}
$$

For a receiver swap:

$$
\boxed{
V_{\text{receiver}}
=

NA(K-S)
}
$$

Therefore:

* payer benefits when the current market swap rate rises above $K$;
* receiver benefits when the current market swap rate falls below $K$.

The swap rate is a weighted price of a sequence of cash flows, not a single-maturity zero rate.

---

## 5. Zero NPV Does Not Mean Zero Risk

A par swap starts with:

$$
K=S_0
$$

$$
V_0=0
$$

At the par point:

$$
\left.
\frac{\partial V_{\text{payer}}}{\partial S}
\right|_{S=K}
=

NA
$$

Therefore:

$$
NPV=0
\not\Rightarrow
DV01=0
$$

Zero NPV is a value-level statement; risk depends on the sensitivity of value to changes in the curve.

---

## 6. Fixed-Leg BPS and Annuity

An increase of $1$ bp in the contractual fixed rate changes the unsigned fixed-leg PV by approximately:

$$
PV01_{\text{fixed}}
=

NA\times10^{-4}
$$

QuantLib's `fixedLegBPS()` is signed:

* negative for a payer fixed leg;
* positive for a receiver fixed leg.

The annuity can be recovered from:

$$
A
=

\frac{
|\text{fixedLegBPS}|
}{
N\times10^{-4}
}
$$

`fixedLegBPS()` is not the full market DV01:

* `fixedLegBPS()` bumps the contractual fixed coupon;
* market DV01 bumps market quotes, rebuilds the curve, and reprices the swap.

---

## 7. QuantLib OIS Object Graph

```text
SOFR Curve Handle
    ├── ql.Sofr(handle)
    │       └── Forecast future overnight fixings
    └── ql.DiscountingSwapEngine(handle)
            └── Discount cash flows

ql.MakeOIS(...)
    ↓
ql.OvernightIndexedSwap
    ↓
NPV / fairRate / leg NPV / leg BPS
```

Typical construction objects are:

```python
sofr_index = ql.Sofr(curve_handle)
engine = ql.DiscountingSwapEngine(curve_handle)
swap = ql.MakeOIS(...)
swap.setPricingEngine(engine)
```

In the simplified SOFR framework, the same curve can be used for forecasting and discounting, but the two functions remain conceptually separate.

---

## 8. Important OIS Conventions

An OIS definition should explicitly specify:

* effective and termination dates;
* payment frequency;
* fixed-leg day count;
* schedule calendar;
* fixing calendar;
* payment calendar;
* business-day conventions;
* payment lag;
* lookback;
* observation shift;
* lockout;
* compounding method;
* payer or receiver direction;
* notional.

The conventions used by `OISRateHelper` during curve construction must match those used by the independently constructed OIS.

---

## 9. Core QuantLib Outputs

```python
swap.NPV()
swap.fairRate()
swap.fixedLegNPV()
swap.overnightLegNPV()
swap.fixedLegBPS()
swap.overnightLegBPS()
```

For a payer swap:

$$
NPV
=

NPV_{\text{fixed leg}}
+
NPV_{\text{overnight leg}}
$$

where the fixed-leg NPV is normally negative and the overnight-leg NPV is normally positive.

A par swap rebuilt using `fairRate()` should satisfy:

$$
NPV\approx0
$$

---

## 10. Cash-Flow Inspection

The fixed leg should be audited coupon by coupon:

```python
for cashflow in swap.fixedLeg():
    coupon = ql.as_fixed_rate_coupon(cashflow)
```

Important fields include:

* accrual start date;
* accrual end date;
* payment date;
* accrual fraction;
* coupon rate;
* coupon amount;
* discount factor;
* present value.

An annual overnight coupon may contain hundreds of daily SOFR observations. Its coupon rate is an annualized compounded rate, not a single fixing.

---

## 11. Spot and Forward-Starting Swaps

A spot-starting 5Y swap starts near the current spot date and matures approximately five years later.

A 1Y5Y swap:

* begins approximately one year from spot;
* runs for five years;
* matures approximately six years from spot.

Its simplified fair rate is:

$$
S_{1Y5Y}
=

\frac{
D(0,T_{\text{start}})
-

D(0,T_{\text{end}})
}{
\sum_i\alpha_iD(0,p_i)
}
$$

Generally:

$$
S_{1Y5Y}\neq S_{5Y}
$$

$$
S_{1Y5Y}\neq S_{6Y}
$$

A forward-starting swap has risk today even though no cash flow occurs before its future effective date.

Forward swaps are useful diagnostics for interpolation because their pricing depends heavily on discount factors between calibration pillars.

---

## 12. Started Swaps and Historical Fixings

For a coupon spanning the evaluation date:

$$
G
=

G_{\text{past}}G_{\text{future}}
$$

where:

$$
G_{\text{past}}
=

\prod_{j\in\text{past}}
(1+r_j^{\text{realized}}\delta_j)
$$

$$
G_{\text{future}}
=

\prod_{j\in\text{future}}
(1+r_j^{\text{forecast}}\delta_j)
$$

Past fixing dates must use historical SOFR observations. Future fixing dates use the forecasting curve.

Historical fixings can be loaded with:

```python
index.addFixing(date, rate)
index.addFixings(dates, rates)
```

Future observations must never be inserted as historical fixings, as this creates look-ahead bias.

The treatment of today's fixing must depend on whether it has already been officially published at the valuation timestamp.

---

## 13. Lookback, Observation Shift and Lockout

With a lookback, the fixing date is moved backward:

$$
o_i
=

d_i-k\text{ business days}
$$

Without observation shift, the shifted fixing is applied to the original accrual interval:

$$
G_{\text{lookback}}
=

\prod_i
\left[
1+r(o_i)\operatorname{YearFraction}(d_i,d_{i+1})
\right]
$$

With observation shift, both the fixing dates and accrual weights are shifted:

$$
G_{\text{shift}}
=

\prod_i
\left[
1+r(o_i)\operatorname{YearFraction}(o_i,o_{i+1})
\right]
$$

Therefore:

* lookback changes which rate is observed;
* observation shift also changes the accrual weight associated with that rate;
* lockout freezes the last several observations at an earlier fixing;
* payment lag changes the payment date but not the fixing structure.

---

## 14. Telescopic Value Dates

`telescopicValueDates=True` can reduce the computational cost of long overnight coupons.

It is appropriate for short-lived calibration instruments such as rate helpers, but requires caution for persistent trade objects whose valuation date may roll forward.

A safer default for long-lived swap objects is:

```python
telescopicValueDates=False
```

unless the instrument is rebuilt whenever the evaluation date changes and the convention combination is known to be compatible.

---

## 15. Repricing Validation

Curve validation should use two independent layers.

### Helper repricing

$$
q_i^{\text{helper}}
-
q_i^{\text{market}}
\approx0
$$

This checks the internal bootstrap object graph.

### Independent swap repricing

Rebuild the swap independently and verify:

$$
S_i^{\text{independent}}
-

q_i^{\text{market}}
\approx0
$$

A swap struck at the market rate should also satisfy:

$$
NPV_i\approx0
$$

If helper repricing succeeds but independent repricing fails, the likely cause is inconsistent conventions or date generation.

---

## 16. Required Sanity Checks

A robust OIS implementation should verify:

```python
assert abs(par_swap.NPV()) < tolerance
assert payer.NPV() == pytest.approx(-receiver.NPV())
assert swap.NPV() == pytest.approx(
    swap.fixedLegNPV() + swap.overnightLegNPV()
)
assert off_market_swap_2x.NPV() == pytest.approx(
    2.0 * off_market_swap.NPV()
)
```

Also check:

* fixed rate above fair rate gives a negative payer NPV;
* fixed rate below fair rate gives a positive payer NPV;
* notional scaling is linear;
* payment and accrual dates follow the configured conventions;
* all required historical fixings are present;
* no future fixing is treated as historical.

---

## 17. Thursday Takeaway

The complete Thursday framework is:

$$
\text{Curve}
\rightarrow
\text{Forecast Fixings}
\rightarrow
\text{Generate Cash Flows}
\rightarrow
\text{Discount Legs}
\rightarrow
\text{Fair Rate and NPV}
$$

For a payer swap:

$$
\boxed{
V_{\text{payer}}=NA(S-K)
}
$$

A production-quality swap pricer requires:

$$
\boxed{
\text{Correct Curve}
+
\text{Correct Conventions}
+
\text{Correct Dates}
+
\text{Correct Fixings}
+
\text{Independent Repricing}
}
$$

The central discipline is to treat a swap as a transparent collection of dated cash flows, not as a black-box call to `swap.NPV()`.
