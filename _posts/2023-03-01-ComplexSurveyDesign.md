---
layout: post
title: "Handle Complex Survey Data in R"
categories: SurveyDesign
excerpt: Understanding how to handle complex survey design is crucial for researchers and analysts who work with survey data, particularly those who want to produce accurate and representative estimates of population parameters. Properly accounting for concepts such as primary survey units, clusters, strata, and sampling weights, as well as using specialized software packages designed for complex survey data analysis, is essential for producing reliable results. This article will teach you how to use R to handle complex survey design in 10 minutes. 
---

## Introduction

Complex survey design is a method of collecting data from a sample of individuals in a population using a sampling scheme that is more intricate than simple random sampling. It involves stratifying the population into subgroups, called strata, and selecting a sample from each stratum using various sampling techniques such as cluster sampling, multistage sampling, and stratified sampling. The goal of complex survey design is to produce estimates of population parameters that are both accurate and representative of the entire population.

In a complex survey design, there are four main concepts that are important to understand: primary survey unit, cluster, stratum, and sampling weights.

<b>Primary Survey Unit</b>: The primary survey unit (PSU) is the smallest unit in the sample design for which information is collected. It is also known as the sampling unit or primary sampling unit. The PSU can be an individual, household, or group of individuals, depending on the research question and sampling design. The selection of PSUs is the first step in the sampling process, and it is important that they are selected randomly to ensure that the sample is representative of the population.

<b>Cluster</b>: A cluster is a group of primary survey units that are selected together in the sample. Clustering is a common sampling method used in complex survey designs, where primary survey units are grouped together into clusters and then a random sample of clusters is selected. Clustering is often used when it is difficult or expensive to sample primary survey units individually, or when it is more efficient to sample them in groups.

<b>Stratum</b>: A stratum is a subgroup of the population that shares similar characteristics or traits. Stratification is the process of dividing the population into different strata based on these characteristics. Strata can be based on demographic variables, geographic location, or other relevant factors. Stratification helps to ensure that the sample is representative of the population and that estimates are accurate for each stratum. When sampling, it is important to select a sample from each stratum to ensure that the estimates are unbiased.

<b>Sampling weights</b>: Sampling weights are necessary in complex survey designs because the probability of selection of each unit or cluster may not be the same, and may be influenced by various factors such as stratification and clustering. Therefore, weighting is used to adjust for these differences and ensure that the sample accurately represents the population.

Overall, complex survey design is an important method for collecting data from a sample of individuals in a population. Understanding the primary survey unit, cluster, and stratum is critical in designing and analyzing complex surveys, as they play a key role in determining the accuracy and representativeness of the sample.

Here we use the R package ```survey``` to achieve the best possible unbiased estimates from a complex survey design.

## R Code

### 1. Load pacakges


```R
packages = c('foreign',  #foreign is used to load a spss file
             'survey') #survey is the package for complex survey design

for (package in packages){
    require(package, character.only = T)
}
```


### 2. Load data 
Here we used the National Epidemiologic Survey on Alcohol and Related Conditions(NESARC) Wave I as an example. The NESARC is a substance-use-focused dataset based on a nationally representative sample 
<br>
<br>
To maximize the representativeness of the data with a limited budget, the NESARC Wave I used a <b> multistage stratified design </b> where primary sampling units (PSUs) were stratified according to certain sociodemographic criteria. You can find methodological details in <a href="https://pubs.niaaa.nih.gov/publications/NESARC_DRM2/NESARC2DRM.pdf ">this document</a>.
<br>
<br>
If you want to try it out, you can download the data <a href="https://catalog.data.gov/dataset/national-epidemiologic-survey-on-alcohol-and-related-conditions-nesarcwave-1-20012002-and-">here</a>.





```R
data = read.spss('../Data/NESARC_merged_Nov2010.sav', to.data.frame = T)
```

    re-encoding from CP1252
    



### 3. Create a subset data of interest

To speed up the calculation, I created a subset of data only containing the required information from the survey design (i.e., PSU, stratum, and sampling weights) and interested research variables. Note that the NESARC I does not contain a cluster variable.


```R
new_data = data.frame(psu = data$psu, #PSU
                      stratum = data$stratum, #Stratum
                      weight = data$weight, #Sampling weight 
                      alc = data$CONSUMER, #Alcohol consumption status 
                      sex = data$sex, #Sex
                      alc_daily_volume = data$etotlcu) #Averaged daily alcohol consumption volume

#Specifiying variable type as factor
new_data$alc = as.factor(new_data$alc) 
new_data$sex = as.factor(new_data$sex)

#Overview of the new data set
str(new_data)
```

    'data.frame':	43093 obs. of  6 variables:
     $ psu             : num  4007 6045 12042 17099 17099 ...
     $ stratum         : num  403 604 1218 1704 1704 ...
     $ weight          : num  3929 3639 5779 1072 4987 ...
     $ alc             : Factor w/ 3 levels "1","2","3": 3 1 3 2 2 1 1 1 1 1 ...
     $ sex             : Factor w/ 2 levels "1","2": 1 2 2 1 1 2 1 2 2 2 ...
     $ alc_daily_volume: num  NA 0.0014 NA NA NA ...



### 4. Specify the complex survey design with the ```svydesign()``` function


```R
dstrat = svydesign(id=~psu,strata=~stratum, weights=~weight, data=new_data)
```

If the survey was ideally designed, you could use ```svymean()``` to obtain the estimated mean and SE of each variable. However, the NESARC research group faced some tough real-world situations and needed to compromise to save money, leading to a few strata only having one primary sampling unit....

You can try run the following command. You'll receive an error message of "Stratum (103) has only one PSU at stag"


```R
svymean(~alc, dstrat) 
```


    Error in onestrat(x[index, , drop = FALSE], clusters[index], nPSU[index][1], : Stratum (103) has only one PSU at stage 1
    Traceback:


    1. svymean(~alc, dstrat)

    2. svymean.survey.design2(~alc, dstrat)

    3. svyrecvar(x * pweights/psum, design$cluster, design$strata, design$fpc, 
     .     postStrata = design$postStrata)

    4. multistage(x, clusters, stratas, fpcs$sampsize, fpcs$popsize, 
     .     lonely.psu = getOption("survey.lonely.psu"), one.stage = one.stage, 
     .     stage = 1, cal = cal)

    5. onestage(x, stratas[, 1], clusters[, 1], nPSUs[, 1], fpcs[, 1], 
     .     lonely.psu = lonely.psu, stage = stage, cal = cal)

    6. tapply(1:NROW(x), list(factor(strata)), function(index) {
     .     onestrat(x[index, , drop = FALSE], clusters[index], nPSU[index][1], 
     .         fpc[index], lonely.psu = lonely.psu, stratum = strata[index][1], 
     .         stage = stage, cal = cal)
     . })

    7. lapply(X = ans[index], FUN = FUN, ...)

    8. FUN(X[[i]], ...)

    9. onestrat(x[index, , drop = FALSE], clusters[index], nPSU[index][1], 
     .     fpc[index], lonely.psu = lonely.psu, stratum = strata[index][1], 
     .     stage = stage, cal = cal)

    10. stop("Stratum (", stratum, ") has only one PSU at stage ", stage)



### 5. Convert the survey design to use replicate weights 

This step is required because some strata had only one primary sampling unit, so we need to produce resampled weights to bypass this problem. 


There are many ways to derive replicate weights, such as the Rau & Wu method we showed below, the jackknife method, balanced repeated replication (BRR), Fay's modification of BRR, Canty and Davison's bootstrap, etc. You can check the function description ```as.svrepdesign()``` in the [survey package documentation](https://cran.r-project.org/web/packages/survey/survey.pdf). 


```R
#Rau & Wu rescaled bootstrap
dstrat1 = as.svrepdesign(dstrat, type="subbootstrap", replicates = 10) 
#It's recommended to set replicates = 100. Just saving calculation time here.
```

The reason for choosing the Rau and Wu method is because it had the closest estimates to the original published in the "main findings from the 2001-2202 NESARC." They used a proprietary software called SUDAAN for estimation. Although we have no idea how SUDAAN estimates the variables, we at least can replicate the results with transparent methods.

Note that a lot of NESARC wave 1 and 2 documents were archived and disappeared from the internet now. The abovementioned document can no longer be found online. To prove that I did not make up this document, here is a [paper](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6527256/) that also mentioned this document. The table I provided below was retrieved from an email I sent to my grad school advisor yeeeeears ago.


The estimates provided by the NESARC team:

|                  |    All       ||       Male   ||     Female   ||
|------------------|--------|------|--------|------|--------|------|
|                  |Estimate|  SE  |Estimate|  SE  |Estimate|  SE  |
|Current drinker   |0.6544  |0.0059|0.7182  |0.0059|0.5957  |0.0077|
|Former drinker    |0.1728  |0.0036|0.1661  |0.0043|0.1791  |0.0043|
|Lifetime abstainer|0.1728  |0.0062|0.1157  |0.05  |0.2253  |0.0086|


<br>
Let's compare the table above to our estimated values below. 



```R
#Inspect the estimated mean and se of alcohol consumption status of the entire sample 
#(alc1= current consumer; alc2=ex-drinker; alc3=life-time abstainer)
svymean(~alc, dstrat1) 

```


            mean     SE
    alc1 0.65439 0.0041
    alc2 0.17285 0.0033
    alc3 0.17276 0.0054



```R
#Inspect the estimated mean and se of alcohol consumption grouped by sexes
#Result by Sex (1 = male; 2 = female; alc1 = Current drinker; alc2 = Former drinker; alc3 = Lifetime abstainer)
boot_rep = svyby(~alc, ~sex, dstrat1, svymean) 
boot_rep
```


<table class="dataframe">
<caption>A svyby: 2 × 7</caption>
<thead>
	<tr><th></th><th scope=col>sex</th><th scope=col>alc1</th><th scope=col>alc2</th><th scope=col>alc3</th><th scope=col>se1</th><th scope=col>se2</th><th scope=col>se3</th></tr>
	<tr><th></th><th scope=col>&lt;fct&gt;</th><th scope=col>&lt;dbl&gt;</th><th scope=col>&lt;dbl&gt;</th><th scope=col>&lt;dbl&gt;</th><th scope=col>&lt;dbl&gt;</th><th scope=col>&lt;dbl&gt;</th><th scope=col>&lt;dbl&gt;</th></tr>
</thead>
<tbody>
	<tr><th scope=row>1</th><td>1</td><td>0.7182303</td><td>0.1660978</td><td>0.1156718</td><td>0.006652246</td><td>0.004821198</td><td>0.005084681</td></tr>
	<tr><th scope=row>2</th><td>2</td><td>0.5956542</td><td>0.1790601</td><td>0.2252857</td><td>0.005991821</td><td>0.002531499</td><td>0.007215825</td></tr>
</tbody>
</table>



You'll need to rearrange this table a bit to compare to the NESARC table above. I trust you can do that.

## Prediction models

The survey package supports generalized linear models (GLM) and ANOVA with complex survey design. Here is a demo of running a linear regression model with GLM.
The Gaussian family model in GLM resembles the common linear regression based on ordinary least squares (OLS). However, GLM uses iteratively reweighted least squares (IWLS) instead of OLS to estimate regression coefficients. There should be minimal differences between Gaussian GLM and the commonly used linear regression function ```lm()``` with the OLS method in simple cases. It is just that the OLS method cannot take complex survey designs into consideration at least for now.

If you want to run multivariate models, there is a package called [lavaan.survey](https://cran.r-project.org/web/packages/lavaan.survey/lavaan.survey.pdf).


```R
#Predict alcohol daily consumption values by sex with glm 

alc_daily_volume_sex = 
    svyglm(alc_daily_volume~sex, #specify the model
           design = dstrat1, #specify survey design
           na.action = na.omit, #specify how to handle missing data; here we omit any missing data
           family = 'gaussian') #specify GLM family as Gaussian

summary(alc_daily_volume_sex)
```


    
    Call:
    svyglm(formula = alc_daily_volume ~ sex, design = dstrat1, na.action = na.omit, 
        family = "gaussian")
    
    Survey design:
    as.svrepdesign(dstrat, type = "subbootstrap", replicates = 10)
    
    Coefficients:
                Estimate Std. Error t value Pr(>|t|)    
    (Intercept)  0.66342    0.01394   47.58 4.21e-11 ***
    sex2        -0.38501    0.01795  -21.45 2.35e-08 ***
    ---
    Signif. codes:  0 ‘***’ 0.001 ‘**’ 0.01 ‘*’ 0.05 ‘.’ 0.1 ‘ ’ 1
    
    (Dispersion parameter for gaussian family taken to be 41012.28)
    
    Number of Fisher Scoring iterations: 2



You can see that sex significantly associated with daily drinking volume. The results showed that female (sex2) on average drank .38 fl oz less than males. 

### Pseudo-<i>R<sup>2</sup></i>

Although the ```glm``` function is more flexible than ```lm``` and can be used for logistic regrssion models or other specified distributions, one downside of ```glm``` is that we cannot obtain <i>R<sup>2</sup></i> directly. Below is how we can compute pseudo-<i>R<sup>2</sup></i> following the formula: R2 = 1 - (Residual Sum of Squares/Total Sum of Squares)

```R
#Compute pseudo R2: (1 - RSS/TSS)
1 - sum(alc_daily_volume_sex$residuals^2)/(var(na.omit(new_data)$alc_daily_volume)*nrow(na.omit(new_data)))

0.025605872614839
```





Of course, the difference in sex only explained a small amount (2.6%) of variance in the daily volume of alcohol consumption. 


Although researchers love the concept of <i>R<sup>2</sup></i> and that is why I provided the code here, to my knowledge, there is no established method to compute pseudo-<i>R<sup>2</sup></i> while incorporating the complex survey design.
