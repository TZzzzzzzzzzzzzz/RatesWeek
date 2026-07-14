# Rates Week: Monday

## 1. Core Framework

Rates products should be understood as dated cash flows discounted by an interest-rate term structure.

$$
\text{Market Quotes}
+
\text{Conventions}
\rightarrow
\text{Cash Flows}
\rightarrow
\text{Present Value}
$$

The fundamental object is the discount factor $D(t,T)$, representing the value at time $t$ of receiving one unit of currency at time $T$.

$$
PV_t(C_T)=C_TD(t,T)
$$

Discount curves, zero curves, forward curves, and par swap curves are different representations of the same underlying pricing information.

---

## 2. Discount Factors and Zero Rates

Under continuous compounding:

$$
D(0,T)=e^{-z(0,T)\tau(0,T)}
$$

$$
z(0,T)=-\frac{\ln D(0,T)}{\tau(0,T)}
$$

Under annual compounding:

$$
D(0,T)=\frac{1}{(1+z(0,T))^{\tau(0,T)}}
$$

Under simple compounding:

$$
D(0,T)=\frac{1}{1+z(0,T)\tau(0,T)}
$$

A quoted rate is incomplete unless its start date, end date, day-count convention, compounding convention, and rate type are specified.

---

## 3. Forward Rates

A forward rate is the rate implied today for a future interval $[T_1,T_2]$.

For simple compounding:

$$
F(0;T_1,T_2)
=
\frac{\frac{D(0,T_1)}{D(0,T_2)}-1}
{\alpha(T_1,T_2)}
$$

Under continuous compounding:

$$
D(0,T)
=
\exp\left(
-\int_0^T f(0,u)\,du
\right)
$$

$$
f(0,T)
=
-\frac{\partial \ln D(0,T)}{\partial T}
$$

The zero rate is an average over future instantaneous forward rates:

$$
z(0,T)
=
\frac{1}{T}
\int_0^T f(0,u)\,du
$$

A smooth zero curve may still imply an unstable or oscillating forward curve, so curve diagnostics must inspect both.

---

## 4. Par Swap Rate

For a fixed leg with payment dates $T_1,\ldots,T_n$:

$$
PV_{\text{fixed}}
=
NK\sum_{i=1}^{n}\alpha_iD(0,T_i)
$$

Define the swap annuity:

$$
A
=
\sum_{i=1}^{n}\alpha_iD(0,T_i)
$$

In a simplified single-curve framework:

$$
PV_{\text{float}}
=
N\left[D(0,T_0)-D(0,T_n)\right]
$$

The par swap rate is:

$$
S(0;T_0,T_n)
=
\frac{D(0,T_0)-D(0,T_n)}
{\sum_{i=1}^{n}\alpha_iD(0,T_i)}
$$

For a spot-starting swap where $D(0,T_0)=1$:

$$
S
=
\frac{1-D(0,T_n)}
{\sum_{i=1}^{n}\alpha_iD(0,T_i)}
$$

A par swap starts with approximately zero NPV, but both legs can have large present values and substantial interest-rate risk.

---

## 5. Swap Direction

A payer swap pays fixed and receives floating:

$$
NPV_{\text{payer}}
=
PV_{\text{floating}}
-
PV_{\text{fixed}}
$$

A receiver swap receives fixed and pays floating:

$$
NPV_{\text{receiver}}
=
PV_{\text{fixed}}
-
PV_{\text{floating}}
$$

For otherwise identical swaps:

$$
NPV_{\text{receiver}}
=
-NPV_{\text{payer}}
$$

A payer swap generally benefits when market swap rates rise; a receiver swap generally benefits when they fall.

---

## 6. Coupon Cash Flows

A generic coupon is:

$$
CF_i=N_iR_i\alpha_i
$$

Its present value is:

$$
PV_i=CF_iD(0,p_i)
$$

The main dates must be distinguished:

- fixing or observation date;
- accrual start date;
- accrual end date;
- payment date.

The payment date determines discounting, while the accrual dates determine the year fraction.

---

## 7. Term-Rate IRS

For a term floating-rate coupon:

$$
CF_i^{\text{float}}
=
N_i(L_i+s)\alpha_i
$$

A standard term-rate coupon is typically fixed near the start of the accrual period and paid at the end.

The following dates are distinct:

$$
\text{Fixing Date}
\rightarrow
\text{Value Date}
\rightarrow
\text{Maturity Date}
$$

The index object should determine these dates; they should not be reconstructed manually.

---

## 8. SOFR OIS

An OIS exchanges a fixed leg against a compounded overnight-rate leg.

For one overnight coupon period:

$$
G_i
=
\prod_{j=1}^{m}
\left(1+r_j\delta_j\right)
$$

$$
CF_i^{\text{ON}}
=
N_i
\left[
\prod_{j=1}^{m}(1+r_j\delta_j)-1
\right]
$$

The annualized compounded rate is:

$$
R_i^{\text{comp}}
=
\frac{
\prod_{j=1}^{m}(1+r_j\delta_j)-1
}{
\alpha_i
}
$$

SOFR OIS uses compounded overnight fixings, not a simple arithmetic average.

Weekend and holiday accruals are reflected through larger day-count fractions, such as $\delta=3/360$ for a Friday-to-Monday interval.

---

## 9. Historical and Forecast Fixings

For a coupon spanning the valuation date:

$$
G_i
=
\prod_{j\in\text{past}}
(1+r_j^{\text{realized}}\delta_j)
\times
\prod_{j\in\text{future}}
(1+r_j^{\text{forecast}}\delta_j)
$$

Past fixings must come from historical market data. Future fixings are projected from the forecasting curve.

Forecasting and discounting are separate operations:

$$
\text{Forecast Curve}
\rightarrow
\text{Coupon Amount}
\rightarrow
\text{Discount Curve}
\rightarrow
PV
$$

In a simple SOFR OIS setup, one curve may serve both roles, but the concepts remain distinct.

---

## 10. OIS Conventions

Important overnight-rate conventions include:

- payment delay;
- lookback;
- observation shift;
- lockout;
- averaging or compounding method.

These conventions change either the observation dates, the accrual weights, the payment dates, or the final economic exposure.

The simplified floating-leg identity $N[D(0,T_0)-D(0,T_n)]$ should only be used as a theoretical sanity check. Production pricing should generate and discount each actual cash flow.

---

## 11. QuantLib Date System

The main QuantLib objects are:

```python
ql.Settings.instance().evaluationDate
ql.Date
ql.Period
ql.Calendar
ql.DayCounter
ql.Schedule
```

The evaluation date defines the valuation date for instruments, fixings, and term structures.

A `Period` is a market tenor, not a fixed number of days:

$$
1M\neq30D,\qquad 1Y\neq365D
$$

`Date + Period` performs calendar arithmetic but does not apply market holidays. Market date movement should normally use `calendar.advance()`.

---

## 12. Calendars and Business-Day Conventions

A calendar defines which dates are valid business days for a specific market.

Common business-day conventions:

- Following;
- Modified Following;
- Preceding;
- Modified Preceding;
- Unadjusted.

`calendar.adjust()` modifies an existing date.

`calendar.advance()` moves a date by a tenor or number of business days and then applies the relevant adjustment and end-of-month rules.

The market-specific calendar must match the instrument convention; choosing a generic calendar by currency is insufficient.

---

## 13. Day-Count Conventions

The accrual fraction is:

$$
\alpha
=
\operatorname{YearFraction}(a,b)
$$

Common conventions include:

- Actual/360;
- Actual/365 Fixed;
- 30/360 variants;
- Actual/Actual variants.

For Actual/360:

$$
\alpha
=
\frac{\text{actual calendar days}}{360}
$$

For Actual/365 Fixed:

$$
\alpha
=
\frac{\text{actual calendar days}}{365}
$$

Instrument day count determines coupon accrual. Curve day count determines the numerical time coordinate of the term structure. They do not need to be identical.

---

## 14. Schedule Construction

A schedule defines accrual boundaries:

$$
d_0,d_1,\ldots,d_n
$$

which generate coupon periods:

$$
[d_0,d_1], [d_1,d_2],\ldots,[d_{n-1},d_n]
$$

Key schedule inputs:

- effective date;
- termination date;
- tenor;
- calendar;
- regular-date convention;
- termination-date convention;
- forward or backward generation;
- end-of-month rule;
- optional first or next-to-last date.

Backward generation usually creates a front stub. Forward generation usually creates a back stub.

A stub is an irregular first or final accrual period and directly affects coupon amounts, par rates, and risk.

---

## 15. End-of-Month and Payment Dates

If end-of-month is enabled and the start date is a business month-end, future schedule dates should remain aligned with business month-end.

Schedule dates are usually accrual boundaries, not necessarily payment dates.

With a payment lag:

$$
\text{Accrual End}
\rightarrow
\text{Payment Date}
$$

The coupon year fraction uses the accrual dates, while discounting uses the payment date.

---

## 16. QuantLib Object Map

```text
Evaluation Date
    ├── Calendar
    ├── DayCounter
    ├── Business-Day Convention
    └── Schedule
          ├── FixedRateLeg
          ├── IborLeg
          └── OvernightLeg
                └── Index
                      ├── Historical Fixings
                      └── Forecast Curve

Legs
    ↓
VanillaSwap / OvernightIndexedSwap
    ↓
DiscountingSwapEngine
    ↓
NPV / Fair Rate / Leg NPV / Leg BPS
```

Core classes:

```python
ql.VanillaSwap
ql.OvernightIndexedSwap
ql.Sofr
ql.MakeOIS
ql.DiscountingSwapEngine
```

---

## 17. Practical Diagnostic Order

When two swap valuations disagree, check in this order:

1. evaluation date;
2. effective and spot dates;
3. market calendar;
4. business-day convention;
5. termination-date convention;
6. schedule generation direction;
7. front or back stub;
8. end-of-month rule;
9. fixed and floating day counts;
10. payment lag and payment calendar;
11. fixing and observation conventions;
12. historical fixings;
13. forecast and discount curve assignments.

Most early QuantLib discrepancies are caused by inconsistent instrument definitions rather than sophisticated numerical errors.

---

## 18. Monday Takeaway

The complete Monday framework is:

$$
\text{Discount Factors}
\rightarrow
\text{Zero and Forward Rates}
\rightarrow
\text{Par Swap Rates}
$$

$$
\text{Conventions}
\rightarrow
\text{Dated Cash Flows}
\rightarrow
\text{Forecasting}
\rightarrow
\text{Discounting}
\rightarrow
\text{NPV and Risk}
$$

The central discipline of rates pricing is to reduce every product to correctly specified dated cash flows and value them using the appropriate term structures.
