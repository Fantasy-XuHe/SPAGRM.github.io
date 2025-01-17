---
layout: default
title: Linear mixed-effect models
nav_order: 1
description: "Just the Docs is a responsive Jekyll theme with built-in search that is easily customizable and hosted on GitHub Pages."
parent: Longitudinal mean
grand_parent: Step 1 Fit the null model
has_children: false
---

<head>
    <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
    <script type="text/x-mathjax-config">
        MathJax.Hub.Config({
            tex2jax: {
            skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
            inlineMath: [['$','$']]
            }
        });
    </script>
</head>

# **Linear mixed-effect models**

We first display how to use [lme4](https://cran.r-project.org/web/packages/lme4/index.html) in R to fit linear mixed model. We also display how to use [WiSER](https://github.com/OpenMendel/WiSER.jl), a Julia package to fit the null model for longitudinal traits.

## Quick start for lme4

Please run the following code in R

### Display the example data set

```
PhenoFile = system.file("extdata", "simuLongPHENO.txt", package = "GRAB")
print(PhenoFile)
# "../GRAB/extdata/simuLongPHENO.txt"

LongPheno = data.table::fread(PhenoFile)
print(LongPheno)
#           IID      AGE GENDER LongPheno
#     1: Subj-1 49.76668      0  21.18404
#     2: Subj-1 46.97617      0  22.63922
#     3: Subj-1 48.60388      0  22.99897
#     4: Subj-1 48.47036      0  22.88830
#     5: Subj-1 50.51208      0  23.17735
#    ---                                 
# 10511:   f9_9 50.34522      1  19.04696
# 10512:   f9_9 50.44379      1  24.20562
# 10513:   f9_9 51.52763      1  24.86237
# 10514:   f9_9 49.90129      1  23.18408
# 10515:   f9_9 50.21092      1  21.56067
```

### Fit the null model

```
library(dplyr)
library(tidyr)
library(lme4)

nullmodel = lmer(LongPheno ~ 1 + AGE + GENDER + (AGE|IID), data = LongPheno)

summary(nullmodel)$coefficients
#               Estimate Std. Error   t value
# (Intercept) 37.1712626 1.10910848  33.51454
# AGE         -0.3263161 0.02216986 -14.71891
# GENDER       0.7080723 0.06096869  11.61370
```

### Obtain model residuals

```
ResidMat = LongPheno %>% mutate(Resid = summary(nullmodel)$residuals) %>% group_by(IID) %>% summarize(Resid = sum(Resid))

names(ResidMat) = c("SubjID", "Resid") # rename the column names of ResidMat.

ResidMatFile = system.file("extdata", "ResidMat.txt", package = "GRAB")
data.table::fwrite(ResidMat, file = ResidMatFile, row.names = FALSE, col.names = TRUE)

summary(ResidMat$Resid)
#    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
#  -5.326  -1.073  -0.026   0.000   1.033   5.724 
```

## Quick start for TrajGWAS

Please run the following code in Julia. Julia package [TrajGWAS](https://github.com/OpenMendel/TrajGWAS.jl) is builded upon the [WiSER](https://github.com/OpenMendel/WiSER.jl) method and can also be used for model fitting. More details can be seen in [WiSER documentation](https://github.com/OpenMendel/WiSER.jl/blob/master/docs/src/model_fitting.md) or [TrajGWAS documentation](https://openmendel.github.io/TrajGWAS.jl/dev/). In this section, our main focus is to demonstrate how to obtain model residuals for testing $\beta$<sub>g</sub> = 0 (i.e., the mean level) from the fitted null model.

### Installation and library

```
using DataFrames, CSV, DelimitedFiles
using Ipopt, NLopt, KNITRO
using WiSER, TrajGWAS

# for TrajGWAS new version, solver settings:
solver = Ipopt.Optimizer(); solver_config = Dict("print_level"=>0, "mehrotra_algorithm"=>"yes", "warm_start_init_point"=>"yes", "max_iter"=>100)
# solver = Ipopt.Optimizer(); solver_config = Dict("print_level"=>0, "watchdog_shortened_iter_trigger"=>3, "max_iter"=>100)
# solver = KNITRO.Optimizer(); solver_config = Dict("outlev"=>3) # (Knitro is commercial software)
# solver = NLopt.Optimizer(); solver_config = Dict("algorithm"=>:LD_MMA, "maxeval"=>4000)
# solver = NLopt.Optimizer(); solver_config = Dict("algorithm"=>:LD_LBFGS, "maxeval"=>4000)
```

### Fit the null model

```
PhenoFile = "../GRAB/extdata/simuLongPHENO.txt" # please copy the filepath of simuLongPHENO.txt.
LongPheno = CSV.read(PhenoFile, DataFrame)

nullmodel = trajgwas(@formula(LongPheno ~ 1 + AGE + GENDER),
                    @formula(LongPheno ~ 1 + AGE),
                    @formula(LongPheno ~ 1 + AGE + GENDER),
                    :IID,
                    LongPheno,
                    nothing;
                    solver=solver,
                    solver_config = solver_config)
# ******************************************************************************
# This program contains Ipopt, a library for large-scale nonlinear optimization.
# Ipopt is released as open source code under the Eclipse Public License (EPL).
# For more information visit https://github.com/coin-or/Ipopt
# ******************************************************************************
# run = 1, ‖Δβ‖ = 0.379288, ‖Δτ‖ = 0.076489, ‖ΔL‖ = 0.653537, status = LOCALLY_SOLVED, time(s) = 0.800000
# run = 2, ‖Δβ‖ = 0.015746, ‖Δτ‖ = 0.028828, ‖ΔL‖ = 0.169548, status = LOCALLY_SOLVED, time(s) = 0.086000
# Within-subject variance estimation by robust regression (WiSER)
# Mean Formula:
# LongPheno ~ 1 + AGE + GENDER
# Random Effects Formula:
# LongPheno ~ 1 + AGE
# Within-Subject Variance Formula:
# LongPheno ~ 1 + AGE + GENDER
# Number of individuals/clusters: 1000
# Total observations: 10515
# Fixed-effects parameters:
# ─────────────────────────────────────────────────────────
#                    Estimate  Std. Error       Z  Pr(>|Z|)
# ─────────────────────────────────────────────────────────
# β1: (Intercept)  37.2366      1.13462     32.82    <1e-99
# β2: AGE          -0.327652    0.022705   -14.43    <1e-46
# β3: GENDER        0.712463    0.094477     7.54    <1e-13
# τ1: (Intercept)  -1.33006     2.07856     -0.64    0.5222
# τ2: AGE           0.0566825   0.042218     1.34    0.1794
# τ3: GENDER        0.132336    0.0642011    2.06    0.0393
# ─────────────────────────────────────────────────────────
# Random effects covariance matrix Σγ:
#  "γ1: (Intercept)"  67.3923   -1.32643
#  "γ2: AGE"          -1.32643   0.0266415
```

### Obtain model residuals of testing $\beta$<sub>g</sub> = 0 (i.e., the mean level)

```
rownames = nullmodel.ids

ResidMatFile_beta = split(PhenoFile,"simuLongPHENO.txt")[1] * "ResidMat.txt"

f1 = open(ResidMatFile_beta, "w")
writedlm(f1, ["SubjID" "Resid"])
for j in 1:length(nullmodel.data)
        Resid_beta = sum(nullmodel.data[j].Dinv_r - transpose(nullmodel.data[j].rt_UUt))
        writedlm(f1, [rownames[j] Resid_beta])
end
close(f1)

ResidMat_beta = CSV.read(ResidMatFile_beta, DataFrame)
# 1000×2 DataFrame
#   Row │ SubjID    Resid      
#       │ String15  Float64    
# ──────┼──────────────────────
#     1 │ Subj-1     0.806224
#     2 │ Subj-10    0.964086
#     3 │ Subj-100  -1.00028
#     4 │ Subj-101   0.156806
#     5 │ Subj-102  -0.62983
#     6 │ Subj-103   0.0177747
#     7 │ Subj-104  -0.0753876
#     8 │ Subj-105   0.769811
#   ⋮   │    ⋮          ⋮
#   994 │ f9_3      -0.132167
#   995 │ f9_4       0.142017
#   996 │ f9_5      -1.25016
#   997 │ f9_6       0.265729
#   998 │ f9_7      -1.84335
#   999 │ f9_8      -0.883836
#  1000 │ f9_9       0.627392
#              985 rows omitted
```

> **Note**  
> - The column names of <code style="color : fuchsia">ResidMatFile</code> must be exactly <code style="color : fuchsia">SubjID</code> in the first column and <code style="color : fuchsia">Resid</code> in the second column.
> - Each subject should match its corresponding residual.
> - We did not observe any difference of fitting linear mixed models using [lme4](https://cran.r-project.org/web/packages/lme4/index.html) and [WiSER](https://github.com/OpenMendel/WiSER.jl) on the final results when testing longitudinal mean.
