# Invivosyn (R)
## Determine BLISS or HSA synergy from longitudinal, IVIS luminiscent based, in vivo data 

An application example of the R software package invivosyn(1) which can assess the combination index and synergy score using the Bliss independence model as well as the highest single agent (HSA) model. The model does not make any assumption on tumor growth kinetics, study duration, data completeness and balance for tumor volume measurement. The used script is provided. 

(1) Mao, B. & Guo, S. Statistical Assessment of Drug Synergy from In Vivo Combination Studies Using  Mouse Tumor Models. Cancer Res. Commun. 3, 2146–2157 (2023).

#### [Example file (CSV)](https://github.com/bartwesterman/Invivosyn/blob/main/syndata1.csv))
The script as provided by Mao and Guo is provided below:
~~~
title: "my-vignette"
output: rmarkdown::html_vignette
vignette: >
  %\VignetteIndexEntry{my-vignette}
  %\VignetteEngine{knitr::rmarkdown}
  %\VignetteEncoding{UTF-8}
~~~

~~~{r, include = FALSE}
knitr::opts_chunk$set(
  collapse = TRUE,
  comment = "#>",
  message = F,
  warning = F,
  fig.height = 8,
  fig.width = 12,
  dpi = 72,
  cache = FALSE
)
~~~

~~~{r setup}
library(invivoSyn)
library(ggplot2)
library(magrittr)
library(kableExtra)
library(dplyr)
~~~

# Introduction

This vignette demonstrates how to use invivoSyn package, which is designed to evaluate synergy for in vivo tumor growth data. Synergy can be calculated based on TGI/AUC based drug effect or linear mixed model. For effect based efficacy, three reference models can be selected, which are HSA (Highest Single Agent), Bliss(Bliss Independence) or RA(Response Addivity).

# User manual

## Load tumor growth data

Tumor growth data should be based on four arms design: Vehicle, Drug A, Drug B, Drug A+Drug B. The input file should have Treatment, Mouse and days in numeric number, and treatment should be in the order of vehicle,Drug A, Drug B, Drug A+Drug B. Please see below for an example of 4x2 design

| Treatment     | Mouse | 0   | 4   | 7   | ... |
|---------------|-------|-----|-----|-----|-----|
| Vehicle       | 1     | ... | ... | ... | ... |
| Vehicle       | 2     | ... | ... | ... | ... |
| Drug A        | 3     | ... | ... | ... | ... |
| Drug A        | 4     | ... | ... | ... | ... |
| Drug B        | 5     | ... | ... | ... | ... |
| Drug B        | 6     | ... | ... | ... | ... |
| Drug A+Drug B | 7     | ... | ... | ... | ... |
| Drug A+Drug B | 8     | ... | ... | ... | ... |

### Load tumor growth data from a csv file
~~~{r, load_data}
tv <- read_tv(system.file("extdata", "test.csv", package = "invivoSyn")) #you can also load your own csv file formatted as above
summary(tv)
~~~

### Plot tumor growth curve, y can be either TV,RTV,DeltaTV or logTV
~~~{r}
plot_tumor_growth_curve(tv)
plot_tumor_growth_curve(tv, y = "RTV")
~~~

### Compare tumor volume among groups for all time points,y can be either TV,RTV,DeltaTV or logTV
~~~{r, fig.width=16,fig.height=12}
plot_group_by_day(tv)
plot_group_by_day(tv, y = "RTV")
~~~

## Calculate TGI and medianAUCRatio

TGI of a treatment group(T) at specific day(t) is defined as 1-Delta(T)/Delta(V), while Delta(T)=TV^T^(t)-TV^T^(0), and Delta(V)=TV^V^(t)-TV^V^(0)^3^. medianAUCRatio is defined as the median ratio of all normalized AUC pairs for mice in the treatment group and vehicle group^1^. The confidence interval of both TGI and medianAUCRatio are calculated using stratified bootstrap method, three options are available: percentile interval, bootstrap t-interval and bias corrected BCa interval.
~~~{r, efficacy}
sel_day=tv %>% filter(Group=='Group 1') %>% count(Day) %>% mutate(prop=n/max(n)) %>% filter(prop>=0.8) %>% pull(Day) %>% max()
TGI_lst <- getTGI(tv, sel_day, ci = 0.9, ci_type = "bca")
TGI_lst$bsTGI_df %>% kable(digits=3)
AUC_lst <- get_mAUCr(tv, ci = 0.9, ci_type = "bca")
AUC_lst$bsAUC_df %>% kable(digits=3)
~~~

## Calculate synergy 

### TGI based synergy
TGI based synergy is based on TGI from a specific day, normally we will use the last day of the vehicle group. Synergy score is defined as the difference between observed and expected TGI  of the combination group in percentage, using Bliss, HSA or RA as the reference model. P-value and confidence interval are calculated using stratified bootstrap method, three options are available: percentile interval, bootstrap t-interval and bias corrected BCa interval. 
~~~{r, fig.width=16}
bliss_synergy_TGI <- TGI_synergy(TGI_lst)
bliss_synergy_TGI
hsa_synergy_TGI <- TGI_synergy(TGI_lst, method = "HSA", ci = 0.9, ci_type = "bca", save = T)
hsa_synergy_TGI
~~~

### AUC based synergy
AUC based synergy is based on area under curve(AUC) of each mouse, first we will estimate tumor growth rate(k) of each mouse by its normalized AUC^1^, which can be used to calculate tumor volume at time t using formula TV~t~=TV~0~exp(kt). We will calcuate probability of survival associated with drug effect by conditional probability Pr(T|C)=S~T~/S~C~, with S~T~ defined as survival of treatment group and S~C~ defined as survival of control group, a.k.a. vehicle group^2^. TV~t~ is equalvalent with S~T~ here. We will compare observed survival and expected survival of combination group, using Bliss, HSA or RA as the reference model. For Bliss model, S~AB~=S~A~S~B~; for HSA model, S~AB~=min(S~A~,S~B~); and for RA model,S~AB~=S~A~+S~B~-1. Combination index is defined as the ratio of observed and expected S~AB~, whicle synergy score is defined as the difference between observed and expected S~AB~ in percentage. P-value and confidence interval are calculated using stratified bootstrap method, three options are available: percentile interval, bootstrap t-interval and bias corrected BCa interval. 
~~~{r,fig.width=16}
bliss_synergy_AUC <- AUC_synergy(AUC_lst)
bliss_synergy_AUC %>% kable(digits=3)
hsa_synergy_AUC <- AUC_synergy(AUC_lst, t = 21, method = "HSA", ci_type = "bca", ci = 0.9, save = T)
hsa_synergy_AUC %>% kable(digits=3)
~~~

### Linear mixed model based synergy
Synergy calculation is based on linear mixed model, using log transformed TV as the response variable. The coefficient and p-value of the interaction term is the readout of synergy calculation.
~~~{r}
lmm_synergy(tv) %>% kable(digits=3)
~~~

## Simulate tumor growth data
We will generate simulated tumor growth data using exponential growth model, based on the number of animals in each group(n), drug effect (tgi_a,tgi_b and tgi_ab), variation of tumor volumes (sd0) and variation of tumor growth rate among mice(sigma_r), time to double at vehicle group (vehicle_double_time) and length of study (t).
~~~{r,simu}
tvs <- simu_TV(n = 5, tv0 = 50, vehicle_double_time = 5, t = 21, sigma_r = 0.1, tgi_a = 0.5, tgi_b = 0.4, tgi_c = 0.8)
plot_tumor_growth_curve(tvs, "RTV")
~~~

## Power calculation
Power calculation are based on simulated dataset. First we will generate 100 simulated dataset, then we will calculate p-value using aforementioned methods. Power is defined as the proportion of p.value < 0.05 in all 100 simulated datasets.
~~~{r eval=FALSE}
p <- power_calc(method = "Bliss", type = "AUC", n = 5, tv0 = 50, vehicle_double_time = 5, t = 21, sigma_r = 0.1, tgi_a = 0.5, tgi_b = 0.4, tgi_c = 0.8)
power_stats <- sim_power(
  type = c("TGI", "AUC", "lmm"),
  method = c("HSA", "Bliss"),
  n = 2:10,
  tgi_c = c(0.7, 0.75, 0.8)
) ## This will take a long time at a 16 core laptop
~~~

~~~{r}
ggplot(power_sim) +
  aes(n, Power, color = type, group = type) +
  geom_point() +
  geom_line() +
  facet_grid(method ~ tgi_c) +
  theme_Publication() +
  scale_colour_Publication()
~~~

# Reference
1. **Guo S, Jiang X, Mao B, et al.** The design, analysis and application of mouse clinical trials in oncology drug development[J]. *BMC cancer*, 2019, 19(1): 1-14.
2. **Demidenko E, Miller T W.** Statistical determination of synergy based on Bliss definition of drugs independence[J]. *PLoS One*, 2019, 14(11): e0224137.
3. **Huang L, Wang J, Fang B, et al.** CombPDX: a unified statistical framework for evaluating drug synergism in patient-derived xenografts[J]. *Scientific reports*, 2022, 12(1): 1-10.
