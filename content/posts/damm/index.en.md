---
title: "Differential AMM: a flexible AMM algorithm based on micro indicators"
date: 2021-05-19
tags: [technical]
---

Automatic Market-Making strategies (AMM) are usually defined by macro indicators. Macro indicators refer to indicators that describe the total amount, including the base amount (BA) and the quote amount (QA) of the trading tokens in the AMM pool. For example, the most common AMM algorithm - the Constant Product Algorithm - requires the product of BA and QA to be equal before and after each transaction. But in fact, for each specific trader, micro indicators are more informative for a single transaction. Micro indicators refer to indicators that characterize market changes, or marginal effects of transactions, such as current market price and market depth. The market price describes the average price of a small transaction in the current state, and the market depth describes the largest potential trading quantity within the specified transaction price. In plain language, let’s assume that the price of ETH-USDT is now 3450. If I want to buy as much ETH as possible at a price not exceeding 3451, this amount can be regarded as the market depth.

From the orderbook perspective, both price and depth are more intuitive than macro indicators. We hope that we can better understand and design the AMM algorithm based on these two micro indicators. 

# Mathematical Background - Relationship between macro and micro indicators

In this section, we will state the relationship between the micro indicators (price and depth) and the base/quote amount. As we will see, the functions of price and depth can both be derived from the base/quote function by several derivative/inverse function transformations.

Assume that base amount and quote amount in AMM pool satisfy the following relationship:

```
quote = f(base)
```

Obviously, we have quote > 0, base > 0. f is monotonically decreasing, i.e., f' < 0 (as in trading, when amount of one token increases, the other will of course decrease).

By definition, price is the absolute value of the derivative of f (as f' is negative, it's actually the reverse value of f'):

```
price = abs(f'(base)) = -f'(base)
```

Normally we demand the price (-f') to be monotonically decreasing, i.e., the smaller the base amount, the higher the price. Thus, f'' > 0. In summary, a reasonable f should be: a convex downward and monotonically decreasing function defined in the first quadrant.

Correspondingly, market depth is actually the derivative of base amount w.r.t. price.

```
depth = f2'(price)
f2 = Inverse(f') 
```
Where `Inverse` means inverse function.

In the above formula, f2 is the inverse function of the price function, and its derivative is the function of depth w.r.t. price.

That concludes the relationship between market price/depth and base/quote function.

# Mathematical Background - Solving

For a creator of AMM pool, they care the most about the initial price (price0) and initial depth (depth0). From mathematic perspective, they need to give an AMM function of the form `quote = f(base)`, so that there's one point in this function satisfying `price = price0` and `depth = depth0`.

In other words, they need to solve the following equation:

```
Knowing depth0, price0,
f is the function describing relationship between base and quote amount, quote = f(base). Find f that satisfies:

* There's base0, such that f'(base0) = price0
* (Inverse(f'))'(price0) = depth0

Additionally, it's required that f > 0, f' < 0 and f'' > 0, that is, f is a convex downward and monotonically decreasing function defined in the first quadrant.
```

Actually, as the above equation has too few constraints, there are infinite number of solutions. f could be a reciprocal function, a more general hyperbolic function, an exponential function, or a power function. However from the next section, we will see that if we specify a certain form of f and control the changeable parameters, we may be able to determine a unique solution.

### One solution - when f is a translated reciprocal function

If f is a reciprocal function in form:

```
quote = f(base) = C / (base + BASE_DELTA) - QUOTE_DELTA 
```

Or equivalantly, using multiple variables:
```
vQuote = C / vBase
vQoute = quote + QUOTE_DELTA
vBase = base + BASE_DELTA
```
where `C`, `BASE_DELTA`, `QUOTE_DELTA` are positive constants.

When f is in this form, the price and depth could be derived as:

```
price = -f'(base) = C / (vBase**2) = vQuote / vBase
depth = vBase**2 / (2 * vQuote)
```

According to the condition that at `price0`, the depth is `depth0`, it can be solved as:

```
C = vBase0 * vQuote0 = 4 * price0**3 * depth0**2
vQuote0 = 2 * price0**2 * depth0 = quote0 + QUOTE_DELTA
vBase0 = 2 * price0 * depth0 = base0 + BASE_DELTA
```

So finally, the AMM function `quote = f(base)` will be:

```
quote = f(base) = C / (base + BASE_DELTA) - QUOTE_DELTA,
```
where `C = 4 * price0**3 * depth0**2`, `BASE_DELTA` and `QUOTE_DELTA` are random constants.

To conclude, we demonstrated that if specified f as reciprocal function, the unique solution of f could be obtained by price and depth. Although f can actually be any types of functions, for simplicity, we will use reciprocal function as our final AMM function in the rest of the post.

# Formula definition of Differential AMM

Synthesizing all the derivations in the previous section, we can get the complete definition of Differential AMM:

```
quote = f(base) = C / (base + BASE_DELTA) - QUOTE_DELTA
```

Meaning of all related parameters of DAMM:

| Name | Meaning |
| --- | --- | 
| price0 | initial market price |
| depth0 |  initial market depth |
| lowPrice |  lowest automatic market-making price, minimum is 0 |
| highPrice |  highest automatic market-making price, maximum is positive inifinite |
| quote0 |  initial actual quote amount |
| base0 |  initial actual base amount |
| QUOTE_DELTA |  virtual quote amount |
| BASE_DELTA |  virtual base amount |
| C |  constant product |
| vQoute0  | initial total quote amount, including actual and virtual |
| vBase0 | initial total base amount, including actual and virtual |

Relationships between the parameters:

```
(1) C = 4 * price0**3 * depth0**2
(2) lowPrice = QUOTE_DELTA**2 / C
(3) highPrice = C / BASE_DELTA**2
(4) price0 = vQuote0 / vBase0
(5) depth0 = vBase**2 / (2 * vQuote)
(6) vQoute0 = 2 * price0**2 * depth0
(7) vBase0 = 2 * price0 * depth0
(8) vQoute0 = quote0 + QUOTE_DELTA
(9) vBase0 = base0 + BASE_DELTA
```

Note1: Among the above formulas, (4)(5) and (6)(7) are equivalant.

Note2: With the same `price0` and `depth0`, the smaller `QUOTE_DELTA` and `BASE_DELTA` are, the larger `base0` and `quote0` are. That is, when providing the same market liquidity, a greater amount of fund will lead to a lower so-called "capital efficiency", but a wider market-making price range (i.e., from `lowPrice` to `highPrice`). Extremely, when `QUOTE_DELTA` and `BASE_DELTA` are both 0, `lowPrice` will be 0 and `highPrice` will be inifinite. In this situation, DAMM degenerates into Constant Product AMM. We could adjust capital efficiency arbitrarily by adjusting `lowPrice` and `highPrice`.

In reality, for the following 3 groups of parameters, (A) depth price, (B) lowPrice highPrice, (C) quote0 base0, given any two groups, the third group, as well as the complete market-making function could be derived.

Here are the three scenarios corresponding to three initial conditions:

### Given price and depth (A), and initial amounts (C)

That is, in the equations, given `price0`, `depth0`, and `quote0`, `base0`, need to calculate market-making price range and other parameters.

Solving process refer to: <https://github.com/Fluidex/differential-amm/blob/673b2801c822bc5e75dc63f1def0204b8d57bb03/main.ts#L39>

### Given price and depth (A), and market-making price range (B)

That is, in the equations, given `price0`, `depth0`, and `lowPrice` (could be 0), `highPrice` (could be inifinity), need to calculate initial amount and other parameters.

Solving process refer to: <https://github.com/Fluidex/differential-amm/blob/673b2801c822bc5e75dc63f1def0204b8d57bb03/main.ts#L53>

### Given market-making price range (B), and initial amounts (C)

This is the most complex scenario. Actually it is equivalant to solving a two-dimensional quadratic equation. By solving the equation we could get all other parameters.

Solving process refer to: <https://github.com/Fluidex/differential-amm/blob/673b2801c822bc5e75dc63f1def0204b8d57bb03/main.ts#L72>

From the above three scenarios, we could see that DAMM has great flexibility. On the one hand, different scenarios may have different requirements and initial conditions. Under the above three initial conditions, we are all able to solve a correct DAMM strategy. On the other hand, **we can abitrarily adjust capital efficiency through the price range or initial amount**.

# Miscellaneous

## Converting AAM to orderbook

By approximating the AMM curve section by section, we can get a discretized orderbook. In the [implementation code](https://github.com/Fluidex/differential-amm/blob/673b2801c822bc5e75dc63f1def0204b8d57bb03/main.ts#L156), we could specify price interval and the number of orders, by calculating BASE_DELTA and QUOTE_DELTA, we could get order amount and average price.

## Mathematical equivalance

It's easy to see that DAMM is mathematically equivalant to the market maker algorithm (x + a)(y + b) = k. The difference is that we provided many micro interpretations, as well as possibilities to solve from different initial conditions.

