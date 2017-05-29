[![Build Status](https://travis-ci.org/jamesdunham/dgo.svg?branch=master)](https://travis-ci.org/jamesdunham/dgo) [![Build status](https://ci.appveyor.com/api/projects/status/1ta36kmoqen98k87?svg=true)](https://ci.appveyor.com/project/jamesdunham/dgo) [![codecov](https://codecov.io/gh/jamesdunham/dgo/branch/master/graph/badge.svg)](https://codecov.io/gh/jamesdunham/dgo)

dgo is an R package for the dynamic estimation of group-level opinion. The package can be used to estimate subpopulation groups' average latent conservatism (or other latent trait) from individuals' responses to dichotomous questions using a Bayesian group-level IRT approach developed by [Caughey and Warshaw 2015](http://pan.oxfordjournals.org/content/early/2015/02/04/pan.mpu021.full.pdf+html) that models latent traits at the level of demographic and/or geographic groups rather than individuals. This approach uses a hierarchical model to borrow strength cross-sectionally and dynamic linear models to do so across time. The group-level estimates can be weighted to generate estimates for geographic units, such as states.

dgo can also be used to estimate smoothed estimates of subpopulation groups' average responses on individual survey questions using a dynamic multi-level regression and poststratification (MRP) model ([Park, Gelman, and Bafumi 2004](http://stat.columbia.edu/~gelman/research/published/StateOpinionsNationalPolls.050712.dkp.pdf)). For instance, it could be used to estimate public opinion in each state on same-sex marriage or the Affordable Care Act.

This model opens up new areas of research on historical public opinion in the United States at the subnational level. It also enables scholars of comparative politics to estimate dynamic models of public opinion opinion at the country or subnational level.

Installation
============

dgo requires a working installation of [RStan](http://mc-stan.org/interfaces/rstan.html). If you don't have already have RStan, follow its "[Getting Started](https://github.com/stan-dev/rstan/wiki/RStan-Getting-Started)" guide before continuing.

dgo can be installed from [GitHub](https://github.com/jamesdunham/dgo) using [devtools](https://github.com/hadley/devtools/):

``` r
if (!require(devtools, quietly = TRUE)) install.packages("devtools")
devtools::install_github("jamesdunham/dgo")
```

Getting started
===============

``` r
library(dgo)
```

The minimal workflow from raw data to estimation is:

1.  shape input data using the `shape` function; and
2.  pass the result to the `dgirt` function to estimate a latent trait (e.g., conservatism) or `dgmrp` function to estimate opinion on a single survey question.

### Set RStan options

These are RStan's recommended options on a local, multicore machine with excess RAM:

``` r
rstan_options(auto_write = TRUE)
options(mc.cores = parallel::detectCores())
```

Abortion Attitudes
------------------

### Prepare input data with `shape`

DGIRT models are *dynamic*, so we need to specify which variable in the data represents time. They are also *group-level*, with groups defined by one variable for respondents' local geographic area and one or more variables for respondent characteristics.

The `time_filter` and `geo_filter` arguments optionally subset the data. Finally, `shape` requires the names of the survey identifier and survey weight variables in the data.

``` r
dgirt_in_abortion <- shape(opinion,
                  item_names = "abortion",
                  time_name = "year",
                  geo_name = "state",
                  group_names = "race3",
                  geo_filter = c("CA", "GA", "LA", "MA"),
                  id_vars = "source")
#> Applying restrictions, pass 1...
#>  Dropped 5 rows for missingness in covariates
#>  Dropped 633 rows for lacking item responses
#> Applying restrictions, pass 2...
#>  No changes
```

The reshaped and subsetted data can be summarized in a few ways before model fitting.

``` r
summary(dgirt_in_abortion)
#> Items:
#> [1] "abortion"
#> Respondents:
#>    23,007 in `item_data`
#> Grouping variables:
#> [1] "year"  "state" "race3"
#> Time periods:
#> [1] 2006 2007 2008 2009 2010
#> Local geographic areas:
#> [1] "CA" "GA" "LA" "MA"
#> Hierarchical parameters:
#> [1] "GA"         "LA"         "MA"         "race3other" "race3white"
#> Modifiers of hierarchical parameters:
#> NULL
#> Constants:
#>  Q  T  P  N  G  H  D 
#>  1  5  5 60 12  1  1
```

Response counts by state:

``` r
get_n(dgirt_in_abortion, by = c("state"))
#>    state     n
#> 1:    CA 14248
#> 2:    GA  4547
#> 3:    LA  1658
#> 4:    MA  2554
```

Response counts by item-year:

``` r
get_item_n(dgirt_in_abortion, by = "year")
#>    year abortion
#> 1: 2006     5275
#> 2: 2007     1690
#> 3: 2008     4697
#> 4: 2009     2141
#> 5: 2010     9204
```

### Fit a model with `dgirt` or `dgmrp`

`dgirt` and `dgmrp` fit estimation models to data from `shape`. `dgirt` can be used to estimate a latent variable based on responses to multiple survey questions (e.g., latent policy conservatism), while `dgmrp` can be used to estimate public opinion on an individual survey question (e.g., abortion) using a dynamic multi-level regression and post-stratification (MRP) model. In this case, we use `dgmrp` to model abortion attitudes.

Under the hood, these functions use RStan for MCMC sampling, and arguments can be passed to RStan's `stan` via the `...` argument of `dgirt` and `dgmrp`. This will almost always be desirable, at a minimum to specify the number of sampler iterations, chains, and cores.

``` r
dgmrp_out_abortion <- dgmrp(dgirt_in_abortion, iter = 1500, chains = 4, cores =
  4, seed = 42)
```

The model results are held in a `dgirtfit` object. Methods from RStan like `extract` are available if needed because `dgirtfit` is a subclass of `stanfit`. But dgo provides its own methods for typical post-estimation tasks.

### Work with `dgirt` or `dgmrp` results

For a high-level summary of the result, use `summary`.

``` r
summary(dgmrp_out_abortion)
#> dgirt samples from 4 chains of 1500 iterations, 750 warmup, thinned every 1 
#>   Drawn Sun May 28 17:20:32 2017 
#>   Package version 0.2.9 
#>   Model version 2017_01_04_singleissue 
#>   117 parameters; 60 theta_bars (year state race3)
#>   5 periods 2006 to 2010 
#> 
#> n_eff
#>    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
#>   95.68  242.50  451.87  685.85  927.30 3000.00
#> 
#> Rhat
#>    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
#>  0.9993  1.0028  1.0068  1.0081  1.0126  1.0406
#> 
#> Elapsed time
#>    chain warmup sample total
#> 1:     1    15S    16S   31S
#> 2:     2    15S    12S   27S
#> 3:     3    15S    18S   33S
#> 4:     4    16S    11S   27S
```

To summarize posterior samples, use `summarize`. The default output gives summary statistics for the `theta_bar` parameters, which represent the mean of the latent outcome for the groups defined by time, local geographic area, and the demographic characteristics specified in the earlier call to `shape`.

``` r
head(summarize(dgmrp_out_abortion))
#>        param state race3 year      mean         sd    median     q_025
#> 1: theta_bar    CA black 2006 0.7739283 0.02098019 0.7749083 0.7307567
#> 2: theta_bar    CA black 2007 0.7980027 0.02771553 0.7979378 0.7439328
#> 3: theta_bar    CA black 2008 0.7232980 0.02362116 0.7231930 0.6786121
#> 4: theta_bar    CA black 2009 0.6863666 0.02128237 0.6863458 0.6463628
#> 5: theta_bar    CA black 2010 0.7407779 0.01682742 0.7414667 0.7058706
#> 6: theta_bar    CA other 2006 0.7347199 0.02322850 0.7354365 0.6872140
#>        q_975
#> 1: 0.8144084
#> 2: 0.8517811
#> 3: 0.7693334
#> 4: 0.7279652
#> 5: 0.7717651
#> 6: 0.7790182
```

Alternatively, `summarize` can apply arbitrary functions to posterior samples for whatever parameter is given by its `pars` argument. Enclose function names with quotes. For convenience, `"q_025"` and `"q_975"` give the 2.5th and 97.5th posterior quantiles.

``` r
summarize(dgmrp_out_abortion, pars = "xi", funs = "var")
#>    param year        var
#> 1:    xi 2006 0.01814362
#> 2:    xi 2007 0.05026942
#> 3:    xi 2008 0.05606188
#> 4:    xi 2009 0.04857038
#> 5:    xi 2010 0.04149793
```

To access posterior samples in tabular form use `as.data.frame`. By default, this method returns post-warmup samples for the `theta_bar` parameters, but like other methods takes a `pars` argument.

``` r
head(as.data.frame(dgmrp_out_abortion))
#>        param state race3 year iteration     value
#> 1: theta_bar    CA black 2006         1 0.7661626
#> 2: theta_bar    CA black 2006         2 0.7690362
#> 3: theta_bar    CA black 2006         3 0.7656257
#> 4: theta_bar    CA black 2006         4 0.7935372
#> 5: theta_bar    CA black 2006         5 0.7544080
#> 6: theta_bar    CA black 2006         6 0.7819740
```

To poststratify the results use `poststratify`. The following example uses the group population proportions bundled as `annual_state_race_targets` to reweight and aggregate estimates to strata defined by state-years.

Read `help("poststratify")` for more details.

``` r
poststratify(dgmrp_out_abortion, annual_state_race_targets, strata_names =
  c("state", "year"), aggregated_names = "race3")
#>     state year     value
#>  1:    CA 2006 0.7187353
#>  2:    CA 2007 0.7469064
#>  3:    CA 2008 0.6562966
#>  4:    CA 2009 0.6272075
#>  5:    CA 2010 0.6754691
#>  6:    GA 2006 0.6339750
#>  7:    GA 2007 0.6225482
#>  8:    GA 2008 0.5232615
#>  9:    GA 2009 0.5095145
#> 10:    GA 2010 0.5705449
#> 11:    LA 2006 0.5266416
#> 12:    LA 2007 0.4769044
#> 13:    LA 2008 0.4142786
#> 14:    LA 2009 0.3985367
#> 15:    LA 2010 0.4229707
#> 16:    MA 2006 0.7629194
#> 17:    MA 2007 0.8099707
#> 18:    MA 2008 0.7058450
#> 19:    MA 2009 0.6624888
#> 20:    MA 2010 0.7078342
```

To plot the results use `dgirt_plot`. This method plots summaries of posterior samples by time period. By default, it shows a 95% credible interval around posterior medians for the `theta_bar` parameters, for each local geographic area. For this (unconverged) toy example we omit the CIs.

``` r
dgirt_plot(dgmrp_out_abortion, y_min = NULL, y_max = NULL)
```

![](https://raw.githubusercontent.com/jamesdunham/dgo/master/README/dgmrp_plot-1.png)

Output from `dgirt_plot` can be customized to some extent using objects from the ggplot2 package.

``` r
dgirt_plot(dgmrp_out_abortion, y_min = NULL, y_max = NULL) + theme_classic()
```

![](https://raw.githubusercontent.com/jamesdunham/dgo/master/README/dgmrp_plot_plus-1.png)

`dgirt_plot` can also plot the `data.frame` output from `poststratify`. This requires arguments that identify the relevant variables in the `data.frame`. Below, `poststratify` aggregates over the demographic grouping variable `race3`, resulting in a `data.frame` of estimates by state-year. So, in the subsequent call to `dgirt_plot`, we pass the names of the state and year variables. The `group_names` argument is `NULL` because there are no grouping variables left after aggregating over `race3`.

``` r
ps <- poststratify(dgmrp_out_abortion, annual_state_race_targets, strata_names =
  c("state", "year"), aggregated_names = "race3")
head(ps)
#>    state year     value
#> 1:    CA 2006 0.7187353
#> 2:    CA 2007 0.7469064
#> 3:    CA 2008 0.6562966
#> 4:    CA 2009 0.6272075
#> 5:    CA 2010 0.6754691
#> 6:    GA 2006 0.6339750
dgirt_plot(ps, group_names = NULL, time_name = "year", geo_name = "state")
```

![](https://raw.githubusercontent.com/jamesdunham/dgo/master/README/dgmrp_plot_ps-1.png)

Policy Liberalism
-----------------

### Prepare input data with `shape`

``` r
dgirt_in_liberalism <- shape(opinion, item_names = c("abortion",
    "affirmative_action","stemcell_research" , "gaymarriage_amendment",
    "partialbirth_abortion") , time_name = "year", geo_name = "state",
  group_names = "race3", geo_filter = c("CA", "GA", "LA", "MA"))
#> Applying restrictions, pass 1...
#>  Dropped 5 rows for missingness in covariates
#>  Dropped 8 rows for lacking item responses
#> Applying restrictions, pass 2...
#>  No changes
```

The reshaped and subsetted data can be summarized in a few ways before model fitting.

``` r
summary(dgirt_in_liberalism)
#> Items:
#> [1] "abortion"              "affirmative_action"    "gaymarriage_amendment"
#> [4] "partialbirth_abortion" "stemcell_research"    
#> Respondents:
#>    23,632 in `item_data`
#> Grouping variables:
#> [1] "year"  "state" "race3"
#> Time periods:
#> [1] 2006 2007 2008 2009 2010
#> Local geographic areas:
#> [1] "CA" "GA" "LA" "MA"
#> Hierarchical parameters:
#> [1] "GA"         "LA"         "MA"         "race3other" "race3white"
#> Modifiers of hierarchical parameters:
#> NULL
#> Constants:
#>   Q   T   P   N   G   H   D 
#>   5   5   5 300  12   1   1
```

Response counts by item-year:

``` r
get_item_n(dgirt_in_liberalism, by = "year")
#>    year abortion affirmative_action stemcell_research
#> 1: 2006     5275               4750              2483
#> 2: 2007     1690               1557              1705
#> 3: 2008     4697               4704              4002
#> 4: 2009     2141               2147                 0
#> 5: 2010     9204               9241              9146
#>    gaymarriage_amendment partialbirth_abortion
#> 1:                  2642                  5064
#> 2:                  1163                  1684
#> 3:                  4265                     0
#> 4:                     0                     0
#> 5:                  9226                     0
```

### Fit a model with `dgirt`

`dgirt` and `dgmrp` fit estimation models to data from `shape`. `dgirt` can be used to estimate a latent variable based on responses to multiple survey questions (e.g., latent policy conservatism), while `dgmrp` can be used to estimate public opinion on an individual survey question using a dynamic multi-level regression and post-stratification (MRP) model.

Under the hood, these functions use RStan for MCMC sampling, and arguments can be passed to RStan's `stan` via the `...` argument of `dgirt` and `dgmrp`. This will almost always be desirable, at a minimum to specify the number of sampler iterations, chains, and cores.

``` r
dgirt_out_liberalism <- dgirt(dgirt_in_liberalism, iter = 3000, chains = 4,
  cores = 4, seed = 42)
```

The model results are held in a `dgirtfit` object. Methods from RStan like `extract` are available if needed because `dgirtfit` is a subclass of `stanfit`. But dgo provides its own methods for typical post-estimation tasks.

### Work with `dgirt` results

For a high-level summary of the result, use `summary`.

``` r
summary(dgirt_out_liberalism)
#> dgirt samples from 4 chains of 3000 iterations, 1500 warmup, thinned every 1 
#>   Drawn Mon May 22 11:08:37 2017 
#>   Package version 0.2.9 
#>   Model version 2017_01_04 
#>   137 parameters; 60 theta_bars (year state race3)
#>   5 periods 2006 to 2010 
#> 
#> n_eff
#>    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
#>   65.07  388.19  585.84 1083.11 1245.79 6000.00
#> 
#> Rhat
#>    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
#>  0.9997  1.0020  1.0059  1.0084  1.0119  1.0553
#> 
#> Elapsed time
#>    chain warmup sample  total
#> 1:     1 2M 14S  2M 2S 4M 16S
#> 2:     2  2M 3S 4M 11S 6M 14S
#> 3:     3 2M 29S  4M 7S 6M 36S
#> 4:     4 2M 27S 4M 15S 6M 42S
```

To summarize posterior samples, use `summarize`. The default output gives summary statistics for the `theta_bar` parameters, which represent the mean of the latent outcome for the groups defined by time, local geographic area, and the demographic characteristics specified in the earlier call to `shape`.

``` r
head(summarize(dgirt_out_liberalism))
#>        param state race3 year      mean         sd    median      q_025
#> 1: theta_bar    CA black 2006 0.5247367 0.06827003 0.5194593 0.40423566
#> 2: theta_bar    CA black 2007 0.5944296 0.08813494 0.5853362 0.44776239
#> 3: theta_bar    CA black 2008 0.5823158 0.10139495 0.5667726 0.43013247
#> 4: theta_bar    CA black 2009 0.5132426 0.08060875 0.5021405 0.38781331
#> 5: theta_bar    CA black 2010 0.4977903 0.05987256 0.4922105 0.39645056
#> 6: theta_bar    CA other 2006 0.1593904 0.05938098 0.1664194 0.02339847
#>        q_975
#> 1: 0.6754660
#> 2: 0.7932721
#> 3: 0.8190219
#> 4: 0.7062299
#> 5: 0.6332728
#> 6: 0.2583196
```

Alternatively, `summarize` can apply arbitrary functions to posterior samples for whatever parameter is given by its `pars` argument. Enclose function names with quotes. For convenience, `"q_025"` and `"q_975"` give the 2.5th and 97.5th posterior quantiles.

``` r
summarize(dgirt_out_liberalism, pars = "xi", funs = "var")
#>    param year         var
#> 1:    xi 2006 0.013076032
#> 2:    xi 2007 0.008053516
#> 3:    xi 2008 0.006789127
#> 4:    xi 2009 0.006144990
#> 5:    xi 2010 0.005945176
```

To access posterior samples in tabular form use `as.data.frame`. By default, this method returns post-warmup samples for the `theta_bar` parameters, but like other methods takes a `pars` argument.

``` r
head(as.data.frame(dgirt_out_liberalism))
#>        param state race3 year iteration     value
#> 1: theta_bar    CA black 2006         1 0.6107959
#> 2: theta_bar    CA black 2006         2 0.4745799
#> 3: theta_bar    CA black 2006         3 0.4980549
#> 4: theta_bar    CA black 2006         4 0.4898826
#> 5: theta_bar    CA black 2006         5 0.4939210
#> 6: theta_bar    CA black 2006         6 0.4746524
```

To poststratify the results use `poststratify`. The following example uses the group population proportions bundled as `annual_state_race_targets` to reweight and aggregate estimates to strata defined by state-years. Read `help("poststratify")` for more details.

``` r
poststratify(dgirt_out_liberalism, annual_state_race_targets, strata_names = c("state",
    "year"), aggregated_names = "race3")
#>     state year        value
#>  1:    CA 2006  0.143321712
#>  2:    CA 2007  0.188969603
#>  3:    CA 2008  0.112907172
#>  4:    CA 2009  0.058219329
#>  5:    CA 2010  0.092557709
#>  6:    GA 2006  0.103355439
#>  7:    GA 2007  0.084458691
#>  8:    GA 2008 -0.011351441
#>  9:    GA 2009 -0.015584764
#> 10:    GA 2010  0.010578655
#> 11:    LA 2006  0.021248643
#> 12:    LA 2007 -0.003170117
#> 13:    LA 2008 -0.095756506
#> 14:    LA 2009 -0.124279123
#> 15:    LA 2010 -0.088613763
#> 16:    MA 2006  0.147235550
#> 17:    MA 2007  0.269984992
#> 18:    MA 2008  0.159194876
#> 19:    MA 2009  0.082495757
#> 20:    MA 2010  0.122864118
```

To plot the results use `dgirt_plot`. This method plots summaries of posterior samples by time period. By default, it shows a 95% credible interval around posterior medians for the `theta_bar` parameters, for each local geographic area. For this (unconverged) toy example we omit the CIs.

``` r
dgirt_plot(dgirt_out_liberalism, y_min = NULL, y_max = NULL)
```

![](https://raw.githubusercontent.com/jamesdunham/dgo/master/README/dgirt_plot-1.png)

`dgirt_plot` can also plot the `data.frame` output from `poststratify`. This requires arguments that identify the relevant variables in the `data.frame`. Below, `poststratify` aggregates over the demographic grouping variable `race3`, resulting in a `data.frame` of estimates by state-year. So, in the subsequent call to `dgirt_plot`, we pass the names of the state and year variables. The `group_names` argument is `NULL` because there are no grouping variables left after aggregating over `race3`.

``` r
ps <- poststratify(dgirt_out_liberalism, annual_state_race_targets, strata_names = c("state",
    "year"), aggregated_names = "race3")
head(ps)
#>    state year      value
#> 1:    CA 2006 0.14332171
#> 2:    CA 2007 0.18896960
#> 3:    CA 2008 0.11290717
#> 4:    CA 2009 0.05821933
#> 5:    CA 2010 0.09255771
#> 6:    GA 2006 0.10335544
dgirt_plot(ps, group_names = NULL, time_name = "year", geo_name = "state")
```

![](https://raw.githubusercontent.com/jamesdunham/dgo/master/README/dgirt_plot_ps-1.png)

Troubleshooting
---------------

Please [report issues](https://github.com/jamesdunham/dgo/issues) that you encounter.

OS X only: RStan creates temporary files during estimation in a location given by `tempdir`, typically an arbitrary location in `/var/folders`. If a model runs for days, these files can be cleaned up while still needed, which induces an error. A good solution is to set a safer path for temporary files, using an environment variable checked at session startup. As described in `?tempdir`,

> The environment variables ‘TMPDIR’, ‘TMP’ and ‘TEMP’ are checked in turn and the first found which points to a writable directory is used: if none succeeds ‘/tmp’ is used. The path should not contain spaces.

For help setting environment variables, see the Stack Overflow question [here](https://stackoverflow.com/questions/17107206/change-temporary-directory). Confirm the new path before starting your model run by restarting R and checking the output from `tempdir()`.

``` r
# Problematic temporary directories on OS X look like this
tempdir()   
#> [1] "/var/folders/2p/_d3c95qd6ljg28j1f5l2jqxm0000gn/T//Rtmp38a10A"
```

Models fitted before October 2016 (specifically &lt; [\#8e6a2cf](https://github.com/jamesdunham/dgo/commit/8e6a2cfbe00b2cd4a908b3067241e06124d143cd)) using dgirtfit are not fully compatible with dgo. Their contents can be extracted without using dgo, however, with the `$` indexing operator. For example: `as.data.frame(dgirtfit_object$stan.cmb)`.

Contributing and citing
-----------------------

dgo is under development and we welcome [suggestions](https://github.com/jamesdunham/dgo/issues).

The package citation is

> Dunham, James, Devin Caughey, and Christopher Warshaw. 2017. dgo: Dynamic Estimation of Group-level Opinion. R package. <https://jamesdunham.github.io/dgo/>.
