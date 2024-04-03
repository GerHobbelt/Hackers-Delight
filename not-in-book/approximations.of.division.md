# Approximations of multiplications (on limited hardware)

Takes the [[Approximations of multiplications (on limited hardware)]] notes as a prelude.

The HD book devotes two whole chapters to division, and that is great!

Here we consider further speedups under the same premise as we had for the [[Approximations of multiplications (on limited hardware)]] notes: minimum capability and/or otherwise reduced hardware and severely limited clock cycle budgets, which require us to cut corners *hard* by using *tolerable* approximations.

So we could, if our hardware didn't have a hardware multiplier on board, reduce a multiplication to bitshifting only the most important bits or factors of the multiplicand, depending on whether we are faster/better off doing it via shift+add or basic multiplier operations.

The same train of thought applies to divisions and is far more relevant here, as very few low grade MCUs have a (decent) `divu` opcode; even sophisticated PC/workstation hardware has trouble with doing divisions in fewest clock cycles!

So the question then becomes: what error term (rather: error factor) are we willing to accept?

First off: as HD shows / addresses: division by constant can be, with some effort, replaced by *multiplication by constant* or rather: replaced by multiplication by near-reciprocal constant plus a bit of corrections' work, as discussed at length in HD chapter 10.

Those concepts in HD start with 

$$ \frac{a}{b}  =  a * \frac{1}{b}  \ident \frac{ a * \frac{2^p}{b} }{ 2^p } $$

using a bit of ad-hoc $2^p$ fixed-point arithmetic.

Thanks to accuracy and rounding loss the plot thickens immediately, thus the need for various corrections, adjustments, etc., some of which -- the way I read it -- also depend on your precise definition of what an integer division should entail, particularly when considering negative integer values!

------

Here, the initial approximation idea (and error factor question) is:

what if we approx $\frac{a}{b}$ with $\frac{a}{2^q}$ where $2^q$ is a close-enough approximation of $b$?

$$
\varepsilon = | \frac{a}{b} - \frac{a}{2^q} |
$$ 

There's two options for $q$ really: $\floor(log2(b))$ and $\ceil(log2(b))$.

```
p = 2 ** floor(log2(b))
err = |a/b - a/p|

==>

err * b * p = |ap - ab|

err * (bp/a) = |p - b|

err = |b - p| * a / (b * p)
```

worst-case choice for p is when b is 1 off from the next / previous power of 2,
in which case we might say that `p = 2(b - 1) ~ 2b` or `p = 1/2 (b + 1) ~ 1/2b`, hence:

```
err = |b - p| * a / (b * p)

==>

err = |b - 2b| * a / (b * 2b)

err = |b| * a / (2b^2)

err = a / 2b

err = 1/2 * a/b
```

or alternatively:

```
err = |b - p| * a / (b * p)

==>

err = |b - 1/2b| * a / (b * 1/2b)

err = |1/2b| * a / (1/2b^2)

err = a / b
```

so it's immediately obvious that the worst-case error factor is worse when we pick the lower power-of-2 as a divider. (If you consider what division by a number does, how it impacts the resulting value, this division-by-a-smaller-divisor-is-potentially-worse is intuitively obvious.)

However, these two errors go in different directions, so then the next question becomes:

can we aim somewhere "in the middle" between these two? As the wordt-case error factor at least is skewed towards the lower side, taking the "middle value" i.e. the average of $\frac{a}{2^p}$ and $\frac{a}{2^{p-1}}$ doesn't seem like such a good idea, so how about aiming nearer to higher-valued divisor $2^p$?

If we have a cheap multiplier opcode, we can help the estimate by weighting these two divisions proportionally:

$$
p = \ceil{\log_2{b}}

d_0 = 2^p

d_1 = 2^{p-1}

w = \frac{ b - d_1 }{ d_0 - d_1 }

\frac{a}{b} \approx \frac{a}{d_0} * w + \frac{a}{d_1} * (1 - w)
$$

where we can get rid of the division in the weight $w$ expression as $d_0$ and $d_1$ are consecutive powers-of-2:

$$
d_0 = 2^p

d_1 = 2^{p-1}

w = \frac{ b - d_1 }{ d_0 - d_1 }

w = \frac{ b - d_1 }{ d_1 * 2 - d_1 }

w = \frac{ b - d_1 }{ d_1 }
$$

which essentially has become a subtraction and a single shift-right operation.

As for the error factor, we expect the biggest discrepancies at the side where the divisors are smaller values, which would make any difference versus reality in the divisor have more impact, percentually, so we could look at the worst case error term at the bottom end: thanks to the weighting we will have gotten closer, but there remains a difference:

```
err = a/b - (w * a/d0 + (1 - w) * a/d_1)

assuming we're worse at the lower end, the weight w will be near zero, hence

b - d_1 = 1 for the worst case and 

err <= a/b - a/d_1

==>

err <= a/b - a/(b - 1)

err * b * (b-1) <= a(b-1) - ab

err <= ( a(b-1) - ab ) / (b * (b-1) )

err <= ( -a ) / (b * (b-1) )

and lower divisors make this worse, so we can say

err <= a / ( (b-1) * (b-1) )

err <= a / (b-1)^2
```

which is a vast improvement over the original worst case error factor `a/b` but still seems quite significant for small `b`, e.g. `b = 2`.

Let's see with an example:

```
a = 42
b = 3

approximation:

d0 = 4
d1 = 2

r0 = 42 / 4 = 10
r1 = 42 / 2 = 21

w = (b - d1) / (d0 - d1) = (3 - 2) / (4 - 2) = 1/2

r = w * r0 + (1 - w) * r1 = 1/2 * 10 + (1 - 1/2) * 21 = 1/2 * 10 + 1/2 * 21 = (1 * 10 + 1 * 21) / 2 = 31 / 2 = 15

versus

42/3 = 14

making that an error of 15-14 = 1 ~= 7%
```

which is possibly acceptable.

How about some larger divisor?

```
a = 247
b = 17

approximation:

d0 = 32
d1 = 16

r0 = 247 / 32 = 7
r1 = 247 / 16 = 15

w = (b - d1) / (d0 - d1) = (17 - 16) / (32 - 16) = 1/16

r = w * r0 + (1 - w) * r1 = 1/16 * 7 + (1 - 1/16) * 15 = 1/16 * 7 + 15/16 * 15 = ( 1 * 7 + 15 * 15) / 16 = 232 / 16 = 14.5

versus

247/17 = 14.529

making that an error of 14-14 = 0% (as we're only interested in *integer* division here)
```

same for a value near the other end:

```
a = 247
b = 31

approximation:

d0 = 32
d1 = 16

r0 = 247 / 32 = 7
r1 = 247 / 16 = 15

w = (b - d1) / (d0 - d1) = (31 - 16) / (32 - 16) = 15/16

r = w * r0 + (1 - w) * r1 = 15/16 * 7 + (1 - 15/16) * 15 = 15/16 * 7 + 1/16 * 15 = ( 15 * 7 + 1 * 15) / 16 = 8 * 15 / 16 = 7.5

versus

247/31 = 7.96

making that an error of 14-14 = 0% (as we're only interested in *integer* division here)

while the worst case error estimate is

err <= a / (b-1)^2

err <= 247 / 30^2 = 0.274
```

but we do note that the error *after the decimal point* is quite a bit larger for this one. Of course, these are just examples and our error formula/estimate was a worst-case one, but we calculated that one as well and we must have dropped the ball as our actual error is *larger* than our worst case estimate.
Oops! hash tag `#shame`!


>
> TBD: 
> To be re-evaluated and checked later, so I can look at this with a fresher eye.
>

-->

Ah! Nothing wrong with the error expression, but the culprit is us rounding (casting to integer) of the *intermediate results* in the `r` calculus expression: divisions `r0` and `r1` have both been rounded down before commencing with calculating the final `r` result and this clearly is a breach of good conduct in any calculus op. ;-)

So let's see that again, now without the premature rounding:


```
a = 247
b = 31

approximation:

d0 = 32
d1 = 16

r0 = 247 / 32
r1 = 247 / 16

w = (b - d1) / (d0 - d1) = (31 - 16) / (32 - 16) = 15/16

r = w * r0 + (1 - w) * r1 = 15/16 * 247 / 32 + (1 - 15/16) * 247 / 16 
  = 15/16 * 247/32 + 1/16 * 247/16 
  = 15/16 * 247/32 + 1/16 * 2 * 247/32
  = (15 * 247 + 1 * 2 * 247 ) / (16 * 32)
  = 4199 / 512
  = 8.201172

versus

247/31 = 7.96

giving an error factor 0.2412 which is less than the worst case error bound calculated above: 

err <= 247 / 30^2 = 0.274
```

so we're good.

However, this also clearly shows we'll be looking at calculating (very) large integer intermediate values before the final shift-right op, making this *correct* approach much more costly then the one above where we injected intermediate divide-by-shift-right `floor()` ops.

Hence our error estimate is too optimistic when we do that, while it would keep both integer sizes during calculation and number of shift-left multiply ops down to a much easier manageable number.

As we are looking at speed over accuracy, the former approach is to be chosen, but we'll need to revise our error estimate for that!







 

























