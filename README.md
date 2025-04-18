# SubgroupBoost

This tutorial guides users through using our package SubgroupBoost to identify treatment performing and non-performing subgroups. The method is based on the following papers
* Zhang P, Ma J, Chen X, Shentu Y. A nonparametric method for value function guided subgroup identification via gradient tree boosting for censored survival data. Stat Med. 2020 Dec 10;39(28):4133-4146.

* Zhang, P., Liu, P., Ma, J., & Shentu, Y. (2021). Value Function Guided Subgroup Identification via Gradient Tree Boosting: A Framework to Handle Multiple Outcomes for Optimal Treatment Recommendation. Statistics in Biopharmaceutical Research, 14(4), 523–531.

## Installation

Install the released version of SubgroupBoost from GitHub with:

``` r
devtools::install_github("liupeng2117/SubgroupBoost")
```

## Data input and preprocessing

```r
library(SubgroupBoost)
```

* Load data. In this tutorial, we will use a simulated data that comes with SubgroupBoost package for demonstration. 

```r
data("simdata")
names(simdata)
str(simdata$train)
str(simdata$test)
```

* The data is a list containing 4 elements: training data, testing data, true subgroup label for training data, and true subgroup label for testing data. The training data contains 600 samples, and testing data contains 5000 samples. The data columns has 26 variables: `trt01p` is treatment indicator. `evnt1` and `aval1` are for the primary outcome overall survival (OS). `evnt1` is the indicator of event (death) for OS, and `aval1` is the time value. `evnt2` and `aval2` are for the secondary outcome time-to-progression (TTP), where `evnt2` is the indicator of event (disease progression), and `aval2` is the time value. Variable `s1` is the predictive variable that interact with treatment, while variables z1 to z20 are not related to treatment effect.

* Preprocess the training and testing data
```r
#----- data manipulation-----#
#training data
dat.train=simdata[[1]][,-c(1:5)]
info.train=simdata[[1]][,1:5]
dtrain <- xgb.DMatrix(as.matrix(dat.train),label = info.train$trt01p)

#testing data
dat.test=simdata[[2]][,-c(1:5)]
info.test=simdata[[2]][,1:5]
dtest <- xgb.DMatrix(as.matrix(dat.test),label = info.test$trt01p)
```

## RMST
* Use RMST based gradient tree boosting method to identify subgroups of patients. Here we choose OS as outcome, as RMST based method can only take one survival outcome.
```r
#----- RMST death -----#
dat1=data.frame(dat.train,trt01p=info.train$trt01p, evnt=info.train$evnt1, aval=info.train$aval1)
set.seed(123)
model_RMST<-SubgroupBoost.RMST(dat1)

#check variables selected
importance=xgb.importance(model_RMST$feature_names, model_RMST)$Feature
importance

# [1] "s1"  "z18" "z20" "z9"  "z6"  "z4"  "z17" "z19" "z16" "z7"  "z13" "z11" "z2"  "z1"  "z12" "z15" "z14" "z3"

# Get predicted value for each sample in training data
pred_RMST<-predict(model_RMST, dtrain)
# Calcualate probabilities of belonging to treatment performing subgroup by sigmoid transformation
prob_RMST<-1/(1+exp(-pred_RMST))
#plot the resulting probabilities
hist(prob_RMST)
```
![Histogram](https://github.com/liupeng2117/SubgroupBoost/raw/master/img/hist.png)

```r
#Wrap it up to be a function, use probability 0.5 as cutoff for subgroup label
SubgroupBoost.predict<-function(model, data){
  # Get predicted value of model
  pred<-predict(model, data)
  # Calcualate probabilities by sigmoid transformation
  prob<-1/(1+exp(-pred))
  # Use 0.5 cutoff to seperate samples into two subgroups
  assign<-ifelse(prob>0.5,1,0)
  return(group_assign=assign)
}

#2 by 2 table of predicted and true subgroup label for training and testing data
RMST.predict.train<-SubgroupBoost.predict(model_RMST, dtrain)
table(Predicted=RMST.predict.train, True=simdata[[3]])

#         True
#Predicted   0   1
#        0 278  76
#        1  12 234
                 
RMST.predict.test<-SubgroupBoost.predict(model_RMST, dtest)
table(Predicted=RMST.predict.test, True=simdata[[4]])

#         True
#Predicted    0    1
#        0 2319  511
#        1  162 2008
```

## Win-difference
* Use win-difference based gradient tree boosting method to identify subgroups of patients. Here we take both OS and TTP as outcomes for win-ratio, OS surives as the primary endpoint and TTP as the secondary outcome, i.e., surrogate of OS when OS is censored. 
```r
#----- win-ratio ----#
#Specify the type of outcome by argument comparison. For two survival outcomes, set it equals "survival survival". 
#Note: Make sure the Primary outcome is defined in variable evnt1 and aval1, and secondary outcome is defined by columns evnt2 and aval2 in info.train object, for two survival outcomes case.
comparison="survival survival"
set.seed(123)
model_wd<-SubgroupBoost.wd(dat, info, comparison)

#check variables selected
importance=xgb.importance(model_wd$feature_names, model_wd)$Feature
importance

# [1] "s1"  "z18" "z20" "z13" "z19"

#2 by 2 table of predicted and true subgroup label for training and testing data
wd.predict.train<-SubgroupBoost.predict(model_wd, dtrain)
table(Predicted=wd.predict.train, True=simdata[[3]])

#         True
#Predicted   0   1
#        0 287  76
#        1   3 234
        
wd.predict.test<-SubgroupBoost.predict(model_wd, dtest)
table(Predicted=wd.predict.test, True=simdata[[4]])

#         True
#Predicted    0    1
#        0 2436  475
#        1   45 2044

```

