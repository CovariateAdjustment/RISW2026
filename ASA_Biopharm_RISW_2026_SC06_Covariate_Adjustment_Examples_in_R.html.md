---
title: "Improving Precision & Power in Randomized Trials Using Covariate Adjustment"

shorttitle: "RISW 2026 SC06: Improving Power and Precision in Randomized Trials Using Covariate Adjustment"

subtitle: "ASA Biopharm 2026: SC06"

date: "today"


author:   
  - name: Josh Betz^1^
    id: JFB
    orcid: 0000-0003-4488-9799
    email: jbetz@jhu.edu
    # affiliations:
    #   - name: Johns Hopkins Bloomberg School of Public Health
    #     city: Baltimore
    #     state: MD
    #     country: USA
    #     department: Department of Biostatistics
        
institute:
  - "^1^Department of Biostatistics,\n\nJohns Hopkins Bloomberg School of Public Health\n\nBaltimore, MD, USA"

format:
  html:
    citation: FALSE
    citation-location: document
    citations-hover: true
    code-fold: "show"
    crossrefs-hover: true
    embed-resources: true
    fig-align: center
    fig-dpi: 72
    fig-height: 8
    fig-width: 8
    grid:
      sidebar-width: 300px
      body-width: 1200px
      margin-width: 300px
    image-height: "8 in"
    image-width: "8 in"
    keep-md: true
    number-sections: true
    number-depth: 3
    page-layout: full
    toc: true
    toc-location: left

bibliography: bibliography.bib
# csl: american-statistical-association.csl
---


::: {.cell}

:::






::: {.cell}

:::



# Required Packages

::: {.panel-tabset}

## Install Packages


::: {.cell}

```{.r .cell-code}
install.packages(
  pkgs = c("dplyr", "drord", "cobalt", "kableExtra", "lmtest", "pak",
           "sandwich", "speff2trial", "table1")
)

# Installing packages from GitHub
pak::pak(
  pkg = 
    c("covariateadjustment/examplercts",
      "jbetz-jhu/drwls", 
      "nt-williams/simul",
      "nt-williams/adjrct")
)
```
:::


Occasionally you may get an error such as this:

```
**Error**:
! error in pak subprocess
**Caused by error** in `stop_task_install(state, worker)`:
! Failed to install binary package drwls.
Type .Last.error to see the more details.
``` 
If so, just try re-running the `pak()` command. This usually resolves itself. You can also try installing packages one-by-one.

## Load Packages


::: {.cell}

```{.r .cell-code}
# Continuous, Binary Analyses
library(dplyr) # Wrangling data
library(drwls) # g-Computation & DR-WLS
library(cobalt) # Standardized Differences
library(examplercts) # Example Datasets
library(kableExtra) # Printing Tables in HTML/TeX
library(lmtest) # Computing CIs with Robust SEs
library(sandwich) # Computing Robust SEs for GLMs
library(table1) # Tabulations

# Ordinal, Time-to-Event Analyses
library(adjrct) # Time-to-Event: Survival & RMST
library(drord) # Adjusted Ordinal Analyses
library(speff2trial) # Adjusted Marginal Hazard Ratio
library(survival) # Logrank tests; Cox PH Model
```
:::


:::




--------------------------------------------------------------------------------




# Continuous: Licorice Gargle


::: {.panel-tabset}


## Data Dictionary

We can use `?licorice_gargle` to open the help file about the data. The data come from a randomized trial assessing whether a licorice gargle provides better management of endotracheal extubation symptoms (couging, pain) at various points post-extubation (30 minutes, 90 minutes, 4 hours, next morning). The baseline covariates include age, sex, physical status, BMI, and Mallampati Score (predicted ease of endotracheal intubation).


::: {.cell}

```{.r .cell-code}
data.frame(
  column = names(licorice_gargle),
  name =
    sapply(X = licorice_gargle, function(x) attr(x = x, which = "label")),
  row.names = NULL
) %>%
  kableExtra::kable(
    x = .,
    caption = "Columns of the `licorice_gargle` dataset."
  )
```

::: {.cell-output-display}


Table: Columns of the `licorice_gargle` dataset.

|column                       |name                                |
|:----------------------------|:-----------------------------------|
|participant_id               |Participant ID                      |
|age_bl                       |Age at Baseline                     |
|gender                       |Gender                              |
|asa_physical_status_bl       |ASA Physical Status                 |
|bmi_bl                       |Body Mass Index (kg/m^2)            |
|mallampati_bl                |Mallampati Score (BL)               |
|smoking_bl                   |Smoking Status (BL)                 |
|pain_yn_bl                   |Preoperative Pain (BL)              |
|arm                          |Study Arm                           |
|tx                           |Treatment Indicator                 |
|surgery_size_post_rnd        |Surgery Size (Post-Randomization)   |
|extubation_cough             |Cough At Extubation                 |
|pacu_30_min_cough            |Cough at PACU: 30 Minutes           |
|pacu_30_min_throat_pain      |Throat Pain at PACU: 30 Minutes     |
|pacu_30_min_any_throat_pain  |Any Throat Pain at PACU: 30 Minutes |
|pacu_30_min_swallow_pain     |Swallow Pain at PACU: 30 Minutes    |
|pacu_90_min_cough            |Cough at PACU: 90 Minutes           |
|pacu_90_min_throat_pain      |Throat Pain at PACU: 90 Minutes     |
|pacu_90_min_any_throat_pain  |Any Throat Pain at PACU: 90 Minutes |
|postop_4h_cough              |Cough: 4H Post-Op                   |
|postop_4h_throat_pain        |Throat Pain: 4H Post-Op             |
|postop_4h_any_throat_pain    |Any Throat Pain: 4H Post-Op         |
|postop_1d_am_cough           |Cough: 1d Post-Op in AM             |
|postop_1d_am_throat_pain     |Throat Pain: 1d Post-Op in AM       |
|postop_1d_am_any_throat_pain |Any Throat Pain: 1d Post-Op in AM   |


:::
:::





## Tabulations: BL

The `table1` package can be used to provide tabulations of the baseline covariates. It is important to assess whether some covariates may have categories with low frequencies that may need to be pooled in accordance with the statistical analysis plan, or missing values that need to be addressed through imputation.


::: {.cell}

```{.r .cell-code}
table1::table1(
  x = ~ age_bl + gender + asa_physical_status_bl + bmi_bl + mallampati_bl |
    arm,
  data = licorice_gargle
)
```

::: {.cell-output-display}

```{=html}
<div class="Rtable1"><table class="Rtable1">
<thead>
<tr>
<th class='rowlabel firstrow lastrow'></th>
<th class='firstrow lastrow'><span class='stratlabel'>0. 5g Sugar<br/><span class='stratn'>(N=117)</span></span></th>
<th class='firstrow lastrow'><span class='stratlabel'>1. 0.5g Licorice<br/><span class='stratn'>(N=118)</span></span></th>
<th class='firstrow lastrow'><span class='stratlabel'>Overall<br/><span class='stratn'>(N=235)</span></span></th>
</tr>
</thead>
<tbody>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Age at Baseline</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>Mean (SD)</td>
<td>58.0 (16.1)</td>
<td>56.7 (14.9)</td>
<td>57.4 (15.5)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>Median [Min, Max]</td>
<td class='lastrow'>63.0 [18.0, 86.0]</td>
<td class='lastrow'>60.5 [19.0, 83.0]</td>
<td class='lastrow'>62.0 [18.0, 86.0]</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Gender</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>0. Female</td>
<td>44 (37.6%)</td>
<td>49 (41.5%)</td>
<td>93 (39.6%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>1. Male</td>
<td class='lastrow'>73 (62.4%)</td>
<td class='lastrow'>69 (58.5%)</td>
<td class='lastrow'>142 (60.4%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>ASA Physical Status</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>1. Healthy</td>
<td>19 (16.2%)</td>
<td>22 (18.6%)</td>
<td>41 (17.4%)</td>
</tr>
<tr>
<td class='rowlabel'>2. Mild Systemic Disease</td>
<td>67 (57.3%)</td>
<td>67 (56.8%)</td>
<td>134 (57.0%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>3. Severe Systemic Disease</td>
<td class='lastrow'>31 (26.5%)</td>
<td class='lastrow'>29 (24.6%)</td>
<td class='lastrow'>60 (25.5%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Body Mass Index (kg/m^2)</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>Mean (SD)</td>
<td>25.6 (4.25)</td>
<td>25.6 (4.32)</td>
<td>25.6 (4.27)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>Median [Min, Max]</td>
<td class='lastrow'>26.1 [15.6, 34.1]</td>
<td class='lastrow'>25.7 [16.4, 36.3]</td>
<td class='lastrow'>25.9 [15.6, 36.3]</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Mallampati Score (BL)</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>1. 1</td>
<td>31 (26.5%)</td>
<td>39 (33.1%)</td>
<td>70 (29.8%)</td>
</tr>
<tr>
<td class='rowlabel'>2. 2</td>
<td>69 (59.0%)</td>
<td>66 (55.9%)</td>
<td>135 (57.4%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>3. 3+</td>
<td class='lastrow'>17 (14.5%)</td>
<td class='lastrow'>13 (11.0%)</td>
<td class='lastrow'>30 (12.8%)</td>
</tr>
</tbody>
</table>
</div>
```

:::
:::





## Imbalance

The imbalance in covariates between arms can be assessed using their standardized differences. Covariate adjustment should be done based on *a priori* knowledge of the domain area, not based on the observed imbalance.



::: {.cell}

```{.r .cell-code}
cobalt::bal.tab(
  x = 
    # Only tabulate baseline variables
    licorice_gargle %>% 
    dplyr::select(
      dplyr::all_of(
        x = c("age_bl", "gender", "asa_physical_status_bl", "bmi_bl",
              "mallampati_bl")
      )
    ),
  treat = licorice_gargle$arm,
  # Compute standardized differences for both binary and continuous variables
  binary = "std",
  continuous = "std",
  s.d.denom = "pooled"
)
```

::: {.cell-output .cell-output-stdout}

```
Balance Measures
                                                     Type Diff.Un
age_bl                                            Contin. -0.0854
gender_1. Male                                     Binary -0.0802
asa_physical_status_bl_1. Healthy                  Binary  0.0634
asa_physical_status_bl_2. Mild Systemic Disease    Binary -0.0098
asa_physical_status_bl_3. Severe Systemic Disease  Binary -0.0440
bmi_bl                                            Contin. -0.0122
mallampati_bl_1. 1                                 Binary  0.1437
mallampati_bl_2. 2                                 Binary -0.0616
mallampati_bl_3. 3+                                Binary -0.1054

Sample sizes
    0. 5g Sugar 1. 0.5g Licorice
All         117              118
```


:::
:::


The results can be saved and reformatted into a table for presentation purposes.




## Tabulations: Outcomes

Outcomes should also be checked for missingness and low frequencies, which
should be addressed according to the statistical analysis plan.


::: {.cell}

```{.r .cell-code}
table1::table1(
  x = ~ pacu_30_min_throat_pain + pacu_30_min_cough +
    pacu_90_min_throat_pain + pacu_90_min_cough +
    postop_4h_throat_pain + postop_4h_cough +
    postop_1d_am_throat_pain + postop_1d_am_cough |
    arm,
  data = licorice_gargle
)
```

::: {.cell-output-display}

```{=html}
<div class="Rtable1"><table class="Rtable1">
<thead>
<tr>
<th class='rowlabel firstrow lastrow'></th>
<th class='firstrow lastrow'><span class='stratlabel'>0. 5g Sugar<br/><span class='stratn'>(N=117)</span></span></th>
<th class='firstrow lastrow'><span class='stratlabel'>1. 0.5g Licorice<br/><span class='stratn'>(N=118)</span></span></th>
<th class='firstrow lastrow'><span class='stratlabel'>Overall<br/><span class='stratn'>(N=235)</span></span></th>
</tr>
</thead>
<tbody>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Throat Pain at PACU: 30 Minutes</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>Mean (SD)</td>
<td>1.03 (1.55)</td>
<td>0.274 (0.678)</td>
<td>0.648 (1.25)</td>
</tr>
<tr>
<td class='rowlabel'>Median [Min, Max]</td>
<td>0 [0, 6.00]</td>
<td>0 [0, 4.00]</td>
<td>0 [0, 6.00]</td>
</tr>
<tr>
<td class='rowlabel lastrow'>Missing</td>
<td class='lastrow'>1 (0.9%)</td>
<td class='lastrow'>1 (0.8%)</td>
<td class='lastrow'>2 (0.9%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Cough at PACU: 30 Minutes</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>0. No Cough</td>
<td>88 (75.2%)</td>
<td>99 (83.9%)</td>
<td>187 (79.6%)</td>
</tr>
<tr>
<td class='rowlabel'>1. Mild Cough</td>
<td>24 (20.5%)</td>
<td>18 (15.3%)</td>
<td>42 (17.9%)</td>
</tr>
<tr>
<td class='rowlabel'>2. Moderate Cough</td>
<td>4 (3.4%)</td>
<td>0 (0%)</td>
<td>4 (1.7%)</td>
</tr>
<tr>
<td class='rowlabel'>3. Severe Cough</td>
<td>0 (0%)</td>
<td>0 (0%)</td>
<td>0 (0%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>Missing</td>
<td class='lastrow'>1 (0.9%)</td>
<td class='lastrow'>1 (0.8%)</td>
<td class='lastrow'>2 (0.9%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Throat Pain at PACU: 90 Minutes</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>Mean (SD)</td>
<td>0.819 (1.32)</td>
<td>0.137 (0.453)</td>
<td>0.476 (1.04)</td>
</tr>
<tr>
<td class='rowlabel'>Median [Min, Max]</td>
<td>0 [0, 6.00]</td>
<td>0 [0, 3.00]</td>
<td>0 [0, 6.00]</td>
</tr>
<tr>
<td class='rowlabel lastrow'>Missing</td>
<td class='lastrow'>1 (0.9%)</td>
<td class='lastrow'>1 (0.8%)</td>
<td class='lastrow'>2 (0.9%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Cough at PACU: 90 Minutes</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>0. No Cough</td>
<td>91 (77.8%)</td>
<td>101 (85.6%)</td>
<td>192 (81.7%)</td>
</tr>
<tr>
<td class='rowlabel'>1. Mild Cough</td>
<td>23 (19.7%)</td>
<td>16 (13.6%)</td>
<td>39 (16.6%)</td>
</tr>
<tr>
<td class='rowlabel'>2. Moderate Cough</td>
<td>2 (1.7%)</td>
<td>0 (0%)</td>
<td>2 (0.9%)</td>
</tr>
<tr>
<td class='rowlabel'>3. Severe Cough</td>
<td>0 (0%)</td>
<td>0 (0%)</td>
<td>0 (0%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>Missing</td>
<td class='lastrow'>1 (0.9%)</td>
<td class='lastrow'>1 (0.8%)</td>
<td class='lastrow'>2 (0.9%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Throat Pain: 4H Post-Op</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>Mean (SD)</td>
<td>0.914 (1.38)</td>
<td>0.350 (0.758)</td>
<td>0.631 (1.15)</td>
</tr>
<tr>
<td class='rowlabel'>Median [Min, Max]</td>
<td>0 [0, 7.00]</td>
<td>0 [0, 3.00]</td>
<td>0 [0, 7.00]</td>
</tr>
<tr>
<td class='rowlabel lastrow'>Missing</td>
<td class='lastrow'>1 (0.9%)</td>
<td class='lastrow'>1 (0.8%)</td>
<td class='lastrow'>2 (0.9%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Cough: 4H Post-Op</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>0. No Cough</td>
<td>77 (65.8%)</td>
<td>89 (75.4%)</td>
<td>166 (70.6%)</td>
</tr>
<tr>
<td class='rowlabel'>1. Mild Cough</td>
<td>35 (29.9%)</td>
<td>26 (22.0%)</td>
<td>61 (26.0%)</td>
</tr>
<tr>
<td class='rowlabel'>2. Moderate Cough</td>
<td>4 (3.4%)</td>
<td>2 (1.7%)</td>
<td>6 (2.6%)</td>
</tr>
<tr>
<td class='rowlabel'>3. Severe Cough</td>
<td>0 (0%)</td>
<td>0 (0%)</td>
<td>0 (0%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>Missing</td>
<td class='lastrow'>1 (0.9%)</td>
<td class='lastrow'>1 (0.8%)</td>
<td class='lastrow'>2 (0.9%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Throat Pain: 1d Post-Op in AM</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>Mean (SD)</td>
<td>0.647 (0.998)</td>
<td>0.316 (0.703)</td>
<td>0.481 (0.876)</td>
</tr>
<tr>
<td class='rowlabel'>Median [Min, Max]</td>
<td>0 [0, 6.00]</td>
<td>0 [0, 3.00]</td>
<td>0 [0, 6.00]</td>
</tr>
<tr>
<td class='rowlabel lastrow'>Missing</td>
<td class='lastrow'>1 (0.9%)</td>
<td class='lastrow'>1 (0.8%)</td>
<td class='lastrow'>2 (0.9%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Cough: 1d Post-Op in AM</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>0. No Cough</td>
<td>68 (58.1%)</td>
<td>86 (72.9%)</td>
<td>154 (65.5%)</td>
</tr>
<tr>
<td class='rowlabel'>1. Mild Cough</td>
<td>42 (35.9%)</td>
<td>27 (22.9%)</td>
<td>69 (29.4%)</td>
</tr>
<tr>
<td class='rowlabel'>2. Moderate Cough</td>
<td>5 (4.3%)</td>
<td>3 (2.5%)</td>
<td>8 (3.4%)</td>
</tr>
<tr>
<td class='rowlabel'>3. Severe Cough</td>
<td>1 (0.9%)</td>
<td>1 (0.8%)</td>
<td>2 (0.9%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>Missing</td>
<td class='lastrow'>1 (0.9%)</td>
<td class='lastrow'>1 (0.8%)</td>
<td class='lastrow'>2 (0.9%)</td>
</tr>
</tbody>
</table>
</div>
```

:::
:::





## Two Sample t-Test


::: {.cell}

```{.r .cell-code}
licorice_30_min_pain_t_test <-
  t.test(
    pacu_30_min_throat_pain ~ arm,
    data = licorice_gargle
  )

licorice_30_min_pain_t_test
```

::: {.cell-output .cell-output-stdout}

```

	Welch Two Sample t-test

data:  pacu_30_min_throat_pain by arm
t = 4.8035, df = 157.3, p-value = 3.608e-06
alternative hypothesis: true difference in means between group 0. 5g Sugar and group 1. 0.5g Licorice is not equal to 0
95 percent confidence interval:
 0.4429931 1.0617225
sample estimates:
     mean in group 0. 5g Sugar mean in group 1. 0.5g Licorice 
                     1.0258621                      0.2735043 
```


:::

```{.r .cell-code}
# Note: t-test is presenting [Control] (Reference) - [Treatment]
# This is simple to correct to give [Treatment] - [Control]
diff(licorice_30_min_pain_t_test$estimate)
```

::: {.cell-output .cell-output-stdout}

```
mean in group 1. 0.5g Licorice 
                    -0.7523578 
```


:::

```{.r .cell-code}
-rev(licorice_30_min_pain_t_test$conf.int)
```

::: {.cell-output .cell-output-stdout}

```
[1] -1.0617225 -0.4429931
```


:::

```{.r .cell-code}
# Unadjusted estimate Standard Error
licorice_30_min_pain_t_test$stderr
```

::: {.cell-output .cell-output-stdout}

```
[1] 0.1566277
```


:::
:::





## ANCOVA

The ANCOVA is a model-based approach to covariate adjustment, but since it uses an identity link, the marginal effect and conditional effect coincide. Only the marginal effect measure is robust to model misspecification, which requires that robust standard errors are used for testing and confidence intervals [@Tsiatis2008Covariate].


::: {.cell}

```{.r .cell-code}
licorice_30_min_pain_glm_adjusted <-
  glm(
    pacu_30_min_throat_pain ~ arm + 
      age_bl + gender + asa_physical_status_bl + bmi_bl + mallampati_bl,
    data = licorice_gargle,
    family = gaussian # Default for GLM
  )

# Model-Based Inference: Assumes correctly-specified model
summary(licorice_30_min_pain_glm_adjusted)
```

::: {.cell-output .cell-output-stdout}

```

Call:
glm(formula = pacu_30_min_throat_pain ~ arm + age_bl + gender + 
    asa_physical_status_bl + bmi_bl + mallampati_bl, family = gaussian, 
    data = licorice_gargle)

Coefficients:
                                                  Estimate Std. Error t value
(Intercept)                                       1.355758   0.509446   2.661
arm1. 0.5g Licorice                              -0.721939   0.156519  -4.612
age_bl                                           -0.004450   0.005869  -0.758
gender1. Male                                     0.288939   0.163041   1.772
asa_physical_status_bl2. Mild Systemic Disease    0.242869   0.247538   0.981
asa_physical_status_bl3. Severe Systemic Disease  0.333920   0.272183   1.227
bmi_bl                                           -0.026472   0.020458  -1.294
mallampati_bl2. 2                                 0.288904   0.187045   1.545
mallampati_bl3. 3+                                0.182806   0.276737   0.661
                                                 Pr(>|t|)    
(Intercept)                                       0.00835 ** 
arm1. 0.5g Licorice                              6.69e-06 ***
age_bl                                            0.44916    
gender1. Male                                     0.07772 .  
asa_physical_status_bl2. Mild Systemic Disease    0.32758    
asa_physical_status_bl3. Severe Systemic Disease  0.22118    
bmi_bl                                            0.19701    
mallampati_bl2. 2                                 0.12386    
mallampati_bl3. 3+                                0.50956    
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Dispersion parameter for gaussian family taken to be 1.411958)

    Null deviance: 361.14  on 232  degrees of freedom
Residual deviance: 316.28  on 224  degrees of freedom
  (2 observations deleted due to missingness)
AIC: 752.43

Number of Fisher Scoring iterations: 2
```


:::

```{.r .cell-code}
confint(licorice_30_min_pain_glm_adjusted)["arm1. 0.5g Licorice",]
```

::: {.cell-output .cell-output-stderr}

```
Waiting for profiling to be done...
```


:::

::: {.cell-output .cell-output-stdout}

```
     2.5 %     97.5 % 
-1.0287106 -0.4151676 
```


:::

```{.r .cell-code}
licorice_30_min_pain_ancova_robust_vcov <-
  sandwich::vcovHC(licorice_30_min_pain_glm_adjusted, type = "HC0")

# Tests based on Robust SEs:
lmtest::coeftest(
  x = licorice_30_min_pain_glm_adjusted,
  vcov. = licorice_30_min_pain_ancova_robust_vcov
)
```

::: {.cell-output .cell-output-stdout}

```

z test of coefficients:

                                                   Estimate Std. Error z value
(Intercept)                                       1.3557577  0.5555588  2.4403
arm1. 0.5g Licorice                              -0.7219391  0.1510823 -4.7784
age_bl                                           -0.0044495  0.0062754 -0.7090
gender1. Male                                     0.2889391  0.1538593  1.8779
asa_physical_status_bl2. Mild Systemic Disease    0.2428688  0.2128941  1.1408
asa_physical_status_bl3. Severe Systemic Disease  0.3339201  0.2635809  1.2669
bmi_bl                                           -0.0264722  0.0192630 -1.3743
mallampati_bl2. 2                                 0.2889037  0.1733967  1.6661
mallampati_bl3. 3+                                0.1828057  0.2619940  0.6977
                                                  Pr(>|z|)    
(Intercept)                                        0.01467 *  
arm1. 0.5g Licorice                              1.767e-06 ***
age_bl                                             0.47830    
gender1. Male                                      0.06039 .  
asa_physical_status_bl2. Mild Systemic Disease     0.25395    
asa_physical_status_bl3. Severe Systemic Disease   0.20521    
bmi_bl                                             0.16936    
mallampati_bl2. 2                                  0.09568 .  
mallampati_bl3. 3+                                 0.48534    
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
```


:::

```{.r .cell-code}
# CIs based on Robust SEs
licorice_30_min_pain_ancova_robust_ci <-
  lmtest::coefci(
    x = licorice_30_min_pain_glm_adjusted,
    vcov. = licorice_30_min_pain_ancova_robust_vcov
  )
licorice_30_min_pain_ancova_robust_ci
```

::: {.cell-output .cell-output-stdout}

```
                                                       2.5 %       97.5 %
(Intercept)                                       0.26688241  2.444633053
arm1. 0.5g Licorice                              -1.01805508 -0.425823178
age_bl                                           -0.01674907  0.007850052
gender1. Male                                    -0.01261948  0.590497703
asa_physical_status_bl2. Mild Systemic Disease   -0.17439596  0.660133622
asa_physical_status_bl3. Severe Systemic Disease -0.18268894  0.850529212
bmi_bl                                           -0.06422693  0.011282550
mallampati_bl2. 2                                -0.05094757  0.628754917
mallampati_bl3. 3+                               -0.33069311  0.696304498
```


:::
:::





## G-Computation: Manual

G-computation, also known as marginal standardization, is a semiparametric approach that uses a fitted model for predicting the expected outcome for each individual under each treatment assignment. These predictions are then averaged, and the averages are contrasted to give a difference or ratio measure (difference in means, risk difference, risk ratio, or odds ratio) [@Colantuoni2015Leveraging].


::: {.cell}

```{.r .cell-code}
# Step 2: Use working model to predict outcomes under each treatment
pred_30_min_pain_licorice <-
  predict(
    object = licorice_30_min_pain_glm_adjusted,
    newdata =
      # Change all treatment assignment to Licorice Gargle arm
      within(
        data = licorice_gargle,
        expr = {arm = "1. 0.5g Licorice"}
      ),
    type = "response"
  )

pred_30_min_pain_sucrose <-
  predict(
    object = licorice_30_min_pain_glm_adjusted,
    newdata =
      # Change all treatment assignment to Sucrose Gargle arm
      within(
        data = licorice_gargle,
        expr = {arm = "0. 5g Sugar"}
      ),
    type = "response"
  )

# Step 3: Average predictions
mean_pred_30_min_pain_licorice <- mean(pred_30_min_pain_licorice)
mean_pred_30_min_pain_sucrose <- mean(pred_30_min_pain_sucrose)

# Step 4: Contrast Predictions
difference_licorice_sucrose <- 
  mean_pred_30_min_pain_licorice - mean_pred_30_min_pain_sucrose
difference_licorice_sucrose
```

::: {.cell-output .cell-output-stdout}

```
[1] -0.7219391
```


:::

```{.r .cell-code}
ratio_licorice_sucrose <- 
  mean_pred_30_min_pain_licorice/mean_pred_30_min_pain_sucrose
ratio_licorice_sucrose
```

::: {.cell-output .cell-output-stdout}

```
[1] 0.2856797
```


:::

```{.r .cell-code}
# ANCOVA coincides with G-Computation
difference_licorice_sucrose
```

::: {.cell-output .cell-output-stdout}

```
[1] -0.7219391
```


:::

```{.r .cell-code}
coef(licorice_30_min_pain_glm_adjusted)["arm1. 0.5g Licorice"]
```

::: {.cell-output .cell-output-stdout}

```
arm1. 0.5g Licorice 
         -0.7219391 
```


:::
:::





## G-Computation: `drwls::standardization`

The `drwls::standardization` performs G-computation (marginal standardization). It imputes missing data using mean/mode imputation, and obtains standard errors either using the nonparametric bootstrap or the sandwich estimator.


::: {.cell}

```{.r .cell-code}
licorice_30_min_pain_g_computation <-
  drwls::standardization(
    outcome_formula = 
      pacu_30_min_throat_pain ~ tx +
      age_bl + gender + asa_physical_status_bl + bmi_bl + mallampati_bl,
    outcome_family = gaussian,
    treatment_column = "tx",
    estimand = "difference", # Compute Difference in Means
    data = licorice_gargle,
    se_method = "asymptotic"
  )

licorice_30_min_pain_g_computation
```

::: {.cell-output .cell-output-stdout}

```

Estimand: Difference in Means 
Method: Marginal Standardization (G-Computation)
Outcome Models: a gaussian GLM with an identity link
 

             Estimand   Estimate        SE       LCL        UCL         Z
1 E[Y|A=1] - E[Y|A=0] -0.7219391 0.1510823 -1.018055 -0.4258232 -4.778448
2            E[Y|A=1]  0.2887267        NA        NA         NA        NA
3            E[Y|A=0]  1.0106658        NA        NA         NA        NA
       p-value
1 1.766534e-06
2           NA
3           NA

Variance Estimation: Sandwich Estimator 
CI Method: Wald 95% CI 
Details:
  Outcomes: (N = 233/235)
  Outcome Model: pacu_30_min_throat_pain ~ tx + age_bl + gender + asa_physical_status_bl +      bmi_bl + mallampati_bl
```


:::
:::





## Precision Gain

An estimate of the precision gain can be obtained by estimating the relative
efficiency of the adjusted and unadjusted estimates.


::: {.cell}

```{.r .cell-code}
# Unadjusted SE
licorice_30_min_pain_t_test$stderr
```

::: {.cell-output .cell-output-stdout}

```
[1] 0.1566277
```


:::

```{.r .cell-code}
# ANOVA: Robust SE
ate_i <- 
  which(colnames(licorice_30_min_pain_ancova_robust_vcov) == "arm1. 0.5g Licorice")
sqrt(licorice_30_min_pain_ancova_robust_vcov[ate_i, ate_i])
```

::: {.cell-output .cell-output-stdout}

```
[1] 0.1510823
```


:::

```{.r .cell-code}
# G-Computation: Robust SE
licorice_30_min_pain_g_computation$result$se[1]
```

::: {.cell-output .cell-output-stdout}

```
[1] 0.1510823
```


:::

```{.r .cell-code}
# Improvement in Efficiency
1 - (licorice_30_min_pain_g_computation$result$se[1]/licorice_30_min_pain_t_test$stderr)^2
```

::: {.cell-output .cell-output-stdout}

```
[1] 0.06955598
```


:::
:::





## Small Sample Corrections

The small sample correction provided in [@Tsiatis2008Covariate] can be implemented using the `variance_adjustment` argument:


::: {.cell}

```{.r .cell-code}
licorice_30_min_pain_g_computation_tsiatis <-
  drwls::standardization(
    outcome_formula = 
      pacu_30_min_throat_pain ~ tx +
      age_bl + gender + asa_physical_status_bl + bmi_bl + mallampati_bl,
    outcome_family = gaussian,
    treatment_column = "tx",
    estimand = "difference", # Compute Difference in Means
    data = licorice_gargle,
    se_method = "asymptotic",
    variance_adjustment = variance_adjustment_tsiatis
  )

licorice_30_min_pain_g_computation
```

::: {.cell-output .cell-output-stdout}

```

Estimand: Difference in Means 
Method: Marginal Standardization (G-Computation)
Outcome Models: a gaussian GLM with an identity link
 

             Estimand   Estimate        SE       LCL        UCL         Z
1 E[Y|A=1] - E[Y|A=0] -0.7219391 0.1510823 -1.018055 -0.4258232 -4.778448
2            E[Y|A=1]  0.2887267        NA        NA         NA        NA
3            E[Y|A=0]  1.0106658        NA        NA         NA        NA
       p-value
1 1.766534e-06
2           NA
3           NA

Variance Estimation: Sandwich Estimator 
CI Method: Wald 95% CI 
Details:
  Outcomes: (N = 233/235)
  Outcome Model: pacu_30_min_throat_pain ~ tx + age_bl + gender + asa_physical_status_bl +      bmi_bl + mallampati_bl
```


:::
:::





## Doubly Robust: `drwls:drwls`

To address missingness, the `drwls()` function weights the complete cases by the inverse probability of being observed at follow-up based on a missingness model [@Robins2007Performance]:


::: {.cell}

```{.r .cell-code}
licorice_30_min_pain_drwls <-
  drwls::drwls(
    outcome_formula = 
      pacu_30_min_throat_pain ~ tx +
      age_bl + gender + asa_physical_status_bl + bmi_bl + mallampati_bl,
    outcome_family = gaussian,
    treatment_column = "tx",
    estimand = "difference", # Compute Difference in Means
    missing_formula = ~ tx + age_bl + bmi_bl,
    missing_family = binomial,
    data = licorice_gargle,
    se_method = "asymptotic"
  )

licorice_30_min_pain_drwls
```

::: {.cell-output .cell-output-stdout}

```

Estimand: Difference in Means 
Method: Doubly-Robust Weighted Least Squares
Outcome Models: a gaussian GLM with an identity link
Censoring Model: a binomial GLM using a logit link 

             Estimand   Estimate        SE       LCL        UCL         Z
1 E[Y|A=1] - E[Y|A=0] -0.7207865 0.1508435 -1.016434 -0.4251386 -4.778373
2            E[Y|A=1]  0.2894750        NA        NA         NA        NA
3            E[Y|A=0]  1.0102615        NA        NA         NA        NA
       p-value
1 1.767198e-06
2           NA
3           NA

Variance Estimation: Sandwich Estimator 
CI Method: Wald 95% CI 
Details:
  Outcomes: (N = 233/235)
  Outcome Model: pacu_30_min_throat_pain ~ tx + age_bl + gender + asa_physical_status_bl +      bmi_bl + mallampati_bl
  Missing Formula: !is.na(pacu_30_min_throat_pain) ~ tx + age_bl + bmi_bl
```


:::
:::


:::




--------------------------------------------------------------------------------




# Binary: Licorice Gargle


::: {.panel-tabset}


## Tabulations: Outcomes


::: {.cell}

```{.r .cell-code}
table1::table1(
  x = ~ 
    pacu_30_min_any_throat_pain + 
    pacu_90_min_any_throat_pain +
    postop_4h_any_throat_pain + 
    postop_1d_am_any_throat_pain |
    arm,
  data = licorice_gargle
)
```

::: {.cell-output-display}

```{=html}
<div class="Rtable1"><table class="Rtable1">
<thead>
<tr>
<th class='rowlabel firstrow lastrow'></th>
<th class='firstrow lastrow'><span class='stratlabel'>0. 5g Sugar<br/><span class='stratn'>(N=117)</span></span></th>
<th class='firstrow lastrow'><span class='stratlabel'>1. 0.5g Licorice<br/><span class='stratn'>(N=118)</span></span></th>
<th class='firstrow lastrow'><span class='stratlabel'>Overall<br/><span class='stratn'>(N=235)</span></span></th>
</tr>
</thead>
<tbody>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Any Throat Pain at PACU: 30 Minutes</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>Mean (SD)</td>
<td>0.362 (0.483)</td>
<td>0.188 (0.392)</td>
<td>0.275 (0.447)</td>
</tr>
<tr>
<td class='rowlabel'>Median [Min, Max]</td>
<td>0 [0, 1.00]</td>
<td>0 [0, 1.00]</td>
<td>0 [0, 1.00]</td>
</tr>
<tr>
<td class='rowlabel lastrow'>Missing</td>
<td class='lastrow'>1 (0.9%)</td>
<td class='lastrow'>1 (0.8%)</td>
<td class='lastrow'>2 (0.9%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Any Throat Pain at PACU: 90 Minutes</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>Mean (SD)</td>
<td>0.353 (0.480)</td>
<td>0.103 (0.305)</td>
<td>0.227 (0.420)</td>
</tr>
<tr>
<td class='rowlabel'>Median [Min, Max]</td>
<td>0 [0, 1.00]</td>
<td>0 [0, 1.00]</td>
<td>0 [0, 1.00]</td>
</tr>
<tr>
<td class='rowlabel lastrow'>Missing</td>
<td class='lastrow'>1 (0.9%)</td>
<td class='lastrow'>1 (0.8%)</td>
<td class='lastrow'>2 (0.9%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Any Throat Pain: 4H Post-Op</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>Mean (SD)</td>
<td>0.448 (0.499)</td>
<td>0.205 (0.406)</td>
<td>0.326 (0.470)</td>
</tr>
<tr>
<td class='rowlabel'>Median [Min, Max]</td>
<td>0 [0, 1.00]</td>
<td>0 [0, 1.00]</td>
<td>0 [0, 1.00]</td>
</tr>
<tr>
<td class='rowlabel lastrow'>Missing</td>
<td class='lastrow'>1 (0.9%)</td>
<td class='lastrow'>1 (0.8%)</td>
<td class='lastrow'>2 (0.9%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Any Throat Pain: 1d Post-Op in AM</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>Mean (SD)</td>
<td>0.397 (0.491)</td>
<td>0.205 (0.406)</td>
<td>0.300 (0.459)</td>
</tr>
<tr>
<td class='rowlabel'>Median [Min, Max]</td>
<td>0 [0, 1.00]</td>
<td>0 [0, 1.00]</td>
<td>0 [0, 1.00]</td>
</tr>
<tr>
<td class='rowlabel lastrow'>Missing</td>
<td class='lastrow'>1 (0.9%)</td>
<td class='lastrow'>1 (0.8%)</td>
<td class='lastrow'>2 (0.9%)</td>
</tr>
</tbody>
</table>
</div>
```

:::
:::





## Logistic Regression

A logistic regression model can be used for model-based inference about the *conditional* odds ratio, assuming that the model is correctly specified:


::: {.cell}

```{.r .cell-code}
licorice_30_min_any_pain_glm_adjusted <-
  glm(
    pacu_30_min_any_throat_pain ~ arm + 
      age_bl + gender + asa_physical_status_bl + bmi_bl + mallampati_bl,
    data = licorice_gargle,
    family = binomial
  )

# Summarize logistic regression coefficients
summary(licorice_30_min_any_pain_glm_adjusted)
```

::: {.cell-output .cell-output-stdout}

```

Call:
glm(formula = pacu_30_min_any_throat_pain ~ arm + age_bl + gender + 
    asa_physical_status_bl + bmi_bl + mallampati_bl, family = binomial, 
    data = licorice_gargle)

Coefficients:
                                                  Estimate Std. Error z value
(Intercept)                                      -0.683311   1.034766  -0.660
arm1. 0.5g Licorice                              -0.886851   0.317326  -2.795
age_bl                                            0.004074   0.011356   0.359
gender1. Male                                     1.189266   0.355833   3.342
asa_physical_status_bl2. Mild Systemic Disease    0.747107   0.545077   1.371
asa_physical_status_bl3. Severe Systemic Disease  0.775223   0.582402   1.331
bmi_bl                                           -0.070571   0.042163  -1.674
mallampati_bl2. 2                                 0.376501   0.383713   0.981
mallampati_bl3. 3+                                0.199758   0.558798   0.357
                                                 Pr(>|z|)    
(Intercept)                                      0.509027    
arm1. 0.5g Licorice                              0.005194 ** 
age_bl                                           0.719800    
gender1. Male                                    0.000831 ***
asa_physical_status_bl2. Mild Systemic Disease   0.170486    
asa_physical_status_bl3. Severe Systemic Disease 0.183163    
bmi_bl                                           0.094178 .  
mallampati_bl2. 2                                0.326492    
mallampati_bl3. 3+                               0.720734    
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Dispersion parameter for binomial family taken to be 1)

    Null deviance: 273.94  on 232  degrees of freedom
Residual deviance: 248.37  on 224  degrees of freedom
  (2 observations deleted due to missingness)
AIC: 266.37

Number of Fisher Scoring iterations: 4
```


:::

```{.r .cell-code}
# Obtain conditional odds ratios
exp(coef(licorice_30_min_any_pain_glm_adjusted))
```

::: {.cell-output .cell-output-stdout}

```
                                     (Intercept) 
                                       0.5049426 
                             arm1. 0.5g Licorice 
                                       0.4119509 
                                          age_bl 
                                       1.0040819 
                                   gender1. Male 
                                       3.2846686 
  asa_physical_status_bl2. Mild Systemic Disease 
                                       2.1108853 
asa_physical_status_bl3. Severe Systemic Disease 
                                       2.1710763 
                                          bmi_bl 
                                       0.9318618 
                               mallampati_bl2. 2 
                                       1.4571767 
                              mallampati_bl3. 3+ 
                                       1.2211070 
```


:::
:::





## G-Computation: Risk Difference

Rather than assuming a correctly specified working model, DR-WLS can be used to infer about the *marginal risk difference*:


::: {.cell}

```{.r .cell-code}
licorice_30_min_any_pain_drwls_rd <-
  drwls::drwls(
    outcome_formula = 
      pacu_30_min_any_throat_pain ~ tx +
      age_bl + gender + asa_physical_status_bl + bmi_bl + mallampati_bl,
    outcome_family = binomial,
    treatment_column = "tx",
    estimand = "difference", # Compute Risk Difference
    missing_formula = ~ tx + age_bl + bmi_bl,
    missing_family = binomial,
    data = licorice_gargle,
    se_method = "asymptotic"
  )

licorice_30_min_any_pain_drwls_rd
```

::: {.cell-output .cell-output-stdout}

```

Estimand: Risk Difference 
Method: Doubly-Robust Weighted Least Squares
Outcome Models: a quasibinomial GLM with a logit link
Censoring Model: a binomial GLM using a logit link 

             Estimand   Estimate         SE        LCL         UCL         Z
1 E[Y|A=1] - E[Y|A=0] -0.1598020 0.05575978 -0.2690892 -0.05051483 -2.865901
2            E[Y|A=1]  0.1940108         NA         NA          NA        NA
3            E[Y|A=0]  0.3538128         NA         NA          NA        NA
      p-value
1 0.004158249
2          NA
3          NA

Variance Estimation: Sandwich Estimator 
CI Method: Wald 95% CI 
Details:
  Outcomes: (N = 233/235)
  Outcome Model: pacu_30_min_any_throat_pain ~ tx + age_bl + gender + asa_physical_status_bl +      bmi_bl + mallampati_bl
  Missing Formula: !is.na(pacu_30_min_any_throat_pain) ~ tx + age_bl + bmi_bl
```


:::
:::





## G-Computation: Marginal Odds Ratio

A *marginal odds ratio* can be computed, but this 


::: {.cell}

```{.r .cell-code}
licorice_30_min_any_pain_drwls_or <-
  drwls::drwls(
    outcome_formula = 
      pacu_30_min_any_throat_pain ~ tx +
      age_bl + gender + asa_physical_status_bl + bmi_bl + mallampati_bl,
    outcome_family = binomial,
    treatment_column = "tx",
    estimand = "oddsratio", # Compute Odds Ratio
    missing_formula = ~ tx + age_bl + bmi_bl,
    missing_family = binomial,
    data = licorice_gargle,
    se_method = "bootstrap"
  )

licorice_30_min_any_pain_drwls_or
```

::: {.cell-output .cell-output-stdout}

```

Estimand: Odds Ratio 
Method: Doubly-Robust Weighted Least Squares
Outcome Models: a quasibinomial GLM with a logit link
Censoring Model: a binomial GLM using a logit link 

                 Estimand  Estimate         SE       LCL       UCL        Z
1 Odds[Y|A=1]/Odds[Y|A=0] 0.4396241 0.13941328 0.2157508 0.8544483 -4.01953
2                E[Y|A=1] 0.1940108 0.03665870 0.1189380 0.2847232       NA
3                E[Y|A=0] 0.3538128 0.04394461 0.2600795 0.4566709       NA
  p-value
1      NA
2      NA
3      NA

Variance Estimation: Non-parametric bootstrap using 10000 replicates 
CI Method: Bias Corrected and Accelerated (BCa) 95% CI 
Details:
  Outcomes: (N = 233/235)
  Outcome Model: pacu_30_min_any_throat_pain ~ tx + age_bl + gender + asa_physical_status_bl +      bmi_bl + mallampati_bl
  Missing Formula: !is.na(pacu_30_min_any_throat_pain) ~ tx + age_bl + bmi_bl
```


:::
:::


:::




--------------------------------------------------------------------------------



# Ordinal: Streptomycin in TB

There are several estimands that may be of interest in an ordinal outcome,
including the Mann-Whitney estimand, log odds ratio, and expected utility.
Covariate adjusted estimates of these quantities that are robust to model
misspecification are available for each of these estimands
[@Diaz2015Enhanced; @Benkeser2020Improving]. These methods will be illustrated
using a randomized trial for streptomycin in Tuberculosis.

::: {.panel-tabset}



## Data Dictionary

The `strep_tb` dataset contains baseline covariates and outcomes at 6 months
post-randomization. The primary outcome at 6 months is an ordinal scale,
combining survival and change in disease based on radiologic measures. While
the outcome is categorized in binary terms as "not improved" or "improved", the
ordinal scale may contain more information.


::: {.cell}

```{.r .cell-code}
data.frame(
  column = names(strep_tb),
  name =
    sapply(X = strep_tb, function(x) attr(x = x, which = "label")),
  row.names = NULL
) %>%
  kableExtra::kable(
    x = .,
    caption = "Columns of the `strep_tb` dataset."
  )
```

::: {.cell-output-display}


Table: Columns of the `strep_tb` dataset.

|column                      |name                                 |
|:---------------------------|:------------------------------------|
|patient_id                  |Participant ID                       |
|arm                         |Study Arm                            |
|tx                          |Treatment Indicator                  |
|gender                      |Gender                               |
|condition_bl                |Condition (BL)                       |
|temperature_bl              |Temperature  (BL)                    |
|sed_rate_esr_bl             |ESR: Sedimentation Rate (BL)         |
|cxr_lung_cavitation_bl      |Lung Cavitation on X-Ray (BL)        |
|strep_resistance_6m         |Strep Resistance - Numeric (6M)      |
|strep_resistance_f_6m       |Strep Resistance (6M)                |
|radiologic_outcome_6m       |Radiologic Outcome - Numeric(6M)     |
|radiologic_outcome_f_6m     |Radiologic Outcome (6M)              |
|radiologic_improvement_6m   |Radiologic Improvement - Binary (6M) |
|radiologic_improvement_f_6m |Radiologic Improvement (6M)          |


:::
:::





## Tabulations: BL

As seen from the tabulations, there are low frequencies that need to be addressed, as well as a missing value in the covariates.


::: {.cell}

```{.r .cell-code}
table1::table1(
  x = ~ gender + condition_bl + sed_rate_esr_bl + cxr_lung_cavitation_bl |
    arm,
  data = strep_tb
)
```

::: {.cell-output-display}

```{=html}
<div class="Rtable1"><table class="Rtable1">
<thead>
<tr>
<th class='rowlabel firstrow lastrow'></th>
<th class='firstrow lastrow'><span class='stratlabel'>0. Control<br/><span class='stratn'>(N=52)</span></span></th>
<th class='firstrow lastrow'><span class='stratlabel'>1. Streptomycin<br/><span class='stratn'>(N=55)</span></span></th>
<th class='firstrow lastrow'><span class='stratlabel'>Overall<br/><span class='stratn'>(N=107)</span></span></th>
</tr>
</thead>
<tbody>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Gender</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>0. Female</td>
<td>28 (53.8%)</td>
<td>31 (56.4%)</td>
<td>59 (55.1%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>1. Male</td>
<td class='lastrow'>24 (46.2%)</td>
<td class='lastrow'>24 (43.6%)</td>
<td class='lastrow'>48 (44.9%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Condition (BL)</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>1. Good</td>
<td>8 (15.4%)</td>
<td>8 (14.5%)</td>
<td>16 (15.0%)</td>
</tr>
<tr>
<td class='rowlabel'>2. Fair</td>
<td>20 (38.5%)</td>
<td>17 (30.9%)</td>
<td>37 (34.6%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>3. Poor</td>
<td class='lastrow'>24 (46.2%)</td>
<td class='lastrow'>30 (54.5%)</td>
<td class='lastrow'>54 (50.5%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>ESR: Sedimentation Rate (BL)</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>2. 11-20</td>
<td>2 (3.8%)</td>
<td>3 (5.5%)</td>
<td>5 (4.7%)</td>
</tr>
<tr>
<td class='rowlabel'>3. 21-50</td>
<td>20 (38.5%)</td>
<td>16 (29.1%)</td>
<td>36 (33.6%)</td>
</tr>
<tr>
<td class='rowlabel'>4. 51+</td>
<td>29 (55.8%)</td>
<td>36 (65.5%)</td>
<td>65 (60.7%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>Missing</td>
<td class='lastrow'>1 (1.9%)</td>
<td class='lastrow'>0 (0%)</td>
<td class='lastrow'>1 (0.9%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Lung Cavitation on X-Ray (BL)</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>0. No</td>
<td>22 (42.3%)</td>
<td>23 (41.8%)</td>
<td>45 (42.1%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>1. Yes</td>
<td class='lastrow'>30 (57.7%)</td>
<td class='lastrow'>32 (58.2%)</td>
<td class='lastrow'>62 (57.9%)</td>
</tr>
</tbody>
</table>
</div>
```

:::
:::





## Tabulations: Outcomes

Low frequencies in the radiologic outcomes also require pooling.


::: {.cell}

```{.r .cell-code}
table1::table1(
  x = 
    ~ strep_resistance_f_6m +
    radiologic_outcome_f_6m +
    radiologic_improvement_f_6m | arm,
  data = strep_tb
)
```

::: {.cell-output-display}

```{=html}
<div class="Rtable1"><table class="Rtable1">
<thead>
<tr>
<th class='rowlabel firstrow lastrow'></th>
<th class='firstrow lastrow'><span class='stratlabel'>0. Control<br/><span class='stratn'>(N=52)</span></span></th>
<th class='firstrow lastrow'><span class='stratlabel'>1. Streptomycin<br/><span class='stratn'>(N=55)</span></span></th>
<th class='firstrow lastrow'><span class='stratlabel'>Overall<br/><span class='stratn'>(N=107)</span></span></th>
</tr>
</thead>
<tbody>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Strep Resistance (6M)</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>1. 0-8: Sensitive</td>
<td>52 (100%)</td>
<td>13 (23.6%)</td>
<td>65 (60.7%)</td>
</tr>
<tr>
<td class='rowlabel'>2. 8-99: Moderate</td>
<td>0 (0%)</td>
<td>8 (14.5%)</td>
<td>8 (7.5%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>3. 100+: Resistant</td>
<td class='lastrow'>0 (0%)</td>
<td class='lastrow'>34 (61.8%)</td>
<td class='lastrow'>34 (31.8%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Radiologic Outcome (6M)</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>1. Death</td>
<td>14 (26.9%)</td>
<td>4 (7.3%)</td>
<td>18 (16.8%)</td>
</tr>
<tr>
<td class='rowlabel'>2. Considerable Deterioration</td>
<td>6 (11.5%)</td>
<td>6 (10.9%)</td>
<td>12 (11.2%)</td>
</tr>
<tr>
<td class='rowlabel'>3. Moderate Deterioration</td>
<td>12 (23.1%)</td>
<td>5 (9.1%)</td>
<td>17 (15.9%)</td>
</tr>
<tr>
<td class='rowlabel'>4. No Change</td>
<td>3 (5.8%)</td>
<td>2 (3.6%)</td>
<td>5 (4.7%)</td>
</tr>
<tr>
<td class='rowlabel'>5. Moderate Improvement</td>
<td>13 (25.0%)</td>
<td>10 (18.2%)</td>
<td>23 (21.5%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>6. Considerable Improvement</td>
<td class='lastrow'>4 (7.7%)</td>
<td class='lastrow'>28 (50.9%)</td>
<td class='lastrow'>32 (29.9%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Radiologic Improvement (6M)</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>0. Not Improved</td>
<td>35 (67.3%)</td>
<td>17 (30.9%)</td>
<td>52 (48.6%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>1. Improved</td>
<td class='lastrow'>17 (32.7%)</td>
<td class='lastrow'>38 (69.1%)</td>
<td class='lastrow'>55 (51.4%)</td>
</tr>
</tbody>
</table>
</div>
```

:::
:::




## Pooling Categories

For this example, category 4 ("No Change") will be pooled with "Moderate Deterioration". Sedimentation rate will be pooled into a binary variable. Since only one value is missing in a binary variable, a value imputing each possibility can be computed, and the sensitivity of imputation assessed.


::: {.cell}

```{.r .cell-code}
strep_tb_pooled <-
  strep_tb %>% 
  dplyr::mutate(
    radiologic_outcome_pool_6m =
      dplyr::case_when(
        radiologic_outcome_6m %in% 1:2 ~ radiologic_outcome_6m,
        radiologic_outcome_6m %in% 3:4 ~ 3,
        radiologic_outcome_6m %in% 5:6 ~ radiologic_outcome_6m - 1
      ),
    condition_bl_pool =
      dplyr::case_when(
        condition_bl %in% c("1. Good", "2. Fair") ~ "0. Good/Fair",
        condition_bl %in% c("3. Poor") ~ "1. Poor",
      ) %>% 
      factor(),
    sed_rate_esr_lte_50_bl_imp_0 =
      dplyr::case_when(
        sed_rate_esr_bl %in% c(NA, "2. 11-20", "3. 21-50") ~ "0. <= 50",
        sed_rate_esr_bl %in% c("4. 51+") ~ "1. > 50"
      ) %>% 
      factor(),
    sed_rate_esr_lte_50_bl_imp_1 =
      dplyr::case_when(
        sed_rate_esr_bl %in% c("2. 11-20", "3. 21-50") ~ "0. <= 50",
        sed_rate_esr_bl %in% c(NA, "4. 51+") ~ "1. > 50"
      ) %>%
      factor()
  )

# Check Pooled Variable
strep_tb_pooled %>% 
  dplyr::count(radiologic_outcome_6m, radiologic_outcome_pool_6m)
```

::: {.cell-output .cell-output-stdout}

```
  radiologic_outcome_6m radiologic_outcome_pool_6m  n
1                     1                          1 18
2                     2                          2 12
3                     3                          3 17
4                     4                          3  5
5                     5                          4 23
6                     6                          5 32
```


:::

```{.r .cell-code}
# Check Pooled + Imputed Covariate
strep_tb_pooled %>% 
  dplyr::count(
    sed_rate_esr_bl, sed_rate_esr_lte_50_bl_imp_0, sed_rate_esr_lte_50_bl_imp_1
  )
```

::: {.cell-output .cell-output-stdout}

```
  sed_rate_esr_bl sed_rate_esr_lte_50_bl_imp_0 sed_rate_esr_lte_50_bl_imp_1  n
1        2. 11-20                     0. <= 50                     0. <= 50  5
2        3. 21-50                     0. <= 50                     0. <= 50 36
3          4. 51+                      1. > 50                      1. > 50 65
4            <NA>                     0. <= 50                      1. > 50  1
```


:::
:::




## Imbalance


::: {.cell}

```{.r .cell-code}
cobalt::bal.tab(
  x = 
    # Only tabulate baseline variables
    strep_tb_pooled %>% 
    dplyr::select(
      dplyr::all_of(
        x = c("gender", "condition_bl_pool", 
              "sed_rate_esr_lte_50_bl_imp_0", "sed_rate_esr_lte_50_bl_imp_1")
      )
    ),
  treat = strep_tb_pooled$arm,
  # Compute standardized differences for both binary and continuous variables
  binary = "std",
  continuous = "std",
  s.d.denom = "pooled"
)
```

::: {.cell-output .cell-output-stdout}

```
Balance Measures
                                       Type Diff.Un
gender_1. Male                       Binary -0.0506
condition_bl_pool_1. Poor            Binary  0.1684
sed_rate_esr_lte_50_bl_imp_0_1. > 50 Binary  0.1992
sed_rate_esr_lte_50_bl_imp_1_1. > 50 Binary  0.1601

Sample sizes
    0. Control 1. Streptomycin
All         52              55
```


:::
:::





## Helper Function

A helper function can make it easier to extract and report results with `drord::drord`:


::: {.cell}

```{.r .cell-code}
summary.drord <-
  function(object, ...){
    results_list <- list()
    for(i in c("mann_whitney", "log_odds", "weighted_mean")){
      if(i == "weighted_mean") {
        object[["weighted_mean"]]$est <- object[["weighted_mean"]]$est$est
      }
      if(i == "mann_whitney") {
        object[["mann_whitney"]]$ci <-
          lapply(
            X = object[["mann_whitney"]]$ci,
            FUN = function(x) if(!is.null(x)) matrix(x, ncol = 2)
          )
      }
      if(i %in% names(object)) results_list[[i]] <- c(results_list, object[[i]])
    }
    ci_wald <- "wald" %in% object$ci
    ci_bca <- "bca" %in% object$ci
    alpha <- object$alpha
    
    for(i in 1:length(results_list)){
      null_i <-
        switch(
          EXPR = names(results_list)[i],
          "mann_whitney" = c(0.5),
          "log_odds" = c(NA, NA, 0),
          "weighted_mean" = c(NA, NA, 0)
        )
      
      estimand_i <-
        switch(
          EXPR = names(results_list)[i],
          "mann_whitney" = "Mann-Whitney",
          "log_odds" = c("Log Odds: A = 1", "Log Odds: A = 1", "Log Odds Ratio: 1/0"),
          "weighted_mean" = c("Mean: A = 1", "Mean: A = 0", "Difference: 1-0")
        )
      
      results_list[[i]] <-
        with(
          data = results_list[[i]],
          expr =
            data.frame(
              estimand = estimand_i,
              estimate = est,
              null_value = null_i,
              se = 
                if(ci_wald){(ci$wald[, 2] - est)/qnorm(p = 1 - alpha/2)} else {NA},
              wald_lcl = if(ci_wald){ci$wald[, 1]} else {NA},
              wald_ucl = if(ci_wald){ci$wald[, 2]} else {NA},
              bca_lcl = if(ci_bca){ci$bca[, 1]} else {NA},
              bca_ucl = if(ci_bca){ci$bca[, 2]} else {NA}
            )
        )
    }
    
    results_table <- do.call(what = rbind, args = results_list)
    rownames(results_table) <- NULL
    results_table$wald_z <- 
      with(
        data = results_table,
        expr = (estimate - null_value)/se
      )
    
    results_table$`p-value` <-
      with(
        data = results_table,
        expr = 2*pnorm(q = -abs(wald_z))
      )
    
    if(!ci_wald){
      results_table[, 
                    c("se", "wald_lcl", "wald_ucl", "wald_z", "p-value")] <- NULL
    }
    
    if(!ci_bca){
      results_table[, c("bca_lcl", "bca_ucl")] <- NULL
    }
    
    return(
      results_table
    )
  }
```
:::


This allows us to back-calculate standard errors and test statistics from Wald confidence intervals.




## Ordinal: Unadjusted


::: {.cell}

```{.r .cell-code}
strep_rad_6m_unadjusted <-
  with(
    data = strep_tb_pooled,
    expr = {
      drord::drord(
        out = as.numeric(radiologic_outcome_pool_6m),
        treat = tx,
        covar = data.frame(gender, condition_bl, sed_rate_esr_bl,
                           cxr_lung_cavitation_bl),
        out_form = "1", # Unadjusted - Intercept Only
        treat_form = "1", # Unadjusted - Intercept Only
      )
    }
  )

strep_rad_6m_unadjusted_table <-
  summary.drord(strep_rad_6m_unadjusted)

kableExtra::kbl(
  x = strep_rad_6m_unadjusted_table,
  digits = c(rep(x = 2, 7), 4),
  caption = "Unadjusted analyses of the `strep_tb` dataset."
)
```

::: {.cell-output-display}
`````{=html}
<table>
<caption>Unadjusted analyses of the `strep_tb` dataset.</caption>
 <thead>
  <tr>
   <th style="text-align:left;"> estimand </th>
   <th style="text-align:right;"> estimate </th>
   <th style="text-align:right;"> null_value </th>
   <th style="text-align:right;"> se </th>
   <th style="text-align:right;"> wald_lcl </th>
   <th style="text-align:right;"> wald_ucl </th>
   <th style="text-align:right;"> wald_z </th>
   <th style="text-align:right;"> p-value </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> Mann-Whitney </td>
   <td style="text-align:right;"> 0.75 </td>
   <td style="text-align:right;"> 0.5 </td>
   <td style="text-align:right;"> 0.06 </td>
   <td style="text-align:right;"> 0.63 </td>
   <td style="text-align:right;"> 0.86 </td>
   <td style="text-align:right;"> 4.13 </td>
   <td style="text-align:right;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Log Odds: A = 1 </td>
   <td style="text-align:right;"> -1.22 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> 0.29 </td>
   <td style="text-align:right;"> -1.79 </td>
   <td style="text-align:right;"> -0.66 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Log Odds: A = 1 </td>
   <td style="text-align:right;"> 0.43 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> 0.26 </td>
   <td style="text-align:right;"> -0.08 </td>
   <td style="text-align:right;"> 0.95 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Log Odds Ratio: 1/0 </td>
   <td style="text-align:right;"> -1.66 </td>
   <td style="text-align:right;"> 0.0 </td>
   <td style="text-align:right;"> 0.39 </td>
   <td style="text-align:right;"> -2.42 </td>
   <td style="text-align:right;"> -0.89 </td>
   <td style="text-align:right;"> -4.25 </td>
   <td style="text-align:right;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Mean: A = 1 </td>
   <td style="text-align:right;"> 3.95 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> 0.18 </td>
   <td style="text-align:right;"> 3.60 </td>
   <td style="text-align:right;"> 4.29 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Mean: A = 0 </td>
   <td style="text-align:right;"> 2.75 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> 0.18 </td>
   <td style="text-align:right;"> 2.40 </td>
   <td style="text-align:right;"> 3.10 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Difference: 1-0 </td>
   <td style="text-align:right;"> 1.20 </td>
   <td style="text-align:right;"> 0.0 </td>
   <td style="text-align:right;"> 0.25 </td>
   <td style="text-align:right;"> 0.70 </td>
   <td style="text-align:right;"> 1.69 </td>
   <td style="text-align:right;"> 4.71 </td>
   <td style="text-align:right;"> 0 </td>
  </tr>
</tbody>
</table>

`````
:::
:::





## Ordinal: Adjusted


::: {.cell}

```{.r .cell-code}
strep_rad_6m_adjusted <-
  with(
    data = strep_tb_pooled,
    expr = {
      drord::drord(
        out = as.numeric(radiologic_outcome_pool_6m),
        treat = tx,
        covar = 
          data.frame(gender, condition_bl_pool, 
                     sed_rate_esr_lte_50_bl_imp_1, cxr_lung_cavitation_bl),
        out_form = "gender + condition_bl_pool + sed_rate_esr_lte_50_bl_imp_1",
        treat_form = "gender + condition_bl_pool + sed_rate_esr_lte_50_bl_imp_1"
      )
    }
  )

strep_rad_6m_adjusted_table <-
  summary.drord(strep_rad_6m_adjusted)

kableExtra::kbl(
  x = strep_rad_6m_adjusted_table,
  digits = c(rep(x = 2, 7), 4),
  caption = "Adjusted analyses of the `strep_tb` dataset."
)
```

::: {.cell-output-display}
`````{=html}
<table>
<caption>Adjusted analyses of the `strep_tb` dataset.</caption>
 <thead>
  <tr>
   <th style="text-align:left;"> estimand </th>
   <th style="text-align:right;"> estimate </th>
   <th style="text-align:right;"> null_value </th>
   <th style="text-align:right;"> se </th>
   <th style="text-align:right;"> wald_lcl </th>
   <th style="text-align:right;"> wald_ucl </th>
   <th style="text-align:right;"> wald_z </th>
   <th style="text-align:right;"> p-value </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> Mann-Whitney </td>
   <td style="text-align:right;"> 0.77 </td>
   <td style="text-align:right;"> 0.5 </td>
   <td style="text-align:right;"> 0.05 </td>
   <td style="text-align:right;"> 0.67 </td>
   <td style="text-align:right;"> 0.87 </td>
   <td style="text-align:right;"> 5.53 </td>
   <td style="text-align:right;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Log Odds: A = 1 </td>
   <td style="text-align:right;"> -1.29 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> 0.27 </td>
   <td style="text-align:right;"> -1.82 </td>
   <td style="text-align:right;"> -0.76 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Log Odds: A = 1 </td>
   <td style="text-align:right;"> 0.57 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> 0.22 </td>
   <td style="text-align:right;"> 0.14 </td>
   <td style="text-align:right;"> 1.00 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Log Odds Ratio: 1/0 </td>
   <td style="text-align:right;"> -1.86 </td>
   <td style="text-align:right;"> 0.0 </td>
   <td style="text-align:right;"> 0.31 </td>
   <td style="text-align:right;"> -2.47 </td>
   <td style="text-align:right;"> -1.24 </td>
   <td style="text-align:right;"> -5.92 </td>
   <td style="text-align:right;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Mean: A = 1 </td>
   <td style="text-align:right;"> 3.99 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> 0.16 </td>
   <td style="text-align:right;"> 3.67 </td>
   <td style="text-align:right;"> 4.31 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Mean: A = 0 </td>
   <td style="text-align:right;"> 2.65 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> 0.14 </td>
   <td style="text-align:right;"> 2.37 </td>
   <td style="text-align:right;"> 2.93 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Difference: 1-0 </td>
   <td style="text-align:right;"> 1.34 </td>
   <td style="text-align:right;"> 0.0 </td>
   <td style="text-align:right;"> 0.19 </td>
   <td style="text-align:right;"> 0.96 </td>
   <td style="text-align:right;"> 1.71 </td>
   <td style="text-align:right;"> 6.99 </td>
   <td style="text-align:right;"> 0 </td>
  </tr>
</tbody>
</table>

`````
:::
:::





## Precision Gain

As seen below, there is appreciable precision gain across each estimand from adjustment.


::: {.cell}

```{.r .cell-code}
est_labels <- c("Mann-Whitney", "Log Odds Ratio: 1/0", "Difference: 1-0")
est_rows <- which(strep_rad_6m_adjusted_table$estimand %in% est_labels)

strep_tb_gain <- 
  setNames(
    object = 1 - (strep_rad_6m_adjusted_table$se[est_rows]/
                    strep_rad_6m_unadjusted_table$se[est_rows])^2,
    nm = est_labels
  )

strep_tb_gain
```

::: {.cell-output .cell-output-stdout}

```
       Mann-Whitney Log Odds Ratio: 1/0     Difference: 1-0 
          0.3348912           0.3527213           0.4304210 
```


:::
:::


:::




--------------------------------------------------------------------------------



# Time-to-Event: Colon Cancer

The Cox proportional hazards (PH) model is commonly used in time-to-event
outcomes. Use of the PH model entails two assumptions: the assumption of non-informative censoring and the assumption of proportional hazards.

Non-informative censoring assumes that within any subgroup of interest, the
event rates among the censored are equal to those who remain uncensored [@Kleinbaum2012].

The proportional hazards assumption does not specify the hazard function for
each arm, but does require that their ratio is approximately constant during
the follow-up period. Mild departures from this assumption still allow the
estimated hazard ratio to be interpreted as a weighted average hazard ratio
over the follow-up time. Larger departures from this assumption can lead to
questions of interpretation and loss of power.

The restricted mean survival time (RMST) and survival probability are two
estimands that do not require the proportional hazards assumption. Instead, a
clinically meaningful time horizon $\tau$ is specified, and the RMST or survival
probability is computed at the time $\tau$.

The RMST can be interpreted as the difference in time individuals can expect to
remain event free if they were assigned to the treatment arm instead of control.

The RMST and survival probability at time $\tau$ both provide a meaningful
effect measure that does not require the proportional hazards assumption.

These approaches will be illustrated in a randomized trial of chemotherapy for
colon cancer.


::: {.panel-tabset}

## Data Dictionary


::: {.cell}

```{.r .cell-code}
data.frame(
  column = names(colon_cancer_active),
  name =
    sapply(X = colon_cancer_active, function(x) attr(x = x, which = "label")),
  row.names = NULL
) %>%
  kableExtra::kable(
    x = .,
    caption = "Columns of the `colon_cancer_active` dataset."
  )
```

::: {.cell-output-display}


Table: Columns of the `colon_cancer_active` dataset.

|column                       |name                               |
|:----------------------------|:----------------------------------|
|participant_id               |Participant ID                     |
|arm                          |Study Arm                          |
|age_bl                       |Age (BL)                           |
|sex                          |Sex (BL)                           |
|obstruction_bl               |Tumor Obstructs Colon (BL)         |
|perforation_bl               |Tumor Perforates Colon (BL)        |
|organ_adherence_bl           |Tumor Organ Adherence (BL)         |
|positive_nodes_bl            |Lymph Nodes with Cancer (BL)       |
|differentiation_bl           |Tumor Differentiation (BL)         |
|local_spread_bl              |Extent of Tumor Spread (BL)        |
|time_surgery_registration_bl |Time: Surgery to Registration (BL) |
|event_death                  |Death On Study                     |
|time_to_death                |Time to Death On Study (d)         |
|event_recurrence             |Recurrence On Study                |
|time_to_recurrence           |Time to Recurrence On Study (d)    |
|tx                           |Treatment Indicator                |
|years_to_death               |Time to Death On Study (Y)         |
|years_to_recurrence          |Time to Recurrence On Study (Y)    |


:::
:::





## Tabulations: BL

No covariates are missing, however, `local_spread_bl` and `perforation_bl` have low frequency categories.


::: {.cell}

```{.r .cell-code}
table1::table1(
  x = ~ age_bl + sex + obstruction_bl + perforation_bl + organ_adherence_bl +
    positive_nodes_bl + differentiation_bl + local_spread_bl + 
    time_surgery_registration_bl |
    arm,
  data = colon_cancer_active
)
```

::: {.cell-output-display}

```{=html}
<div class="Rtable1"><table class="Rtable1">
<thead>
<tr>
<th class='rowlabel firstrow lastrow'></th>
<th class='firstrow lastrow'><span class='stratlabel'>0. Lev<br/><span class='stratn'>(N=310)</span></span></th>
<th class='firstrow lastrow'><span class='stratlabel'>1. Lev+5FU<br/><span class='stratn'>(N=304)</span></span></th>
<th class='firstrow lastrow'><span class='stratlabel'>Overall<br/><span class='stratn'>(N=614)</span></span></th>
</tr>
</thead>
<tbody>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Age (BL)</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>Mean (SD)</td>
<td>60.1 (11.6)</td>
<td>59.7 (12.3)</td>
<td>59.9 (11.9)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>Median [Min, Max]</td>
<td class='lastrow'>61.0 [27.0, 83.0]</td>
<td class='lastrow'>62.0 [26.0, 81.0]</td>
<td class='lastrow'>61.0 [26.0, 83.0]</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Sex (BL)</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>0. Female</td>
<td>133 (42.9%)</td>
<td>163 (53.6%)</td>
<td>296 (48.2%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>1. Male</td>
<td class='lastrow'>177 (57.1%)</td>
<td class='lastrow'>141 (46.4%)</td>
<td class='lastrow'>318 (51.8%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Tumor Obstructs Colon (BL)</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>0. No</td>
<td>247 (79.7%)</td>
<td>250 (82.2%)</td>
<td>497 (80.9%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>1. Yes</td>
<td class='lastrow'>63 (20.3%)</td>
<td class='lastrow'>54 (17.8%)</td>
<td class='lastrow'>117 (19.1%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Tumor Perforates Colon (BL)</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>0. No</td>
<td>300 (96.8%)</td>
<td>296 (97.4%)</td>
<td>596 (97.1%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>1. Yes</td>
<td class='lastrow'>10 (3.2%)</td>
<td class='lastrow'>8 (2.6%)</td>
<td class='lastrow'>18 (2.9%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Tumor Organ Adherence (BL)</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>0. No</td>
<td>261 (84.2%)</td>
<td>265 (87.2%)</td>
<td>526 (85.7%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>1. Yes</td>
<td class='lastrow'>49 (15.8%)</td>
<td class='lastrow'>39 (12.8%)</td>
<td class='lastrow'>88 (14.3%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Lymph Nodes with Cancer (BL)</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>Mean (SD)</td>
<td>3.68 (3.53)</td>
<td>3.51 (3.39)</td>
<td>3.60 (3.46)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>Median [Min, Max]</td>
<td class='lastrow'>2.00 [0, 33.0]</td>
<td class='lastrow'>2.00 [1.00, 24.0]</td>
<td class='lastrow'>2.00 [0, 33.0]</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Tumor Differentiation (BL)</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>1. Well</td>
<td>37 (11.9%)</td>
<td>29 (9.5%)</td>
<td>66 (10.7%)</td>
</tr>
<tr>
<td class='rowlabel'>2. Moderate</td>
<td>225 (72.6%)</td>
<td>219 (72.0%)</td>
<td>444 (72.3%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>3. Poor</td>
<td class='lastrow'>48 (15.5%)</td>
<td class='lastrow'>56 (18.4%)</td>
<td class='lastrow'>104 (16.9%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Extent of Tumor Spread (BL)</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>1. Submucosa</td>
<td>3 (1.0%)</td>
<td>10 (3.3%)</td>
<td>13 (2.1%)</td>
</tr>
<tr>
<td class='rowlabel'>2. Muscle</td>
<td>36 (11.6%)</td>
<td>32 (10.5%)</td>
<td>68 (11.1%)</td>
</tr>
<tr>
<td class='rowlabel'>3. Serosa</td>
<td>259 (83.5%)</td>
<td>251 (82.6%)</td>
<td>510 (83.1%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>4. Contiguous structures</td>
<td class='lastrow'>12 (3.9%)</td>
<td class='lastrow'>11 (3.6%)</td>
<td class='lastrow'>23 (3.7%)</td>
</tr>
<tr>
<td class='rowlabel firstrow'><span class='varlabel'>Time: Surgery to Registration (BL)</span></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
<td class='firstrow'></td>
</tr>
<tr>
<td class='rowlabel'>0. Short</td>
<td>230 (74.2%)</td>
<td>228 (75.0%)</td>
<td>458 (74.6%)</td>
</tr>
<tr>
<td class='rowlabel lastrow'>1. Long</td>
<td class='lastrow'>80 (25.8%)</td>
<td class='lastrow'>76 (25.0%)</td>
<td class='lastrow'>156 (25.4%)</td>
</tr>
</tbody>
</table>
</div>
```

:::
:::





## Pooling Categories


::: {.cell}

```{.r .cell-code}
colon_cancer_active_pool <-
  colon_cancer_active %>% 
  dplyr::mutate(
    local_spread_bl_pool =
      dplyr::case_when(
        local_spread_bl %in% c("1. Submucosa", "2. Muscle") ~ 
          "0. Submucosa or Muscle",
        local_spread_bl %in% c("3. Serosa", "4. Contiguous structures") ~ 
          "1. Serosa or Contiguous Structures",
      ) %>% 
      factor
  )

colon_cancer_active_pool %>% 
  count(local_spread_bl, local_spread_bl_pool)
```

::: {.cell-output .cell-output-stdout}

```
           local_spread_bl               local_spread_bl_pool   n
1             1. Submucosa             0. Submucosa or Muscle  13
2                2. Muscle             0. Submucosa or Muscle  68
3                3. Serosa 1. Serosa or Contiguous Structures 510
4 4. Contiguous structures 1. Serosa or Contiguous Structures  23
```


:::
:::





## Imbalance

Most of the covariates in this trial show fairly good balance with the exception of `sex`.


::: {.cell}

```{.r .cell-code}
cobalt::bal.tab(
  x = 
    # Only tabulate baseline variables
    colon_cancer_active_pool %>% 
    dplyr::select(
      dplyr::all_of(
        x = c("age_bl", "sex", "obstruction_bl", "perforation_bl",
              "organ_adherence_bl", "positive_nodes_bl", "differentiation_bl",
              "local_spread_bl_pool", "time_surgery_registration_bl")
      )
    ),
  treat = colon_cancer_active_pool$arm,
  # Compute standardized differences for both binary and continuous variables
  binary = "std",
  continuous = "std",
  s.d.denom = "pooled"
)
```

::: {.cell-output .cell-output-stdout}

```
Balance Measures
                                                           Type Diff.Un
age_bl                                                  Contin. -0.0345
sex_1. Male                                              Binary -0.2157
obstruction_bl_1. Yes                                    Binary -0.0652
perforation_bl_1. Yes                                    Binary -0.0352
organ_adherence_bl_1. Yes                                Binary -0.0851
positive_nodes_bl                                       Contin. -0.0493
differentiation_bl_1. Well                               Binary -0.0775
differentiation_bl_2. Moderate                           Binary -0.0121
differentiation_bl_3. Poor                               Binary  0.0783
local_spread_bl_pool_1. Serosa or Contiguous Structures  Binary -0.0365
time_surgery_registration_bl_1. Long                     Binary -0.0185

Sample sizes
    0. Lev 1. Lev+5FU
All    310        304
```


:::
:::





## Logrank Test

The `survival::survdiff` function carries out the $G^{\rho}$ family of tests for time-to-event outcomes. The default value of the `rho` parameter gives the logrank test:


::: {.cell}

```{.r .cell-code}
survival::survdiff(
  formula = 
    survival::Surv(time = time_to_death, event = event_death) ~ tx,
  data = colon_cancer_active_pool
)
```

::: {.cell-output .cell-output-stdout}

```
Call:
survival::survdiff(formula = survival::Surv(time = time_to_death, 
    event = event_death) ~ tx, data = colon_cancer_active_pool)

       N Observed Expected (O-E)^2/E (O-E)^2/V
tx=0 310      161      137      4.24      8.21
tx=1 304      123      147      3.95      8.21

 Chisq= 8.2  on 1 degrees of freedom, p= 0.004 
```


:::
:::





## Unadjusted Cox

The unadjusted Cox proportional hazards model can be fit as follows to obtain the marginal hazard ratio:


::: {.cell}

```{.r .cell-code}
colon_cancer_cox_unadjusted <-
  survival::coxph(
    formula = survival::Surv(time = time_to_death, event = event_death) ~ tx,
    data = colon_cancer_active_pool,
    robust = TRUE # Use Sandwich Standard Errors
  ) 

summary(colon_cancer_cox_unadjusted)
```

::: {.cell-output .cell-output-stdout}

```
Call:
survival::coxph(formula = survival::Surv(time = time_to_death, 
    event = event_death) ~ tx, data = colon_cancer_active_pool, 
    robust = TRUE)

  n= 614, number of events= 284 

      coef exp(coef) se(coef) robust se      z Pr(>|z|)   
tx -0.3417    0.7106   0.1199    0.1198 -2.851  0.00435 **
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

   exp(coef) exp(-coef) lower .95 upper .95
tx    0.7106      1.407    0.5618    0.8987

Concordance= 0.541  (se = 0.015 )
Likelihood ratio test= 8.21  on 1 df,   p=0.004
Wald test            = 8.13  on 1 df,   p=0.004
Score (logrank) test = 8.21  on 1 df,   p=0.004,   Robust = 8.18  p=0.004

  (Note: the likelihood ratio and score tests assume independence of
     observations within a cluster, the Wald and robust score tests do not).
```


:::
:::


Before interpreting the Hazard Ratio, the proportional hazards assumption can be checked with `survival::cox.zph()`:


::: {.cell}

```{.r .cell-code}
survival::cox.zph(colon_cancer_cox_unadjusted)
```

::: {.cell-output .cell-output-stdout}

```
       chisq df    p
tx     0.112  1 0.74
GLOBAL 0.112  1 0.74
```


:::
:::





## Adjusted Conditional Cox


::: {.cell}

```{.r .cell-code}
colon_cancer_cox_adjusted <-
  survival::coxph(
    formula = 
      survival::Surv(time = time_to_death, event = event_death) ~ tx +
      age_bl + sex + obstruction_bl + organ_adherence_bl +
      positive_nodes_bl + differentiation_bl + local_spread_bl_pool +
      time_surgery_registration_bl,
    data = colon_cancer_active_pool,
    robust = TRUE # Use Sandwich Standard Errors
  ) 

summary(colon_cancer_cox_adjusted)
```

::: {.cell-output .cell-output-stdout}

```
Call:
survival::coxph(formula = survival::Surv(time = time_to_death, 
    event = event_death) ~ tx + age_bl + sex + obstruction_bl + 
    organ_adherence_bl + positive_nodes_bl + differentiation_bl + 
    local_spread_bl_pool + time_surgery_registration_bl, data = colon_cancer_active_pool, 
    robust = TRUE)

  n= 614, number of events= 284 

                                                            coef exp(coef)
tx                                                     -0.317288  0.728121
age_bl                                                  0.004983  1.004995
sex1. Male                                             -0.046771  0.954306
obstruction_bl1. Yes                                    0.343982  1.410553
organ_adherence_bl1. Yes                                0.206154  1.228943
positive_nodes_bl                                       0.079441  1.082682
differentiation_bl2. Moderate                           0.122222  1.130004
differentiation_bl3. Poor                               0.358444  1.431100
local_spread_bl_pool1. Serosa or Contiguous Structures  0.559125  1.749141
time_surgery_registration_bl1. Long                     0.295338  1.343581
                                                        se(coef) robust se
tx                                                      0.121262  0.122432
age_bl                                                  0.005190  0.005459
sex1. Male                                              0.121125  0.123615
obstruction_bl1. Yes                                    0.147532  0.154463
organ_adherence_bl1. Yes                                0.159562  0.170112
positive_nodes_bl                                       0.012153  0.015408
differentiation_bl2. Moderate                           0.210593  0.211963
differentiation_bl3. Poor                               0.247249  0.260995
local_spread_bl_pool1. Serosa or Contiguous Structures  0.215759  0.203238
time_surgery_registration_bl1. Long                     0.133452  0.136939
                                                            z Pr(>|z|)    
tx                                                     -2.592  0.00955 ** 
age_bl                                                  0.913  0.36139    
sex1. Male                                             -0.378  0.70516    
obstruction_bl1. Yes                                    2.227  0.02595 *  
organ_adherence_bl1. Yes                                1.212  0.22556    
positive_nodes_bl                                       5.156 2.52e-07 ***
differentiation_bl2. Moderate                           0.577  0.56420    
differentiation_bl3. Poor                               1.373  0.16964    
local_spread_bl_pool1. Serosa or Contiguous Structures  2.751  0.00594 ** 
time_surgery_registration_bl1. Long                     2.157  0.03103 *  
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

                                                       exp(coef) exp(-coef)
tx                                                        0.7281     1.3734
age_bl                                                    1.0050     0.9950
sex1. Male                                                0.9543     1.0479
obstruction_bl1. Yes                                      1.4106     0.7089
organ_adherence_bl1. Yes                                  1.2289     0.8137
positive_nodes_bl                                         1.0827     0.9236
differentiation_bl2. Moderate                             1.1300     0.8850
differentiation_bl3. Poor                                 1.4311     0.6988
local_spread_bl_pool1. Serosa or Contiguous Structures    1.7491     0.5717
time_surgery_registration_bl1. Long                       1.3436     0.7443
                                                       lower .95 upper .95
tx                                                        0.5728    0.9256
age_bl                                                    0.9943    1.0158
sex1. Male                                                0.7490    1.2159
obstruction_bl1. Yes                                      1.0421    1.9093
organ_adherence_bl1. Yes                                  0.8805    1.7153
positive_nodes_bl                                         1.0505    1.1159
differentiation_bl2. Moderate                             0.7459    1.7120
differentiation_bl3. Poor                                 0.8580    2.3869
local_spread_bl_pool1. Serosa or Contiguous Structures    1.1744    2.6051
time_surgery_registration_bl1. Long                       1.0273    1.7572

Concordance= 0.661  (se = 0.016 )
Likelihood ratio test= 70.15  on 10 df,   p=4e-11
Wald test            = 59.64  on 10 df,   p=4e-09
Score (logrank) test = 85.39  on 10 df,   p=4e-14,   Robust = 69.51  p=6e-11

  (Note: the likelihood ratio and score tests assume independence of
     observations within a cluster, the Wald and robust score tests do not).
```


:::
:::


The proportional hazards assumption should be checked for the covariate adjusted Cox model:


::: {.cell}

```{.r .cell-code}
survival::cox.zph(colon_cancer_cox_adjusted)
```

::: {.cell-output .cell-output-stdout}

```
                               chisq df       p
tx                            0.3026  1 0.58224
age_bl                        0.7435  1 0.38853
sex                           0.1938  1 0.65975
obstruction_bl                7.1944  1 0.00731
organ_adherence_bl            0.5577  1 0.45519
positive_nodes_bl             0.0856  1 0.76989
differentiation_bl           15.2842  2 0.00048
local_spread_bl_pool          6.4447  1 0.01113
time_surgery_registration_bl  1.9779  1 0.15962
GLOBAL                       34.2098 10 0.00017
```


:::
:::





## Adjusted Conditional Cox

While the Cox model provides a conditional hazard ratio, the `speff2trial::speffSurv` function computes a covariate adjusted marginal hazard ratio:


::: {.cell}

```{.r .cell-code}
colon_cancer_speffsurv <-
  speff2trial::speffSurv(
    formula = 
      survival::Surv(time = time_to_death, event = event_death) ~
      age_bl + sex + obstruction_bl + organ_adherence_bl +
      positive_nodes_bl + differentiation_bl + local_spread_bl_pool +
      time_surgery_registration_bl,
    data = colon_cancer_active_pool,
    trt.id = "tx",
    fixed = TRUE,
    
  )

summary(colon_cancer_speffsurv)
```

::: {.cell-output .cell-output-stdout}

```

Treatment effect
            Log HR       SE   LowerCI   UpperCI          p
Prop Haz  -0.34170  0.11986  -0.57661  -0.10678  0.0043600
Speff     -0.34916  0.11283  -0.57030  -0.12802  0.0019708
```


:::
:::





## Precision Gain: HR

Since this provides the estimates and standard errors of both the unadjusted and adjusted marginal hazard ratios, this simplifies computing the relative efficiency of the adjusted analysis:


::: {.cell}

```{.r .cell-code}
colon_cancer_speffsurv_gain <-
  with(
    data = colon_cancer_speffsurv,
    expr = as.numeric(1 - (varbeta["Speff"]/varbeta["Prop Haz"]))
  )

colon_cancer_speffsurv_gain
```

::: {.cell-output .cell-output-stdout}

```
[1] 0.1138518
```


:::
:::





## Unadjusted RMST and Survival

The `adjrct` package computes doubly robust estimates of the RMST and SP, two estimands that are clinically relevant and do not depend on the proportional hazards assumption. The first step involves 


::: {.cell}

```{.r .cell-code}
colon_cancer_meta_unadj <-
  adjrct::survrct(
    outcome.formula = 
      survival::Surv(time = time_to_death, event = event_death) ~ tx,
    trt.formula = tx ~ 1,
    data = colon_cancer_active_pool,
    coarsen = 30 # Time scale: Months vs. Days
  )

# Restricted Mean Survival Time
colon_cancer_rmst_unadj <-
  adjrct::rmst(
    metadata = colon_cancer_meta_unadj,
    # Survival at 1, 3, and 5-years post-randomization
    horizon = round(c(1, 3, 5)*365.25/30)
  )

colon_cancer_rmst_unadj
```

::: {.cell-output .cell-output-stderr}

```
RMST Estimator: tmle
```


:::

::: {.cell-output .cell-output-stdout}

```

           Confidence level: 95%
          Mult. Bootstrap C: 2.276457 

  horizon treatment control theta point-wise 95% CI  uniform 95% CI
1      12     11.65   11.64  0.02   (-0.22 to 0.26) (-0.26 to 0.30)
2      37     32.14   30.89  1.25   (-0.27 to 2.77) (-0.51 to 3.02)
3      61     48.64   44.42  4.22    (1.14 to 7.30)  (0.64 to 7.79)
```


:::

```{.r .cell-code}
# Survival Probability
colon_cancer_sp_unadj <-
  adjrct::survprob(
    metadata = colon_cancer_meta_unadj,
    # Survival at 1, 3, and 5-years post-randomization
    horizon = round(c(1, 3, 5)*365.25/30)
  )

colon_cancer_sp_unadj
```

::: {.cell-output .cell-output-stderr}

```
Survival Probability Estimator: tmle
```


:::

::: {.cell-output .cell-output-stdout}

```

           Confidence level: 95%
          Mult. Bootstrap C: 2.318198 

  horizon treatment control theta point-wise 95% CI  uniform 95% CI
1      12      0.92    0.91  0.01   (-0.03 to 0.06) (-0.04 to 0.06)
2      37      0.74    0.62  0.12    (0.05 to 0.19)  (0.03 to 0.21)
3      61      0.63    0.53  0.10    (0.02 to 0.18)  (0.01 to 0.19)
```


:::
:::





## Helper Functions

A helper function can make it easier to extract and report results with `adjrct::survprob` and `adjrct::rmst`:


::: {.cell}

```{.r .cell-code}
summary.rmst <-
  summary.survprob <-
  function(object, ...){
    if(!inherits(x = object, what = c("rmst", "survprob"))){
      stop("`object` must inherit class \"rmst\" or \"survprob\"")
    }
    estimand <-
      switch(
        EXPR = class(object),
        "rmst" = "RMST:",
        "survprob" = "Survival:"
      )
    time_horizon <- object$horizon
    df_list <- list()
    for(i in 1:length(time_horizon)){
      df_list[[i]] <-
        with(
          data = object$estimates[[i]],
          data.frame(
            Estimand = paste(estimand, c("Arm 1", "Arm 0", "Difference")),
            Estimate = c(arm1, arm0, theta),
            Horizon = time_horizon[i],
            SE = c(arm1.std.error, arm0.std.error, std.error),
            LCL = c(arm1.conf.low, arm0.conf.low, theta.conf.low),
            UCL = c(arm1.conf.high, arm0.conf.high, theta.conf.high)
          )
        )
    }
    return(do.call(what = rbind, args = df_list))
  }
```
:::





## Unadjusted RMST and Survival

Performing a covariate-adjusted analysis involves adding covariates to the outcome and treatment assignment formulas in `adjrct::survrct()`:


::: {.cell}

```{.r .cell-code}
colon_cancer_meta_adj <-
  adjrct::survrct(
    outcome.formula = 
      survival::Surv(time = time_to_death, event = event_death) ~
      age_bl + sex + obstruction_bl + organ_adherence_bl +
      positive_nodes_bl + differentiation_bl + local_spread_bl +
      time_surgery_registration_bl,
    trt.formula = tx ~ 
      age_bl + sex + obstruction_bl + organ_adherence_bl +
      positive_nodes_bl + differentiation_bl + local_spread_bl +
      time_surgery_registration_bl,
    data = colon_cancer_active_pool,
    coarsen = 30
  )
```
:::


The syntax for `adjrct::rmst` and `adjrct::survprob` are identical to the unadjusted analysis:


::: {.cell}

```{.r .cell-code}
colon_cancer_rmst_adj <-
  adjrct::rmst(
    metadata = colon_cancer_meta_adj,
    horizon = round(c(1, 3, 5)*365.25/30)
  )

summary(colon_cancer_rmst_adj)
```

::: {.cell-output .cell-output-stdout}

```
          Estimand    Estimate Horizon         SE        LCL        UCL
1      RMST: Arm 1 11.65758886      12 0.08878838 11.4835668 11.8316109
2      RMST: Arm 0 11.63774100      12 0.08320284 11.4746664 11.8008156
3 RMST: Difference  0.01984785      12 0.12106344 -0.2174321  0.2571278
4      RMST: Arm 1 32.26941493      37 0.52251206 31.2453101 33.2935198
5      RMST: Arm 0 31.06935571      37 0.53972227 30.0115195 32.1271919
6 RMST: Difference  1.20005922      37 0.73384297 -0.2382466  2.6383650
7      RMST: Arm 1 48.83886259      61 1.06249354 46.7564135 50.9213117
8      RMST: Arm 0 44.90332775      61 1.08946481 42.7680160 47.0386395
9 RMST: Difference  3.93553484      61 1.48082330  1.0331745  6.8378952
```


:::

```{.r .cell-code}
colon_cancer_sp_adj <-
  adjrct::survprob(
    metadata = colon_cancer_meta_adj,
    horizon = round(c(1, 3, 5)*365.25/30)
  )
```

::: {.cell-output .cell-output-stderr}

```
Warning: step size truncated due to increasing deviance
Warning: step size truncated due to increasing deviance
Warning: step size truncated due to increasing deviance
Warning: step size truncated due to increasing deviance
Warning: step size truncated due to increasing deviance
Warning: step size truncated due to increasing deviance
Warning: step size truncated due to increasing deviance
```


:::

```{.r .cell-code}
summary(colon_cancer_sp_adj)
```

::: {.cell-output .cell-output-stdout}

```
              Estimand   Estimate Horizon         SE         LCL        UCL
1      Survival: Arm 1 0.92130117      12 0.01560147  0.89072286 0.95187949
2      Survival: Arm 0 0.91050390      12 0.01615225  0.87884607 0.94216173
3 Survival: Difference 0.01079727      12 0.02224657 -0.03280519 0.05439974
4      Survival: Arm 1 0.74933010      37 0.02449032  0.70132995 0.79733025
5      Survival: Arm 0 0.63564082      37 0.02649917  0.58370341 0.68757824
6 Survival: Difference 0.11368928      37 0.03535880  0.04438730 0.18299125
7      Survival: Arm 1 0.63731974      61 0.02732173  0.58377012 0.69086935
8      Survival: Arm 0 0.54223432      61 0.02794764  0.48745796 0.59701068
9 Survival: Difference 0.09508542      61 0.03826418  0.02008901 0.17008183
```


:::
:::



## Precision Gain: SP + RMST

The precision gain from covariate adjustment can be estimated by retrieving the standard errors from results:


::: {.cell}

```{.r .cell-code}
colon_cancer_rmst_gain <-
  1 - (colon_cancer_rmst_adj$estimates[[3]]$std.error/
         colon_cancer_rmst_unadj$estimates[[3]]$std.error)^2

colon_cancer_rmst_gain
```

::: {.cell-output .cell-output-stdout}

```
[1] 0.1123156
```


:::

```{.r .cell-code}
colon_cancer_sp_gain <-
  1 - (colon_cancer_sp_adj$estimates[[3]]$std.error/
         colon_cancer_sp_unadj$estimates[[3]]$std.error)^2

colon_cancer_sp_gain
```

::: {.cell-output .cell-output-stdout}

```
[1] 0.06936247
```


:::
:::


:::
