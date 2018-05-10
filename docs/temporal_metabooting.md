---
title: "Temporal MetaBooting"
author: "Pablo Montero-Manso"
date: "2018-05-10"
output: rmarkdown::html_vignette
vignette: >
  %\VignetteIndexEntry{Metabooting}
  %\VignetteEngine{knitr::rmarkdown}
  %\VignetteEncoding{UTF-8}
---



This is a example to study the following question:

> If we train a model on a temporal crossvalidated version of the dataset, Can the results be better than just using the temporal crossvalidated errors directly?

#Introduction

For any set of series we want to forecast (including the M4), the true future values are not available, but we need to train the metalearning on something. Previous domain knowledge may be available, but also the last observations of the series in each dataset may be used as training "true"" future values, and the remaining part of each series as the training observed values. I think this is called *temporal crossvalidation*.

When we apply a metalearning method on this new temporal crossvalidation set, errors for each of the forecasting methods in the pool are calculated. We train the metalearning model to mimick the result of the forecast temporal crossvalidation, i.e. *pick the best errors in the temporal crossvalidated version*.
One could argue that the error results on the temporal crossvalidated set can be used directly to pick the best method for predicting the "whole" series. In other words:

> "Pick the forecasting methods that produce the least crossvalidated error instead of training a metalearning approach in this crossvalidated dataset to predict this very result."

Another way:

> "For the series X, the best method for prediciting observations 101:110 in X, trained from observations 1:100, may be also the best to predict observations 111:120, trained from 1:110"

If we train a meta ensemble on the temporal cv subset, it could end up approximating the whole set as well as this "proxy" subset, but not better.

#Experiment and Results

What we do in this example is check these ideas empirically.

1. Create the temporal cv subset of the M3 dataset
2. Process it as if it was the training set for the metaensemble
3. Train the metaensemble on it
4. Compare the performance of the subset and metaensemble on the orginal M3

We start in step 1 creating the temporal crossval dataset and processing it applying the forecasting methods in our pool. 

```r
library(M4metalearning)
#this creates a dataset applying temporal cv to the input dataset
tempcross_M3 <- create_tempcv_dataset(Mcomp::M3)
#process it as if was an other dataset, calculate the forecasts for the methods, errors, etc.
#time consuming step, we will load the results instead
# forec_tempcross_M3 <- process_forecast_dataset(tempcross_M3,
#                                                create_seas_method_list(),
#                                                n.cores = 3)
data(forec_tempcross_M3)
```

Now we have the processed temporal cv dataset, we can train an ensemble method on it.

```r
#extract the features
#another expensive step, load de precalc data
# feat_forec_tempcross_M3 <- generate_THA_feature_dataset(forec_tempcross_M3, n.cores = 2)
data(feat_forec_tempcross_M3)
#refactor the data as a set of matrices
train_data <- create_feat_classif_problem(feat_forec_tempcross_M3)
#traning the mode
tempcross_model <- train_selection_ensemble(train_data$data,
                                            train_data$errors,
                                            train_data$labels)
```

We proceed to compare results, on one hand we have the predictions of the metalearning, and on the other we have the forecasting errors of the temporal cv datset (as approximation of the erros in the whole dataset). We test these two approaches on the original M3 dataset. We begin by calculating the results of the metalearning approach. 


```r
test_data <- M4metalearning::create_feat_classif_problem(M4metalearning::feat_forec_M3)

meta_preds <- predict_selection_ensemble(tempcross_model, test_data$data)

summary_performance(meta_preds, errors = test_data$errors,
                                    labels = test_data$labels,
                                    dataset = M4metalearning::feat_forec_M3)
#> [1] "Classification error:  0.8165"
#> [1] "Selected OWI :  0.8965"
#> [1] "Weighted OWI :  0.8284"
#> [1] "Oracle OWI:  0.5648"
#> [1] "Single method OWI:  0.95"
#> [1] "Average OWI:  1.132"
```

We will now apply a transformation to the errors produced in the temporal crossvalidated dataset to turn them into probabilities of each class: softmax transform but previously inverting the sign of the errors, so higher is better. This will be the predictions of the "temporal cv dataset".

```r
preds <- t(sapply(forec_tempcross_M3, function(seriesdata) {
 exp(-seriesdata$errors) / sum(exp(-seriesdata$errors) )
}))

summary_performance(preds, errors = test_data$errors,
                                    labels = test_data$labels,
                                    dataset = M4metalearning::feat_forec_M3)
#> [1] "Classification error:  0.8192"
#> [1] "Selected OWI :  0.9887"
#> [1] "Weighted OWI :  0.8506"
#> [1] "Oracle OWI:  0.5648"
#> [1] "Single method OWI:  0.95"
#> [1] "Average OWI:  1.132"
```

The results are somehow surprising, **the metalearning approach produces less forecasting error than the actual temporal crossvalidated errors**, ***even tough the metalearning was trained on it!*** This happens for both selection and weighting methods!.

#### Hence the name MetaBooting, because metalearning is pulling itself from its own bootstraps by improving over the dataset it was trained from!