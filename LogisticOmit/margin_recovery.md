Solving for Hidden Data
================
2023-08-25

## Introduction

Let’s continue along the lines discussed in [Omitted Variable Effects in
Logistic
Regression](https://win-vector.com/2023/08/18/omitted-variable-effects-in-logistic-regression/).

The issue is as follows. For logistic regression, omitted variables
cause estimation bias. This *even for independent variables*, which is
not the case for more familiar linear regression.

This is a known problem with known mitigations:

- Rhian Daniel, Jingjing Zhang, Daniel Farewell, “Making apples from
  oranges: Comparing noncollapsible effect estimators and their standard
  errors after adjustment for different covariate sets”, Biometrical
  Journal, (2020), DOI: 10.1002/bimj.201900297.
- John M. Neuhaus and Nicholas P. Jewell, “A Geometric Approach to
  Assess Bias Due to Omitted Covariates in Generalized Linear Models”,
  Biometrika, Vol. 80, No. 4 (Dec. 1993), pp. 807-815.
- Zhang, Zhiwei, “Estimating a Marginal Causal Odds Ratio Subject to
  Confounding”, Communications in Statistics - Theory and Methods, 38:3,
  (2009), 309-321.

(Thank you, Tom Palmer and Robert Horton for the references!)

For this note, let’s work out how to directly try and overcome the
omitted variable bias by solving for the hidden data. We will work our
example in [`R`](https://www.r-project.org). We will derive some deep
results out of a simple set-up.

## Our Example

For an example let’s set up a logistic regression on two explanatory
variables `x1` and `x2` . For simplicity we will take the case where
`x1` and `x2` only take on the values `0` and `1`.

Our data is then keyed by the values of these explanatory variables and
the dependent or outcome variable `y`. The keying looks like the
following.

| x1  | x2  | y     |
|:----|:----|:------|
| 0   | 0   | FALSE |
| 1   | 0   | FALSE |
| 0   | 1   | FALSE |
| 1   | 1   | FALSE |
| 0   | 0   | TRUE  |
| 1   | 0   | TRUE  |
| 0   | 1   | TRUE  |
| 1   | 1   | TRUE  |

### The Example Data

Let’s specify the joint probability distribution of our two explanatory
variables. We choose them as independent with the following expected
values.

``` r
# specify explanatory variable distribution 
pX1 <- 0.3
pX2 <- 0.8
```

Our data set can then be completely described by above explanatory
variable distribution *and* the conditional probability of the dependent
outcomes. For our logistic regression problem we set up our outcome
conditioning as `P[y == TRUE] ~ sigmoid(c0 + b1 * x1 + b2 * x2)`. Our
example coefficients are as follows.

``` r
# 0.5772
(c0 <- -digamma(1))
```

    ## [1] 0.5772157

``` r
# 3.1415
(b1 <- pi)
```

    ## [1] 3.141593

``` r
# 27.182
(b2 <- -3 * exp(1))
```

    ## [1] -8.154845

Please remember these coefficients in this order for later.

``` r
# show constants in an order will see again
c(c0, b1, b2)
```

    ## [1]  0.5772157  3.1415927 -8.1548455

Using the methodology of [Replicating a Linear
Model](https://win-vector.com/2019/07/03/replicating-a-linear-model/) we
can build an example data set that obeys the specified explanatory
variable distribution and has specified outcome probabilities. This is
just us building a data set matching an assumed known answer. Our data
distribution is going to be determined by `pX1`, `pX2`, and
`P[y == TRUE] ~ sigmoid(c0 + b1 * x1 + b2 * x2)`. Our inference task is
to recover the parameters `pX1`, `pX2`, `c0`, `b1`, and `b2` from data,
even in the situation where observers have omitted variable issues.

The complete detailed data is generated as follows. The `proportion`
column is what proportion of a data set drawn from this specified
distribution matches the row keys `x1`, `x2`, `y`, or is the joint
probability of a given row type. We can derive all the detailed
probabilities as follows.

``` r
# get joint distribution of explanatory variables
detailed_frame$x_distribution <- (
  (detailed_frame$x1 * pX1 + (1 - detailed_frame$x1) * (1 - pX1))
    * (detailed_frame$x2 * pX2 + (1 - detailed_frame$x2) * (1 - pX2))
)
# get conditional probability of observed outcome
y_linear <- c0 + b1 * detailed_frame$x1 + b2 * detailed_frame$x2
# converting "links" to probabilities
sigmoid <- function(x) {1 / (1 + exp(-x))}
y_probability <- sigmoid(y_linear)
# record probability of observation
detailed_frame$p_observed_outcome <- ifelse(
  detailed_frame$y, 
  y_probability, 
  1 - y_probability)
# compute joint explanatory plus outcome probability of each row
detailed_frame$proportion <- (
  detailed_frame$x_distribution * detailed_frame$p_observed_outcome)
```

The following table relates `x1`, `x2`, `y` combinations to the
`proportion` column (which shows how common each such row is).

|  x1 |  x2 | y     | x_distribution | p_observed_outcome | proportion |
|----:|----:|:------|---------------:|-------------------:|-----------:|
|   0 |   0 | FALSE |           0.14 |          0.3595735 |  0.0503403 |
|   1 |   0 | FALSE |           0.06 |          0.0236881 |  0.0014213 |
|   0 |   1 | FALSE |           0.56 |          0.9994885 |  0.5597136 |
|   1 |   1 | FALSE |           0.24 |          0.9882958 |  0.2371910 |
|   0 |   0 | TRUE  |           0.14 |          0.6404265 |  0.0896597 |
|   1 |   0 | TRUE  |           0.06 |          0.9763119 |  0.0585787 |
|   0 |   1 | TRUE  |           0.56 |          0.0005115 |  0.0002864 |
|   1 |   1 | TRUE  |           0.24 |          0.0117042 |  0.0028090 |

For a logistic regression problem, the relation between `x1`, `x2` and
`y` is encoded in the `proportion` distribution that gives the joint
expected frequency of each possible data row in a drawn sample.

### An Extra Invariant

There is an interesting non-linear invariant the `proportion` column
obeys. We will use this invariant later, so it is worth establishing.
The principle is: our solution disappears with respect to certain
test-vectors, which will help us re-identify it later.

Consider the following test vector.

``` r
test_vec <- (
  (-1)^detailed_frame$x1 
  * (-1)^detailed_frame$x2 
  * (-1)^detailed_frame$y)

test_vec
```

    ## [1]  1 -1 -1  1 -1  1  1 -1

`log(detailed_frame$proportion))` is orthogonal to this test vector.

``` r
p_vec <- test_vec * log(detailed_frame$proportion)
stopifnot(  # abort render if claim is not true
  abs(sum(p_vec)) < 1e-8)

sum(p_vec)
```

    ## [1] -2.553513e-15

This can be confirmed to always be the case by using algebra to sum-out
the `y` values and checking this expands into a sum over only constant
and linear terms.

``` r
s_vec <- -(
  (-1)^detailed_frame$x1[1:4] 
  * (-1)^detailed_frame$x2[1:4] 
  * (c0 + b1 * detailed_frame$x1[1:4] + b2 * detailed_frame$x2[1:4]))

s_vec
```

    ## [1] -0.5772157  3.7188083 -7.5776298  4.4360372

The above indeed is the terms where the two different `y` outcomes have
been combined. This is a consequence of the linear structure of the
logit of the logistic regression prediction.

``` r
p_comb <- p_vec[1:4] + p_vec[5:8]
stopifnot(  # abort render if claim is not true
  abs(s_vec - p_comb) < 1e-8)

p_comb
```

    ## [1] -0.5772157  3.7188083 -7.5776298  4.4360372

A sum over varying signs, such as above, will annihilate constant and
linear terms. So `s_vec` must always sum to zero, thus `p_vec` must also
always sum to zero. Roughly this is a check the model has no
interactions, or models without interactions pass this check. We are
using two very strong assumptions on `x1` and `x2`: they are independent
*and* the modeled probabilities don’t contain an `x1` and `x2`
interaction.

### Inferring From Fully Observed Data

We can confirm this data set encodes the expected logistic relationship
by recovering the coefficients through fitting.

``` r
# suppressWarnings() to avoid fractional data weight complaint
correct_coef <- suppressWarnings(
  glm(
    y ~ x1 + x2,
    data = detailed_frame,
    weights = detailed_frame$proportion,
    family = binomial()
  )$coef
)

correct_coef
```

    ## (Intercept)          x1          x2 
    ##   0.5772157   3.1415927  -8.1548455

Notice we recover the
`c0 + b1 * detailed_frame$x1 + b2 * detailed_frame$x2` form.

## The Problem

Now let’s get to our issue. Suppose we have two experimenters, each of
which only observing one of the explanatory variables. As we saw in
[Omitted Variable Effects in Logistic
Regression](https://win-vector.com/2023/08/18/omitted-variable-effects-in-logistic-regression/)
each of these experimenters will in fact estimate coefficients that are
**biased towards zero**, due to the non-collapsibility of the modeling
set up. This differs from linear regression where for independent
explanatory variables (as we have here) we would expect each
experimenter to be able to get an unbiased estimate of the coefficient
for the explanatory variable available to them!

To show this let’s build a linear operator that computes the margins the
experimenters actually observe.

<table class="table" style="font-size: 10px; margin-left: auto; margin-right: auto;">
<thead>
<tr>
<th style="text-align:left;">
</th>
<th style="text-align:right;">
p(0,0,FALSE)
</th>
<th style="text-align:right;">
p(1,0,FALSE)
</th>
<th style="text-align:right;">
p(0,1,FALSE)
</th>
<th style="text-align:right;">
p(1,1,FALSE)
</th>
<th style="text-align:right;">
p(0,0,TRUE)
</th>
<th style="text-align:right;">
p(1,0,TRUE)
</th>
<th style="text-align:right;">
p(0,1,TRUE)
</th>
<th style="text-align:right;">
p(1,1,TRUE)
</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left;">
p(0,&ast;,FALSE)
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
</tr>
<tr>
<td style="text-align:left;">
p(1,&ast;,FALSE)
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
</tr>
<tr>
<td style="text-align:left;">
p(0,&ast;,TRUE)
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
</tr>
<tr>
<td style="text-align:left;">
p(1,&ast;,TRUE)
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
</tr>
<tr>
<td style="text-align:left;">
p(&ast;,0,FALSE)
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
</tr>
<tr>
<td style="text-align:left;">
p(&ast;,1,FALSE)
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
</tr>
<tr>
<td style="text-align:left;">
p(&ast;,0,TRUE)
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
</tr>
<tr>
<td style="text-align:left;">
p(&ast;,1,TRUE)
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
1
</td>
</tr>
<tr>
<td style="text-align:left;">
p(0,0,&ast;)
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
</tr>
<tr>
<td style="text-align:left;">
p(1,0,&ast;)
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
</tr>
<tr>
<td style="text-align:left;">
p(0,1,&ast;)
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
</tr>
<tr>
<td style="text-align:left;">
p(1,1,&ast;)
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
</tr>
</tbody>
</table>

The above matrix linearly maps our earlier `proportions` columns to
various interesting roll-ups or aggregations.

``` r
# apply the linear operator to compute marginalized observations
proportion <- margin_transform %*% detailed_frame$proportion
```

<table>
<thead>
<tr>
<th style="text-align:left;">
</th>
<th style="text-align:left;">
x1
</th>
<th style="text-align:left;">
x2
</th>
<th style="text-align:left;">
y
</th>
<th style="text-align:right;">
proportion
</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left;">
p(0,&ast;,FALSE)
</td>
<td style="text-align:left;">
0
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:left;">
FALSE
</td>
<td style="text-align:right;">
0.6100538
</td>
</tr>
<tr>
<td style="text-align:left;">
p(1,&ast;,FALSE)
</td>
<td style="text-align:left;">
1
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:left;">
FALSE
</td>
<td style="text-align:right;">
0.2386123
</td>
</tr>
<tr>
<td style="text-align:left;">
p(0,&ast;,TRUE)
</td>
<td style="text-align:left;">
0
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:left;">
TRUE
</td>
<td style="text-align:right;">
0.0899462
</td>
</tr>
<tr>
<td style="text-align:left;">
p(1,&ast;,TRUE)
</td>
<td style="text-align:left;">
1
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:left;">
TRUE
</td>
<td style="text-align:right;">
0.0613877
</td>
</tr>
<tr>
<td style="text-align:left;">
p(&ast;,0,FALSE)
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:left;">
0
</td>
<td style="text-align:left;">
FALSE
</td>
<td style="text-align:right;">
0.0517616
</td>
</tr>
<tr>
<td style="text-align:left;">
p(&ast;,1,FALSE)
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:left;">
1
</td>
<td style="text-align:left;">
FALSE
</td>
<td style="text-align:right;">
0.7969046
</td>
</tr>
<tr>
<td style="text-align:left;">
p(&ast;,0,TRUE)
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:left;">
0
</td>
<td style="text-align:left;">
TRUE
</td>
<td style="text-align:right;">
0.1482384
</td>
</tr>
<tr>
<td style="text-align:left;">
p(&ast;,1,TRUE)
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:left;">
1
</td>
<td style="text-align:left;">
TRUE
</td>
<td style="text-align:right;">
0.0030954
</td>
</tr>
<tr>
<td style="text-align:left;">
p(0,0,&ast;)
</td>
<td style="text-align:left;">
0
</td>
<td style="text-align:left;">
0
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:right;">
0.1400000
</td>
</tr>
<tr>
<td style="text-align:left;">
p(1,0,&ast;)
</td>
<td style="text-align:left;">
1
</td>
<td style="text-align:left;">
0
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:right;">
0.0600000
</td>
</tr>
<tr>
<td style="text-align:left;">
p(0,1,&ast;)
</td>
<td style="text-align:left;">
0
</td>
<td style="text-align:left;">
1
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:right;">
0.5600000
</td>
</tr>
<tr>
<td style="text-align:left;">
p(1,1,&ast;)
</td>
<td style="text-align:left;">
1
</td>
<td style="text-align:left;">
1
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:right;">
0.2400000
</td>
</tr>
</tbody>
</table>

The above margin frame describes how the detailed experiment is
marginalized or censored down to different experimenters see. In our
set-up experimenter 1 sees only the first four rows, and experimenter 2
sees only the next 4 rows. We consider the rest of the data
“unobserved”.

### Experimenter 1’s view

Let’s see what happens when an experimenter tries to perform inference
on their fraction of the data.

``` r
# select data available to d1
d1 <- margin_frame[
  margin_frame$x2 == asterisk_symbol, , drop = FALSE]

knitr::kable(d1)
```

<table>
<thead>
<tr>
<th style="text-align:left;">
</th>
<th style="text-align:left;">
x1
</th>
<th style="text-align:left;">
x2
</th>
<th style="text-align:left;">
y
</th>
<th style="text-align:right;">
proportion
</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left;">
p(0,&ast;,FALSE)
</td>
<td style="text-align:left;">
0
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:left;">
FALSE
</td>
<td style="text-align:right;">
0.6100538
</td>
</tr>
<tr>
<td style="text-align:left;">
p(1,&ast;,FALSE)
</td>
<td style="text-align:left;">
1
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:left;">
FALSE
</td>
<td style="text-align:right;">
0.2386123
</td>
</tr>
<tr>
<td style="text-align:left;">
p(0,&ast;,TRUE)
</td>
<td style="text-align:left;">
0
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:left;">
TRUE
</td>
<td style="text-align:right;">
0.0899462
</td>
</tr>
<tr>
<td style="text-align:left;">
p(1,&ast;,TRUE)
</td>
<td style="text-align:left;">
1
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:left;">
TRUE
</td>
<td style="text-align:right;">
0.0613877
</td>
</tr>
</tbody>
</table>

``` r
# solve from d1's point of view
d1_est <- suppressWarnings(
  glm(
    y ~ x1,
    data = d1,
    weights = d1$proportion,
    family = binomial()
  )$coef
)

d1_est
```

    ## (Intercept)          x1 
    ##  -1.9143360   0.5567057

Notice experimenter 1 got a *much* too small estimate of the `x1`
coefficient of 0.5567057, whereas the correct value is 3.1415927. From
experimenter 1’s point of view, the effect of the omitted variable `x2`
is making `x1` hard to correctly infer.

### Experimenter 2’s view

Experimenter 2 has the following portion of data, which also is not
enough to get an unbiased coefficient estimate.

``` r
# select data available to d2
d2 <- margin_frame[
  margin_frame$x1 == asterisk_symbol, , drop = FALSE]

knitr::kable(d2)
```

<table>
<thead>
<tr>
<th style="text-align:left;">
</th>
<th style="text-align:left;">
x1
</th>
<th style="text-align:left;">
x2
</th>
<th style="text-align:left;">
y
</th>
<th style="text-align:right;">
proportion
</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left;">
p(&ast;,0,FALSE)
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:left;">
0
</td>
<td style="text-align:left;">
FALSE
</td>
<td style="text-align:right;">
0.0517616
</td>
</tr>
<tr>
<td style="text-align:left;">
p(&ast;,1,FALSE)
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:left;">
1
</td>
<td style="text-align:left;">
FALSE
</td>
<td style="text-align:right;">
0.7969046
</td>
</tr>
<tr>
<td style="text-align:left;">
p(&ast;,0,TRUE)
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:left;">
0
</td>
<td style="text-align:left;">
TRUE
</td>
<td style="text-align:right;">
0.1482384
</td>
</tr>
<tr>
<td style="text-align:left;">
p(&ast;,1,TRUE)
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:left;">
1
</td>
<td style="text-align:left;">
TRUE
</td>
<td style="text-align:right;">
0.0030954
</td>
</tr>
</tbody>
</table>

### A Critique

From the original data set’s point of view: both experimenters have
wrong estimates of their respective coefficients. The question then is:
if the experimenters pool their effort can they infer the correct
coefficients? From our point of view only `d1` plus `d2` are observable,
even after the experimenters choose to collaborate. This is because we
are assuming neither of them had access to the original data as each
failed to measure one of the explanatory variables.

## The Solution Strategy

Each experimenter knows a lot about the data. They known the
distribution of their explanatory variable, and even the joint
distribution of their explanatory and the dependent and outcome data.
Assuming the two explanatory variables are independent, they even know
the joint distribution of the explanatory variables. We will show how to
use their combined observations to estimate the hidden data elements.
This data can then be used for standard detailed analysis, like we
showed on the original full data set.

This isn’t the first time we have proposed a “guess at the original
data, as it wasn’t shared” as we played with this in [Checking claims in
published statistics
papers](https://win-vector.com/2013/04/08/checking-claims-in-published-statistics-papers/).

## Solution Steps

Our solutions strategy is as follows:

- We will estimate the joint distribution of `x1` and `x2` from the
  observed marginal distributions of `x1` and `x2` plus an assumption of
  independence.
- We will plug the above and other details in to the inverse of
  `margin_transform` to get an estimate of the original hidden data.
- We will perform inference on this data to get coefficient estimates.

### Estimating the `x1` and `x2` joint distribution

Neither experimenter observed the following part of the marginal frame:

``` r
# show x1 x2 distribution poriton of margin_frame
dx <- margin_frame[
  margin_frame$y == asterisk_symbol, , drop = FALSE]

knitr::kable(dx)
```

<table>
<thead>
<tr>
<th style="text-align:left;">
</th>
<th style="text-align:left;">
x1
</th>
<th style="text-align:left;">
x2
</th>
<th style="text-align:left;">
y
</th>
<th style="text-align:right;">
proportion
</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left;">
p(0,0,&ast;)
</td>
<td style="text-align:left;">
0
</td>
<td style="text-align:left;">
0
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:right;">
0.14
</td>
</tr>
<tr>
<td style="text-align:left;">
p(1,0,&ast;)
</td>
<td style="text-align:left;">
1
</td>
<td style="text-align:left;">
0
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:right;">
0.06
</td>
</tr>
<tr>
<td style="text-align:left;">
p(0,1,&ast;)
</td>
<td style="text-align:left;">
0
</td>
<td style="text-align:left;">
1
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:right;">
0.56
</td>
</tr>
<tr>
<td style="text-align:left;">
p(1,1,&ast;)
</td>
<td style="text-align:left;">
1
</td>
<td style="text-align:left;">
1
</td>
<td style="text-align:left;">
&ast;
</td>
<td style="text-align:right;">
0.24
</td>
</tr>
</tbody>
</table>

However, under the independence assumption they can estimate it from
their pooled observations as follows.

``` r
# estimate x1 x2 distribution from d1 and d2
d1a <- aggregate(proportion ~ x1, data = d1, sum)
d2a <- aggregate(proportion ~ x2, data = d2, sum)
dxe <- merge(d1a, d2a, by = c())
dxe["proportion"] <- dxe$proportion.x * dxe$proportion.y
dxe$proportion.x <- NULL
dxe$proportion.y <- NULL
dxe <- dxe[order(dxe$x2, dxe$x1), , drop = FALSE]

knitr::kable(dxe)
```

<table>
<thead>
<tr>
<th style="text-align:right;">
x1
</th>
<th style="text-align:right;">
x2
</th>
<th style="text-align:right;">
proportion
</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.14
</td>
</tr>
<tr>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0.06
</td>
</tr>
<tr>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.56
</td>
</tr>
<tr>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.24
</td>
</tr>
</tbody>
</table>

Notice `dxe` is build only from `dx1` and `dx2` (plus the assumed
independence of `x1` and `x2`). At this point we have inferred the `pX1`
and `pX2` parameters from the observed data.

### Combining Observations

We new combine all of our known data to get an estimate of the
(unobserved) summaries produced by `margin_transform`.

``` r
# put together experimenter 1 and 2's joint estimate of marginal proportions
# from data they have in their sub-experiments.
estimated_proportions <- c(
  d1$proportion,
  d2$proportion,
  dxe$proportion
)

estimated_proportions
```

    ##  [1] 0.610053847 0.238612287 0.089946153 0.061387713 0.051761580 0.796904554
    ##  [7] 0.148238420 0.003095446 0.140000000 0.060000000 0.560000000 0.240000000

### Solving For the Full Joint Distribution

We use linear algebra to pull `estimated_proportions` back through
`margin_transform` inverse to get a linear estimate of the unobserved
original data.

``` r
# typical solution (in the linear sense, signs not enforced)
v <- solve(qr(margin_transform, LAPACK = TRUE), estimated_proportions)

v
```

    ## p(0,0,FALSE) p(1,0,FALSE) p(0,1,FALSE) p(1,1,FALSE)  p(0,0,TRUE)  p(1,0,TRUE) 
    ##  0.047964126  0.003797454  0.562089720  0.234814833  0.092035874  0.056202546 
    ##  p(0,1,TRUE)  p(1,1,TRUE) 
    ## -0.002089720  0.005185167

Note this estimate has negative entries, so is not yet a sequence of
valid frequencies or probabilities. We will correct this by adding
elements that don’t change the forward mapping under `margin_transform`.
In linear algebra this means we need a basis for `margin_transform`’s
“null space.” This is gotten as follows.

``` r
# our degree of freedom between solutions
ns <- MASS::Null(t(margin_transform))  # also uses QR decomposition, could combine
```

``` r
ns <- ns / (mean(abs(ns)) * ifelse(ns[[1]] >= 0, 1, -1))

ns
```

    ## [1]  1 -1 -1  1 -1  1  1 -1

All valid solutions are of the form `v + z * ns` for scalars `z`. In
fact all solutions are some interval of `z` values. We can solve for
this interval.

Our attempted recovered solutions to the (unknown to either
experimenter!) original data distribution details can be seen below.

<table>
<thead>
<tr>
<th style="text-align:right;">
x1
</th>
<th style="text-align:right;">
x2
</th>
<th style="text-align:left;">
y
</th>
<th style="text-align:right;">
proportion
</th>
<th style="text-align:right;">
recovered_1
</th>
<th style="text-align:right;">
recovered_2
</th>
<th style="text-align:right;">
null vector
</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:left;">
FALSE
</td>
<td style="text-align:right;">
0.0503403
</td>
<td style="text-align:right;">
0.0500538
</td>
<td style="text-align:right;">
0.0517616
</td>
<td style="text-align:right;">
1
</td>
</tr>
<tr>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:left;">
FALSE
</td>
<td style="text-align:right;">
0.0014213
</td>
<td style="text-align:right;">
0.0017077
</td>
<td style="text-align:right;">
0.0000000
</td>
<td style="text-align:right;">
-1
</td>
</tr>
<tr>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:left;">
FALSE
</td>
<td style="text-align:right;">
0.5597136
</td>
<td style="text-align:right;">
0.5600000
</td>
<td style="text-align:right;">
0.5582923
</td>
<td style="text-align:right;">
-1
</td>
</tr>
<tr>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:left;">
FALSE
</td>
<td style="text-align:right;">
0.2371910
</td>
<td style="text-align:right;">
0.2369046
</td>
<td style="text-align:right;">
0.2386123
</td>
<td style="text-align:right;">
1
</td>
</tr>
<tr>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:left;">
TRUE
</td>
<td style="text-align:right;">
0.0896597
</td>
<td style="text-align:right;">
0.0899462
</td>
<td style="text-align:right;">
0.0882384
</td>
<td style="text-align:right;">
-1
</td>
</tr>
<tr>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:left;">
TRUE
</td>
<td style="text-align:right;">
0.0585787
</td>
<td style="text-align:right;">
0.0582923
</td>
<td style="text-align:right;">
0.0600000
</td>
<td style="text-align:right;">
1
</td>
</tr>
<tr>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:left;">
TRUE
</td>
<td style="text-align:right;">
0.0002864
</td>
<td style="text-align:right;">
0.0000000
</td>
<td style="text-align:right;">
0.0017077
</td>
<td style="text-align:right;">
1
</td>
</tr>
<tr>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:left;">
TRUE
</td>
<td style="text-align:right;">
0.0028090
</td>
<td style="text-align:right;">
0.0030954
</td>
<td style="text-align:right;">
0.0013877
</td>
<td style="text-align:right;">
-1
</td>
</tr>
</tbody>
</table>

As we can see these two extreme solutions are in fact actually fairly
close.

### Picking a point-estimate

The standard trick with an under-specified system is to add an
objective. A great choice is: maximize the entropy of (or flatness of)
the distribution we are solving for.

This works as follows.

``` r
entropy <- function(v) {
  v <- v[v > 0]
  v <- v / sum(v)
  -sum(v * log2(v))
}
```

``` r
# brute force solve for maximum entropy mix
# obviously this can be done a bit slicker
opt_soln <- optimize(
  function(z) {
    entropy(
      z * detailed_frame$recovered_1 + 
        (1 - z) * detailed_frame$recovered_2)},
  c(0, 1),
  maximum = TRUE)

z_opt <- opt_soln$maximum
detailed_frame["maxent_dist"] <- (
  z_opt * detailed_frame$recovered_1 + 
    (1 - z_opt) * detailed_frame$recovered_2)
```

Notice that the recovered `maxent_dist` *is* the unobserved original
`proportion`.

<table>
<thead>
<tr>
<th style="text-align:right;">
x1
</th>
<th style="text-align:right;">
x2
</th>
<th style="text-align:left;">
y
</th>
<th style="text-align:right;">
proportion
</th>
<th style="text-align:right;">
maxent_dist
</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:left;">
FALSE
</td>
<td style="text-align:right;">
0.0503403
</td>
<td style="text-align:right;">
0.0503403
</td>
</tr>
<tr>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:left;">
FALSE
</td>
<td style="text-align:right;">
0.0014213
</td>
<td style="text-align:right;">
0.0014213
</td>
</tr>
<tr>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:left;">
FALSE
</td>
<td style="text-align:right;">
0.5597136
</td>
<td style="text-align:right;">
0.5597135
</td>
</tr>
<tr>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:left;">
FALSE
</td>
<td style="text-align:right;">
0.2371910
</td>
<td style="text-align:right;">
0.2371910
</td>
</tr>
<tr>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:left;">
TRUE
</td>
<td style="text-align:right;">
0.0896597
</td>
<td style="text-align:right;">
0.0896597
</td>
</tr>
<tr>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0
</td>
<td style="text-align:left;">
TRUE
</td>
<td style="text-align:right;">
0.0585787
</td>
<td style="text-align:right;">
0.0585787
</td>
</tr>
<tr>
<td style="text-align:right;">
0
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:left;">
TRUE
</td>
<td style="text-align:right;">
0.0002864
</td>
<td style="text-align:right;">
0.0002865
</td>
</tr>
<tr>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:left;">
TRUE
</td>
<td style="text-align:right;">
0.0028090
</td>
<td style="text-align:right;">
0.0028090
</td>
</tr>
</tbody>
</table>

And these are our estimated coefficients.

``` r
recovered_coef <- suppressWarnings(
  glm(
    y ~ x1 + x2,
    data = detailed_frame,
    weights = detailed_frame[["maxent_dist"]],
    family = binomial()
  )$coef
)

recovered_coef
```

    ## (Intercept)          x1          x2 
    ##   0.5772157   3.1415927  -8.1548455

This matches the correct (c0=0.5772, b1=3.1416, b2=-8.1548). We have
correctly inferred the actual coefficient values from the observed data.
I.e. we have removed the bias.

### Why the Maximum Entropy Solution is So Good

We have the direction we are varying (`ns`) before, it is `test_vec`! So
we know the actual solution is orthogonal to `ns`. Some calculus will
show us in our case the entropy is maximized where the gradient is zero,
and that this gradient being zero is the same condition as our
distribution being orthogonal to `ns`. So the maximum entropy condition
is enforcing the “no interactions” invariant we commented on earlier.

The funny thing is, we don’t have to know exactly what the maximum
entropy objective was doing to actually benefit from it. It tends to be
a helpful objective in modeling.

## Conclusion

By pooling observations we can recover a good estimate of a joint
analysis on data that was not available to us. The strategy is: try to
estimate plausible pre-images of the data that formed the observations,
and then analyze that. This in fact gives us a method to invert the bias
introduced by the omitted variables in logistic regression.

In machine learning the [maximum entropy
principle](https://en.wikipedia.org/wiki/Principle_of_maximum_entropy)
plays the role that the [stationary-action principle
action](https://en.wikipedia.org/wiki/Stationary-action_principle) plays
in classic mechanics. While *nature* isn’t forced to put equal
probabilities on different states, deterministic models must put equal
probabilities on indistinguishable states. And, [maximum entropy
modeling is very related to logistic regresion
modeling](https://win-vector.com/2011/09/23/the-equivalence-of-logistic-regression-and-maximum-entropy-models/).

In the real world we would at best be looking at marginalizations of
different draws of related data. So we would not have exact matches we
can invert- but instead would have to estimate low-discrepancy
pre-images of the data. And, as we are now introducing a lot of
unobserved parameters, we could go to Bayesian graphical model methods
to sum this all out (instead of proposing a specific point-wise
heuristic as we did here).

## Links

The source code of this article is available
[here](https://github.com/WinVector/Examples/blob/main/LogisticOmit/margin_recovery.Rmd)
(plus render
[here](https://github.com/WinVector/Examples/blob/main/LogisticOmit/margin_recovery.md)).