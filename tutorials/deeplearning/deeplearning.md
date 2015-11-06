# Classification and Regression with H2O Deep Learning

- Introduction
  - Installation and Startup
  - Decision Boundaries
- Cover Type Dataset
  - Exploratory Data Analysis
  - Deep Learning Model
  - Hyper-Parameter Search
  - Checkpointing
  - Cross-Validation
  - Model Save & Load
- Regression and Binary Classification
- Deep Learning Tips & Tricks

## Introduction
This tutorial shows how a [Deep Learning](http://en.wikipedia.org/wiki/Deep_learning) model can be used to do supervised classification and regression. This file is available in plain R, R markdown and regular markdown formats. More examples and explanations can be found in our [H2O Deep Learning booklet](http://h2o.ai/resources/) and on our [Github Repository](http://github.com/h2oai/h2o-3/).

First, set the path to the directory in which the tutorial is located on the server that runs H2O (here, locally):

```r
ROOT_PATH <- "/users/arno/h2o-world-2015-training/tutorials/"
```

### H2O R Package

Load the H2O R package:

```r
## R installation instructions are at http://h2o.ai/download
library(h2o)
```

### Start H2O
Start up a 1-node H2O server on your local machine, and allow it to use all CPU cores and up to 2GB of memory:

```r
h2o.init(nthreads=-1, max_mem_size="2G")
h2o.removeAll() ## clean slate - just in case the cluster was already running
```

The `h2o.deeplearning` function fits H2O's Deep Learning models from within R.
We can run the example from the man page using the `example` function, or run a longer demonstration from the `h2o` package using the `demo` function:

```{r help}
args(h2o.deeplearning)
help(h2o.deeplearning)
example(h2o.deeplearning)
#demo(h2o.deeplearning)
```

While H2O Deep Learning has many parameters, it was designed to be just as easy to use as the other supervised training methods in H2O. Early stopping, automatic data standardization and handling of categorical variables and missing values and adaptive learning rates (per weight) reduce the amount of parameters the user has to specify. Often, it's just the number and sizes of hidden layers, the number of epochs and the activation function and maybe some regularization techniques.

### Let's have some fun first: Decision Boundaries
To get familiar with the nature of Deep Learning (DL) in comparison to H2O's tree methods (GBM/DRF) and linear modeling (GLM), we plot contour plots of the probability density to see how they separate two spirals in 2D:

```r
dev.new(noRStudioGD=FALSE) ##direct plotting output to a new window
spiral <- h2o.importFile(paste0(ROOT_PATH, "/data/spiral.csv"))
grid   <- h2o.importFile(paste0(ROOT_PATH, "/data/grid.csv"))
# Define helper to plot contours
plotC <- function(name, model, data=spiral, g=grid) {
  data <- as.data.frame(data) #get data from into R
  pred <- as.data.frame(h2o.predict(model, g))
  n=0.5*(sqrt(nrow(g))-1); d <- 1.5; h <- d*(-n:n)/n
  plot(data[,-3],pch=19,col=data[,3],cex=1.0,
       xlim=c(-d,d),ylim=c(-d,d),main=name)
  contour(h,h,z=array(ifelse(pred[,1]=="Red",0,1),
          dim=c(2*n+1,2*n+1)),col="blue",lwd=2,add=T)
}
par(mfrow=c(2,2)) # set up the canvas for 2x2 plots
plotC( "DL", h2o.deeplearning(1:2,3,spiral,epochs=1e3))
plotC("GBM", h2o.gbm         (1:2,3,spiral))
plotC("DRF", h2o.randomForest(1:2,3,spiral))
plotC("GLM", h2o.glm         (1:2,3,spiral,family="binomial"))
```  

Let's investigate some more Deep Learning models:
```r
for (epochs in c(1,200,500,1000)) {
  plotC( paste0("DL ",epochs," epochs"), h2o.deeplearning(1:2,3,spiral,epochs=epochs))
}
```

```r
for (activation in c("Tanh", "Maxout", "Rectifier", "RectifierWithDropout")) {
  plotC( paste0("DL ",activation," activation"), h2o.deeplearning(1:2,3,spiral,epochs=1000))
}
```

## Cover Type Dataset
We important the full cover type dataset (581k rows, 13 columns, 10 numerical, 3 categorical).
We also split the data 3 ways: 60% for training, 20% for validation (hyper parameter tuning) and 20% for final testing.

```r
par(mfrow=c(1,1)) # reset canvas
df <- h2o.importFile(paste0(ROOT_PATH, "/data/covtype.full.csv"))
dim(df)
df
splits <- h2o.splitFrame(df, c(0.6,0.2), seed=1234)
train  <- h2o.assign(splits[[1]], "train.hex")
valid  <- h2o.assign(splits[[2]], "valid.hex")
test   <- h2o.assign(splits[[3]], "test.hex")
```

Here's a scalable way to do scatter plots via binning (works for categorical and numeric columns) to get more familiar with the dataset.

```r
plot(h2o.tabulate(df, "Elevation",                       "Cover_Type"))
plot(h2o.tabulate(df, "Horizontal_Distance_To_Roadways", "Cover_Type"))
plot(h2o.tabulate(df, "Soil_Type",                       "Cover_Type"))
plot(h2o.tabulate(df, "Horizontal_Distance_To_Roadways", "Elevation" ))
```

### First Run of H2O Deep Learning
Let's run our first Deep Learning model on the covtype dataset. 
We want to predict the `Cover_Type` column, a categorical feature with 7 levels, and the Deep Learning model will be tasked to perform (multi-class) classification. It uses the other 12 predictors of the dataset, of which 10 are numerical, and 2 are categorical with a total of 44 levels. We can expect the Deep Learning model to have 56 input neurons (after automatic one-hot encoding).

To keep it fast, we only run for one epoch (one pass over the training data).
```r
response <- "Cover_Type"
predictors <- setdiff(names(df), response)
predictors
```

```r
m1 <- h2o.deeplearning(
  model_id="dl_model_first", 
  training_frame=train, 
  validation_frame=valid,   ## validation dataset: used for scoring and early stopping
  x=predictors,
  y=response,
  #activation="Rectifier",  ## default
  #hidden=c(200,200),       ## default: 2 hidden layers with 200 neurons each
  epochs=1,
  variable_importances=T    ## not enabled by default
)
summary(m1)
```

### Variable Importances
Variable importances for Neural Network models are notoriously difficult to compute, and there are many [pitfalls](ftp://ftp.sas.com/pub/neural/importance.html). H2O Deep Learning has implemented the method of [Gedeon](http://cs.anu.edu.au/~./Tom.Gedeon/pdfs/ContribDataMinv2.pdf), and returns relative variable importances in descending order of importance.

```r
head(as.data.frame(h2o.varimp(m1)))
```

### Early Stopping
Now we run another, smaller network, and we let it stop automatically once the misclassification rate converges. We also sample the validation set to 10,000 rows for faster scoring.

```r
m2 <- h2o.deeplearning(
  model_id="dl_model_faster", 
  training_frame=train, 
  validation_frame=valid,
  x=predictors,
  y=response,
  hidden=c(32,32,32),                  ## small network, runs faster
  epochs=1000000,
  score_validation_samples=10000,      ## sample the valiation dataset, faster and accurate enough
  stopping_rounds=1,
  stopping_metric="misclassification", ## could be "MSE","logloss","r2"
  stopping_tolerance=0.01
)
summary(m2)
plot(m2)
```

### Adaptive Learning Rate
By default, H2O Deep Learning uses an adaptive learning rate ([ADADELTA](http://arxiv.org/pdf/1212.5701v1.pdf)) for its stochastic gradient descent optimization. There are only two tuning parameters for this method: `rho` and `epsilon`, which balance the global and local search efficiencies. `rho` is the similarity to prior weight updates (similar to momentum), and `epsilon` is a parameter that prevents the optimization to get stuck in local optima. Defaults are `rho=0.99` and `epsilon=1e-8`. For cases where convergence speed is very important, it might make sense to perform a few runs to optimize these two parameters (e.g., with `rho=c(0.9,0.95,0.99,0.999)` and `epsilon=c(1e-10,1e-8,1e-6,1e-4)`). Of course, as always with grid searches, caution has to be applied when extrapolating grid search results to a different parameter regime (e.g., for more epochs or different layer topologies or activation functions, etc.).

If `adaptive_rate` is disabled, several manual learning rate parameters become important: `rate`, `rate_annealing`, `rate_decay`, `momentum_start`, `momentum_ramp`, `momentum_stable` and `nesterov_accelerated_gradient`, the discussion of which we leave to [H2O Deep Learning booklet](http://h2o.ai/resources/).

### Tuning
With some tuning, it is possible to obtain less than 10% test set error rate in about one minute. Error rates of below 5% are possible with larger models. Deep tree methods are more effective for this dataset than Deep Learning, as the space needs to be simply be partitioned into the corresponding hyper-space corners to solve this problem.

```r
m3 <- h2o.deeplearning(
  model_id="dl_model_tuned", 
  training_frame=train, 
  validation_frame=valid, 
  x=predictors, 
  y=response, 
  overwrite_with_best_model=F,
  hidden=c(128,128,128),          ## more hidden layers -> more complex interactions
  epochs=10,                      ## to keep it short enough
  score_validation_samples=10000, ## downsample validation set for faster scoring
  score_duty_cycle=0.025,         ## don't score more than 2.5% of the wall time
  adaptive_rate=F,                ## manually tuned learning rate
  rate=0.02, 
  rate_annealing=2e-6,            
  momentum_start=0.2,             ## manually tuned momentum
  momentum_stable=0.4, 
  momentum_ramp=1e7, 
  l1=1e-5,                        ## add some L1/L2 regularization
  l2=1e-5,
  max_w2=10                       ## helps stability for Rectifier
) 
summary(m3)
```

Let's compare the training error with the validation and test set errors

```r
h2o.performance(m3, train=T)       ## sampled training data (from model building)
h2o.performance(m3, valid=T)       ## sampled validation data (from model building)
h2o.performance(m3, data=train)    ## full training data
h2o.performance(m3, data=valid)    ## full validation data
h2o.performance(m3, data=test)     ## full test data
```

To confirm that the reported confusion matrix on the validation set (here, the test set) was correct, we make a prediction on the test set and compare the confusion matrices explicitly:

```r
pred <- h2o.predict(m3, test)
pred
test$Accuracy <- pred$predict == test$Cover_Type
1-mean(test$Accuracy)
```
    
### Hyper-parameter Tuning with Grid Search
Since there are a lot of parameters that can impact model accuracy, hyper-parameter tuning is especially important for Deep Learning:

For speed, we will only train on the first 10,000 rows of the training dataset:

```r
sampled_train=train[1:10000,]
```
  
The simplest hyperparameter search method is a brute-force scan of the full Cartesian product of all combinations specified by a grid search:

```r
hyper_params <- list(
  hidden=list(c(32,32,32),c(64,64)),
  input_dropout_ratio=c(0,0.05),
  rate=c(0.01,0.02),
  rate_annealing=c(1e-8,1e-7,1e-6)
)
hyper_params
grid <- h2o.grid(
  "deeplearning",
  model_id="dl_grid", 
  training_frame=sampled_train,
  validation_frame=valid, 
  x=predictors, 
  y=response,
  epochs=10,
  stopping_metric="misclassification",
  stopping_tolerance=1e-2,        ## stop when logloss does not improve by >=1% for 2 scoring events
  stopping_rounds=2,
  score_validation_samples=10000, ## downsample validation set for faster scoring
  score_duty_cycle=0.025,         ## don't score more than 2.5% of the wall time
  adaptive_rate=F,                ## manually tuned learning rate
  momentum_start=0.5,             ## manually tuned momentum
  momentum_stable=0.9, 
  momentum_ramp=1e7, 
  l1=1e-5,
  l2=1e-5,
  activation=c("Rectifier"),
  max_w2=10,                      ## can help improve stability for Rectifier
  hyper_params=hyper_params
)
grid
```
                                
Let's see which model had the lowest validation error:

```r
## Find the best model and its full set of parameters (clunky for now, will be improved)
scores <- cbind(as.data.frame(unlist((lapply(grid@model_ids, function(x) 
  { h2o.confusionMatrix(h2o.performance(h2o.getModel(x),valid=T))$Error[8] })) )), unlist(grid@model_ids))
names(scores) <- c("misclassification","model")
sorted_scores <- scores[order(scores$misclassification),]
head(sorted_scores)
best_model <- h2o.getModel(as.character(sorted_scores$model[1]))
print(best_model@allparameters)
best_err <- sorted_scores$misclassification[1]
print(best_err)
```
    
### Random Hyper-Parameter Search
Often, hyper-parameter search for more than 4 parameters can be done more efficiently with random parameter search than with grid search. Basically, chances are good to find one of many good models in less time than performing an exhaustive grid search. We simply build `N` models with parameters drawn randomly from user-specified distributions (here, uniform). For this example, we use the adaptive learning rate and focus on tuning the network architecture and the regularization parameters.

```r
models <- c()
for (i in 1:10) {
  rand_activation <- c("TanhWithDropout", "RectifierWithDropout")[sample(1:2,1)]
  rand_numlayers <- sample(2:5,1)
  rand_hidden <- c(sample(10:50,rand_numlayers,T))
  rand_l1 <- runif(1, 0, 1e-3)
  rand_l2 <- runif(1, 0, 1e-3)
  rand_dropout <- c(runif(rand_numlayers, 0, 0.6))
  rand_input_dropout <- runif(1, 0, 0.5)
  dlmodel <- h2o.deeplearning(
    model_id=paste0("dl_random_model_", i),
    training_frame=sampled_train,
    validation_frame=valid, 
    x=predictors, 
    y=response,
#    epochs=100,                    ## for real parameters: set high enough to get to convergence
    epochs=1,
    stopping_metric="misclassification",
    stopping_tolerance=1e-2,        ## stop when logloss does not improve by >=1% for 2 scoring events
    stopping_rounds=2,
    score_validation_samples=10000, ## downsample validation set for faster scoring
    score_duty_cycle=0.025,         ## don't score more than 2.5% of the wall time
    max_w2=10,                      ## can help improve stability for Rectifier

    ### Random parameters
    activation=rand_activation, 
    hidden=rand_hidden, 
    l1=rand_l1, 
    l2=rand_l2,
    input_dropout_ratio=rand_input_dropout, 
    hidden_dropout_ratios=rand_dropout
  )                                
  models <- c(models, dlmodel)
}
```
  
We continue to look for the model with the lowest validation misclassification rate:

```r
best_err <- 1      ##start with the best reference model from the grid search above, if available
for (i in 1:length(models)) {
  err <- h2o.confusionMatrix(h2o.performance(models[[i]],valid=T))$Error[8]
  if (err < best_err) {
    best_err <- err
    best_model <- models[[i]]
  }
}
h2o.confusionMatrix(best_model,valid=T)
best_params <- best_model@allparameters
best_params$hidden
best_params$l1
best_params$l2
best_params$input_dropout_ratio
```            
    
###Checkpointing
Let's continue training the manually tuned model from before, for 2 more epochs. Note that since many important parameters such as `epochs, l1, l2, max_w2, score_interval, train_samples_per_iteration, input_dropout_ratio, hidden_dropout_ratios, score_duty_cycle, classification_stop, regression_stop, variable_importances, force_load_balance` can be modified between checkpoint restarts, it is best to specify as many parameters as possible explicitly.

```r
max_epochs <- 12 ## Add two more epochs
m_cont <- h2o.deeplearning(
  model_id="dl_model_tuned_continued", 
  checkpoint="dl_model_tuned", 
  training_frame=train, 
  validation_frame=valid, 
  x=predictors, 
  y=response, 
  hidden=c(128,128,128),          ## more hidden layers -> more complex interactions
  epochs=max_epochs,              ## hopefully long enough to converge (otherwise restart again)
  stopping_metric="logloss",      ## logloss is directly optimized by Deep Learning
  stopping_tolerance=1e-2,        ## stop when validation logloss does not improve by >=1% for 2 scoring events
  stopping_rounds=2,
  score_validation_samples=10000, ## downsample validation set for faster scoring
  score_duty_cycle=0.025,         ## don't score more than 2.5% of the wall time
  adaptive_rate=F,                ## manually tuned learning rate
  rate=0.02, 
  rate_annealing=2e-6,            
  momentum_start=0.2,           ## manually tuned momentum
  momentum_stable=0.4, 
  momentum_ramp=1e7, 
  l1=1e-5,                        ## add some L1/L2 regularization
  l2=1e-5,
  max_w2=10                     ## helps stability for Rectifier
) 
summary(m_cont)
plot(m_cont)
```

Once we are satisfied with the results, we can save the model to disk (on the cluster).
In this example, the cluster is running on the same file system as the client, so are fine.
```r
path <- h2o.saveModel(m_cont, path=paste0(ROOT_PATH,"mybest_deeplearning_covtype_model"), force=T)
```

It can be loaded later with the following command:
```r
print(path)
m_loaded <- h2o.loadModel(path)
summary(m_loaded)
```
This model can then be used for a checkpoint restart, or to score a dataset, etc.

###Cross-Validation
For N-fold cross-validation, specify nfolds instead of a validation frame, and `N+1` models will be built: 1 model on the full training data, and N models with each 1/N-th of the data held out (there are different holdout strategies). Those N models then score on the held out data, and their combined predictions on the full training data are scored to get the cross-validation metrics.
    
```r
dlmodel <- h2o.deeplearning(
  x=predictors,
  y=response, 
  training_frame=train,
  hidden=c(10,10),
  epochs=0.1,
  nfolds=5)
summary(dlmodel)
```

N-fold cross-validation is especially useful with early stopping, as the main model will pick the ideal number of epochs from the convergence behavior of the nfolds cross-validation models.

##Regression and Binary Classification
Assume we want to turn the multi-class problem above into a binary classification problem. We create a binary response as follows:

```r
train$bin_response <- ifelse(train[,response]=="class_1", 0, 1)
```

Let's build a quick model and inspect the model:

```r
dlmodel <- h2o.deeplearning(
  x=predictors,
  y="bin_response", 
  training_frame=train,
  hidden=c(10,10),
  epochs=0.1
)
summary(dlmodel)
```

We find a regression model (`H2ORegressionModel`) that contains only 1 output neuron, instead of 2. The reason is that the response was a numerical feature (numbers 0 and 1), and H2O Deep Learning was run with `distribution=AUTO`, which defaulted to a Gaussian regression problem for a real-valued response.

To perform classification, the response must first be turned into a categorical (factor):

```r
train$bin_response <- as.factor(train$bin_response) ## Turn into categorical levels "0"/"1"
dlmodel <- h2o.deeplearning(
  x=predictors,
  y="bin_response", 
  training_frame=train,
  hidden=c(10,10),
  epochs=0.1
  #balance_classes=T    ## enable this for high class imbalance
)
summary(dlmodel) ## Now the model metrics contain AUC for binary classification
plot(h2o.performance(dlmodel)) ## display ROC curve
```

Now the model performs (binary) classification, and has multiple (2) output neurons.

##H2O Deep Learning Tips & Tricks
####Activation Functions
While sigmoids have been used historically for neural networks, H2O Deep Learning implements `Tanh`, a scaled and shifted variant of the sigmoid which is symmetric around 0. Since its output values are bounded by -1..1, the stability of the neural network is rarely endangered. However, the derivative of the tanh function is always non-zero and back-propagation (training) of the weights is more computationally expensive than for rectified linear units, or `Rectifier`, which is `max(0,x)` and has vanishing gradient for `x<=0`, leading to much faster training speed for large networks and is often the fastest path to accuracy on larger problems. In case you encounter instabilities with the `Rectifier` (in which case model building is automatically aborted), try a limited value to re-scale the weights: `max_w2=10`. The `Maxout` activation function is least computationally effective, but can lead to higher accuracy. It is a generalized version of the Rectifier with two channels: `max(Ax+b, Cx+d)`.

####Generalization Techniques
L1 and L2 penalties can be applied by specifying the `l1` and `l2` parameters. Intuition: L1 lets only strong weights survive (constant pulling force towards zero), while L2 prevents any single weight from getting too big. [Dropout](http://arxiv.org/pdf/1207.0580.pdf) has recently been introduced as a powerful generalization technique, and is available as a parameter per layer, including the input layer. `input_dropout_ratio` controls the amount of input layer neurons that are randomly dropped (set to zero), while `hidden_dropout_ratios` are specified for each hidden layer. The former controls overfitting with respect to the input data (useful for high-dimensional noisy data such as MNIST), while the latter controls overfitting of the learned features. Note that `hidden_dropout_ratios` require the activation function to end with `...WithDropout`.

####Early stopping and optimizing for lowest validation error
By default, `overwrite_with_best_model` is set to TRUE and the model returned after training for the specified number of epochs is the model that has the best training set error, or, if a validation set is provided, the lowest validation set error. This is equivalent to early stopping, except that the determination to stop is made in hindsight, similar to a full grid search over the number of epochs at the granularity of the scoring intervals. Note that for N-fold cross-validation, `overwrite_with_best_model` is disabled to give fair results (all N cross-validation models must run to completion to avoid overfitting on the validation set). Also note that the training or validation set errors can be based on a subset of the training or validation data, depending on the values for `score_validation_samples` or `score_training_samples`, see below. For actual early stopping on a predefined error rate on the *training data* (accuracy for classification or MSE for regression), specify `classification_stop` or `regression_stop`.

####Training Samples per (MapReduce) Iteration
This parameter is explained in the [H2O Deep Learning booklet](http://h2o.ai/resources/), and becomes important in multi-node operation. It controls the number of rows trained on for each MapReduce iteration. Cluster nodes then communicate via the network to agree on the best neural net model coefficients (weights/biases) between iterations, and have the opportunity to perform scoring (controlled by other parameters below). The default value of `-2` indicates auto-tuning, which attemps to keep the communication overhead at 5% of the total runtime. The parameter `target_ratio_comm_to_comp` controls this ratio.

####Categorical Data
For categorical data, a feature with K factor levels is automatically one-hot encoded (horizontalized) into K-1 input neurons. Hence, the input neuron layer can grow substantially for datasets with high factor counts. In these cases, it might make sense to reduce the number of hidden neurons in the first hidden layer, such that large numbers of factor levels can be handled. In the limit of 1 neuron in the first hidden layer, the resulting model is similar to logistic regression with stochastic gradient descent, except that for classification problems, there's still a softmax output layer, and that the activation function is not necessarily a sigmoid (`Tanh`). If variable importances are computed, it is recommended to turn on `use_all_factor_levels` (K input neurons for K levels). The experimental option `max_categorical_features` uses feature hashing to reduce the number of input neurons via the hash trick at the expense of hash collisions and reduced accuracy.

####Missing Values
H2O Deep Learning automatically does mean imputation for missing values during training (leaving the input layer activation at 0 after standardizing the values). For testing, missing test set values are also treated the same way by default. See the `h2o.impute` function to do your own mean imputation.

####Reproducibility
Every run of DeepLearning results in different results since multithreading is done via [Hogwild!](http://www.eecs.berkeley.edu/~brecht/papers/hogwildTR.pdf) that benefits from intentional lock-free race conditions between threads. To get reproducible results for small datasets and testing purposes, set reproducible=T and set seed=<any integer>. This will not work for big data for technical reasons, and is probably also not desired because of the significant slowdown.
    
####Scoring on Training/Validation Sets During Training  
The training and/or validation set errors *can* be based on a subset of the training or validation data, depending on the values for `score_validation_samples` (defaults to 0: all) or `score_training_samples` (defaults to 10,000 rows, since the training error is only used for early stopping and monitoring). For large datasets, Deep Learning can automatically sample the validation set to avoid spending too much time in scoring during training, especially since scoring results are not currently displayed in the model returned to R.
                                
Note that the default value of `score_duty_cycle=0.1` limits the amount of time spent in scoring to 10%, so a large number of scoring samples won't slow down overall training progress too much, but it will always score once after the first MapReduce iteration, and once at the end of training.

Stratified sampling of the validation dataset can help with scoring on datasets with class imbalance.  Note that this option also requires `balance_classes` to be enabled (used to over/under-sample the training dataset, based on the max. relative size of the resulting training dataset, `max_after_balance_size`):
    
### More information can be found in the [H2O Deep Learning booklet](http://h2o.ai/resources/) and in our [presentations](http://www.slideshare.net/0xdata/presentations).