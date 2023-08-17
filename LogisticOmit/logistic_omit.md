Sneaky Omitted Variable Bias-Like Effects in Logistic Regression
================
2023-08-16

## Introduction

I would like to illustrate a key way which omitted variables behave
differently in [logistic
regression](https://en.wikipedia.org/wiki/Logistic_regression) than in
[linear regression](https://en.wikipedia.org/wiki/Linear_regression).

## Our Example Data

Let’s start with a data example in [`R`](https://www.r-project.org).

``` r
x_frame <- data.frame(
  x = c(-2, 1),
  wt = 1
)
```

`x_frame` is `data.frame` with a single variable called `x`, and an
example weight or row weight called `wt`.

``` r
omitted_frame <- data.frame(
  omitted = c(-1, 1),
  wt = 1
)
```

`omitted_frame` is `data.frame` with a single variable called `omitted`,
and an example weight called `wt`.

We take the cross-product of there data frames to get every combination
of variable values, and their relative proportions (or weights) in the
joined data frame.

``` r
d <- merge(
  x_frame, 
  omitted_frame, 
  by = c())
d$wt = d$wt.x * d$wt.y
d$wt.x <- NULL
d$wt.y <- NULL
d$wt <- d$wt / sum(d$wt)
```

``` r
knitr::kable(d)
```

|   x | omitted |   wt |
|----:|--------:|-----:|
|  -2 |      -1 | 0.25 |
|   1 |      -1 | 0.25 |
|  -2 |       1 | 0.25 |
|   1 |       1 | 0.25 |

The idea is: `d` is specifying what proportion of an arbitrarily large
data set (with repeated rows) has each possible set of values. For us,
`d` is not a sample- it is an entire population. This is just a
long-winded way of trying to explain why we have row weights and why we
are not concerned with observation counts, uncertainly bars, or
significances/p-values for this example.

Let’s define a few common constants: Euler’s constant, `pi`, and `e`.

``` r
(Euler_constant <- -digamma(1))
```

    ## [1] 0.5772157

``` r
pi
```

    ## [1] 3.141593

``` r
(e <- exp(1))
```

    ## [1] 2.718282

Please remember these constants in this order for later.

``` r
c(Euler_constant, pi, e)
```

    ## [1] 0.5772157 3.1415927 2.7182818

## The Linear Case

For our example we call our outcome (or dependent variable) `y_linear`.
We say that it is exactly the following linear combination of the `x`
and `omitted` variables, plus a constant.

``` r
d$y_linear <- pi * d$x + e * d$omitted + Euler_constant
```

``` r
knitr::kable(d)
```

|   x | omitted |   wt |  y_linear |
|----:|--------:|-----:|----------:|
|  -2 |      -1 | 0.25 | -8.424252 |
|   1 |      -1 | 0.25 |  1.000527 |
|  -2 |       1 | 0.25 | -2.987688 |
|   1 |       1 | 0.25 |  6.437090 |

As we expect, linear regression can recover the constants of the linear
equation from data.

``` r
lm(
  y_linear ~ x + omitted, 
  data = d, 
  weights = d$wt, 
  )$coef
```

    ## (Intercept)           x     omitted 
    ##   0.5772157   3.1415927   2.7182818

Notice the recovered coefficients are the three constants we specified.

This is nice, and as expected.

### Omitting a Varaible

Now we ask: what happens if we omit from the model the variable named
“`omitted`”? For a linear model, we do not expect [omitted variable
bias](https://en.wikipedia.org/wiki/Omitted-variable_bias), as the
variables `x` and `omitted` are fully [statistically
independent](https://en.wikipedia.org/wiki/Independence_(probability_theory)).

We can confirm `omitted` is nice, in that it is mean-`0` and has zero
correlation with `x` under the specified data distribution.

``` r
sum(d$omitted * d$wt) / sum(d$wt)
```

    ## [1] 0

``` r
knitr::kable(
  cov.wt(
    d[, c('x', 'omitted')],
    wt = d$wt
  )$cov
)
```

|         |   x |  omitted |
|:--------|----:|---------:|
| x       |   3 | 0.000000 |
| omitted |   0 | 1.333333 |

And all of this worrying pays off. If we fit a model with the `omitted`
variable left out, we still get the original valid estimates of the
`x`-coefficient and the intercept.

``` r
lm(
  y_linear ~ x, 
  data = d,
  weights = d$wt, 
  )$coef
```

    ## (Intercept)           x 
    ##   0.5772157   3.1415927

## The Logistic Case

Let’s convert this problem to modeling the probability a new outcome
variable, called `y_observed` takes on the values `TRUE` and `FALSE`. We
use the encoding strategy from [“replicate linear and logistic
models”](https://win-vector.com/2019/07/03/replicating-a-linear-model/)
(which can simplify steps in many data science projects). How this
example arises isn’t critical, we want to investigate the properties of
this resulting data. So let’s take a moment and derive our data.

``` r
sigmoid <- function(x) {1 / (1 + exp(-x))}

d$y_probability <- sigmoid(d$y_linear)
```

``` r
d_plus <- d
d_plus$y_observed <- TRUE
d_plus$wt <- d_plus$wt * d_plus$y_probability
d_minus <- d
d_minus$y_observed <- FALSE
d_minus$wt <- d_minus$wt * (1 - d_minus$y_probability)
d_logistic <- rbind(d_plus, d_minus)
d_logistic$wt <- d_logistic$wt / sum(d_logistic$wt)
```

``` r
knitr::kable(d_logistic)
```

|   x | omitted |        wt |  y_linear | y_probability | y_observed |
|----:|--------:|----------:|----------:|--------------:|:-----------|
|  -2 |      -1 | 0.0000549 | -8.424252 |     0.0002194 | TRUE       |
|   1 |      -1 | 0.1827905 |  1.000527 |     0.7311621 | TRUE       |
|  -2 |       1 | 0.0119963 | -2.987688 |     0.0479852 | TRUE       |
|   1 |       1 | 0.2496004 |  6.437090 |     0.9984015 | TRUE       |
|  -2 |      -1 | 0.2499451 | -8.424252 |     0.0002194 | FALSE      |
|   1 |      -1 | 0.0672095 |  1.000527 |     0.7311621 | FALSE      |
|  -2 |       1 | 0.2380037 | -2.987688 |     0.0479852 | FALSE      |
|   1 |       1 | 0.0003996 |  6.437090 |     0.9984015 | FALSE      |

The point is: this data has our original coefficients encoded in it as
the coefficients of the generative process for `y_observed`. We confirm
this by fitting a logistic regression.

``` r
suppressWarnings(
  glm(
    y_observed ~ x + omitted, 
    data = d_logistic, 
    weights = d_logistic$wt, 
    family = binomial(link = "logit")
    )$coef
)
```

    ## (Intercept)           x     omitted 
    ##   0.5772151   3.1415914   2.7182800

Notice we recover our expected coefficients. We could use these inferred
coefficients to answer questions about how probability of outcomes
varies with changes in variables.

### Omitting a Variable, Again

Now, let’s try (and fail to) repeat our omitted variable experiment.

First we confirm `omitted` is mean zero and uncorrelated with our
variable `x`, even under the new data set and new row weight
distribution.

``` r
sum(d_logistic$omitted * d_logistic$wt) / sum(d_logistic$wt)
```

    ## [1] 2.022037e-17

``` r
knitr::kable(
  cov.wt(
    d_logistic[, c('x', 'omitted')],
    wt = d_logistic$wt
  )$cov
)
```

|         |        x |  omitted |
|:--------|---------:|---------:|
| x       | 2.882739 | 0.000000 |
| omitted | 0.000000 | 1.281217 |

We pass the check. But, as we will see, this doesn’t guarantee
non-entangled behavior for a logistic regression.

``` r
suppressWarnings(
  glm(
    y_observed ~ x, 
    data = d_logistic, 
    weights = d_logistic$wt, 
    family = binomial(link = "logit")
    )$coef
)
```

    ## (Intercept)           x 
    ##  0.00337503  1.85221234

Notice the new `x` coefficient is nowhere near the value we saw before.

### Explaining The Result

The bad way of interpreting our logistic experiment is:

> For a logistic model: an omitted explanatory variable can bias
> coefficient estimates. This even when the omitted explanatory variable
> is mean zero, symmetric, and uncorrelated with the other model
> explanitory variables. This differs from the situation for linear
> models.

The good way of interpreting logistic experiment is:

> For a logistic model: the correct inference for a given explanatory
> variable often depends on what other explanatory variables are present
> in the model.

That is: we didn’t get a wrong inference. We just got a different one,
as we are inferring in a different situation. The fallacy was thinking a
change in variable value has the same effect no matter what the values
of other explanatory variables are. This is not the case for logistic
regression, due to the non-linear shape of the logistic curve.

Diagrammatically what happened is the following.

![](logistic_omit_files/figure-gfm/unnamed-chunk-22-1.png)<!-- -->

In the above diagram we have the `sigmoid()`, or logistic curve. The
horizontal axis is the linear or “link space” for predictions and the
vertical axis is the probability or response space for predictions. The
curve is the transform the logistic regression’s linear or link
prediction is run through to get probabilities or responses. On this
curve we have added as dots the four different combinations of values
for `x` and `omitted` in our data set. The dots attached by lines differ
only by changes in `omitted`, i.e. those that have given value for `x`.

Without the extra variable `omitted` we can’t tell the joined pairs
apart, and we are forced to use compromise effect estimates. However,
the amount of interference is different for each value of `x`. This is a
common observation in logistic regression: you can’t tell if a variable
and coefficient has large or small effects without knowing the typical
values of the complementary explanatory variables.

## My Interpretation

You get different estimates for variables depending on what other
variables are present in a logistic model. This looks a lot like omitted
variable bias, but it is also interpretable as different column-views of
the data having fundamentally different models. A possible source of
surprise is: linear models avoid this in the special case of independent
variables.

Some care has to be taken in taking inferred logistic coefficients out
of their surrounding model context.

## Discussion Points

What are your opinions to the nature of the effect?

- Is it important in your uses of inferred coefficients?
- Do you feel it is intrinsic to the modeling process, or introduced by
  attempted interpretation?
- What is the correct value of the `x`-coefficient? in the logistic
  regression? `3.1415927` or `1.85221234`? (One may have to think of
  this as a “depends on what you are going to use it for”, or a
  [Simpson’s paradox](https://en.wikipedia.org/wiki/Simpson%27s_paradox)
  unanswerable.)

R source for this article can be found
[here](https://github.com/WinVector/Examples/tree/main/LogisticOmit).