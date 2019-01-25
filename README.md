# R package conformal.glm 

## Conformal Prediction for Generalized Linear Regression Models

This package computes and compares prediction regions for the normal, Gamma, 
and inverse Gaussian families in the `glm` package.  There is functionality to 
construct the parametric conformal prediction region and the nonparametric 
conformal prediction region. 


## Usage 

```r
library(devtools)
install_github(repo = "DEck13/conformal.glm", subdir="conformal.glm")
```

## Illustrative Example 

We provide a gamma regression example with perfect model specification to 
illustrate the performance of conformal predictions when the model is known 
and the model does not have additive symmetric errors.  We also compare 
conformal prediction regions to the oracle highest density region under 
the correct model. This example is included in the corresponding paper:  

  Eck, D.J., Crawford, F.W., and Aronow, P.M. (2018+)
  Conformal prediction for exponential families and generalized linear models.
  Preprint available on request (email daniel.eck@yale.edu).

```r
alpha <- 0.10
n <- 500
shape <- 2
beta <- c(1/4, 2)

set.seed(13)
x <- matrix(runif(n), ncol = 1)
rate <- cbind(1, x) %*% beta * shape
y <- rgamma(n = n, shape = shape, rate = rate)
data.readme <- data.frame(y = y, x = x)
colnames(data.readme)[2] <- c("x1")

fit.readme = glm(y ~ x1, family = Gamma, data = data.readme)
summary(fit.readme)
```


Compute the parametric and nonparametric conformal prediction regions.
```r
bins <- 3
system.time(cpred <- conformal.glm(fit.readme, 
  nonparametric = TRUE, bins = bins, cores = 6))
paraCI <- cpred$paraconformal
nonparaCI <- cpred$nonparaconformal
```


Compute the least squares (LS) conformal prediction region with local 
weighting using the conformalInference package.
```r
## least squares conformal prediction region
library(conformalInference)
funs <- lm.funs(intercept = TRUE)
train.fun <- funs$train.fun
predict.fun <- funs$predict.fun

cubic.model <- lm(y ~ x + I(x^2) + I(x^3))
abs.resid <- abs(cubic.model$resid)
smooth.call <- smooth.spline(x, abs.resid, 
  nknots = 10)
lambda <- smooth.call$lambda
df <- smooth.call$df
mad.train.fun <- function(x, y, out = NULL){
  smooth.spline(x[, 1], y, lambda = lambda, 
    df = df, nknots = 10)
}
mad.predict.fun <- function(out, newx){
  predict(out, newx[, 1])$y
}
system.time(p1.tibs <- conformal.pred(x = cbind(x,x^2,x^3), 
  y = y, x0 = cbind(x,x^2,x^3), 
  train.fun = train.fun, predict.fun = predict.fun, 
  mad.train.fun = mad.train.fun,
  mad.predict.fun = mad.predict.fun,
  alpha = alpha))
LSLW = cbind(p1.tibs$lo, p1.tibs$up)
```


Compute the highest density (HD) prediction region.
```r
library(HDInterval)
betaMLE <- coefficients(fit.readme)
shapeMLE <- as.numeric(gamma.shape(fit.readme)[1])
rateMLE <- cbind(1, x) %*% betaMLE * shapeMLE
HDCI <- do.call(rbind, 
  lapply(1:nrow(x), function(j){ 
    hdi(qgamma, 0.90, shape = shapeMLE, rate = rateMLE[j, 1])
  }))
```


## Plot of all prediction regions

The four prediction regions for this data are depicted below.  The top row 
depicts the parametric conformal prediction region (left panel) and the 
nonparametric conformal prediction region (right panel).  The bin width was 
specified as 1/3 for these conformal prediction regions.  The bottom row 
depicts the least squares conformal prediction region (left panel) and the 
highest density region (right panel).  We see that the parametric conformal 
prediction region is a close discretization of the HD region, the 
nonparametric conformal prediction region is quite jagged and unnatural, and 
the LS conformal prediction region includes negative response 
values and systematically over (under) includes small (large) values of the 
response across x. 

```r
#########################################
## make plot
par(mfrow = c(2,2), oma = c(4,4,0,0), mar = c(1,1,1,1))

# parametric conformal prediction region
ix <- sort(x, index.return = TRUE)$ix
plot.new()
plot.window(xlim = c(0,1), ylim = c(min(y), max(y)))
points(x, y, pch = 19, col = "gray")
lines(x[ix], paraCI[ix, 1], type = "l", col = "red")
lines(x[ix], paraCI[ix, 2], type = "l", col = "red")
axis(2)

# nonparametric conformal prediction region
plot.nonparametric <- function(region){
  if(class(region) != "list"){ 
    stop("Only appropriate for nonparametric conformal prediction region")
  }
  plot.new()
  plot.window(xlim = c(0,1), ylim = c(min(y), max(y)))
  points(x, y, pch = 19, col = "gray")
  for(i in 1:bins){ 
    nonpar.i  <- nonparaCI[[i]]
    odd <- which(1:length(nonpar.i) %% 2 == 1) 
    even <- which(1:length(nonpar.i) %% 2 == 0)
    segments(x0 = 1/bins * (i-1), y0 = nonpar.i, x1 =1/bins * i, 
      col = "red")
    if(i != 1){ 
      nonpar.i.previous <- nonparaCI[[i-1]]
      nonpar <- sort(c(nonpar.i, nonpar.i.previous))
      odd2 <- which(1:length(nonpar) %% 2 == 1) 
      even2 <- which(1:length(nonpar) %% 2 == 0)
      segments(x0 = 1/bins * (i-1), y0 = nonpar[odd2], 
        y1 = nonpar[even2], col = "red")
    }
  }
}    
plot.nonparametric(nonparaCI)

# least squares conformal prediction region
plot.new()
plot.window(xlim = c(0,1), ylim = c(0,max(y)))
points(x, y, pch = 19, col = "gray")
lines(x[ix], LSLW[ix, 1], type = "l", col = "red")
lines(x[ix], LSLW[ix, 2], type = "l", col = "red")
axis(1); axis(2)

# highest density region
plot.new()
plot.window(xlim = c(0,1), ylim = c(0,max(y)))
points(x, y, pch = 19, col = "gray")
lines(x[ix], HDCI[ix, 1], type = "l", col = "red")
lines(x[ix], HDCI[ix, 2], type = "l", col = "red")
axis(1)

# axis labels
mtext("x", side = 1, line = 2.5, outer = TRUE, cex = 2)
mtext("y", side = 2, line = 2.5, outer = TRUE, cex = 2)
```


![Plot of prediction regions](gammasimexample.png)


## Coverage properties and estimated area of all prediction regions

All of the presented prediction regions exhibit close to finite-sample 
marginal validity and local validity with respect to bining.  However, 
the LS conformal prediction region and the HD prediction region do not exhibit 
finite-sample local validity in the second bin and the HD prediction region 
does not quite possess finite-sample marginal validity.  The parametric 
conformal prediction region is smallest in size with an estimated area of 
2.20.  The HD prediction region is a close second with an estimated area of 
2.21.  LS conformal prediction region has an estimated area of 2.57 and 
The nonparametric conformal prediction region has an estimated area of 2.69.  
Under correct model specification, the parametric conformal prediction 
region is similar in performance to that of the highest density prediction 
region.


```r
#########################################
## area and coverage

## parametric conformal prediction region
# estimated area
mean(apply(paraCI, 1, diff))
# local coverage
p <- length(beta) - 1
local.coverage(region = paraCI, data = data.readme, d = p, 
  bins = bins, at.data = "TRUE")
# marginal coverage
local.coverage(region = paraCI, data = data.readme, d = p, 
  bins = 1, at.data = "TRUE")

## nonparametric conformal prediction region
# estimated area
area.nonparametric <- function(region){
  if(class(region) != "list"){ 
    stop("Only appropriate for nonparametric conformal prediction region")
  }
  bins <- length(region); wn <- 1 / bins
  area <- 0
  for(i in 1:bins){
    nonpar.region <- region[[i]]
    area <- area + wn * as.numeric(rep(c(-1,1), 
      length(nonpar.region)/2) %*% nonpar.region)
  }
  area
}
area.nonparametric(nonparaCI)
# local coverage
local.coverage(region = nonparaCI, data = data.readme, d = p, 
  nonparametric = "TRUE", bins = bins, at.data = "TRUE")
# marginal coverage
local.coverage(region = nonparaCI, data = data.readme, d = p, 
  nonparametric = "TRUE", bins = 1, at.data = "TRUE")

## LSLW conformal prediction region
# estimated area
mean(apply(LSLW, 1, diff))
# local coverage
local.coverage(region = LSLW, data = data.readme, d = p, 
  bins = bins, at.data = "TRUE")
# marginal coverage
local.coverage(region = LSLW, data = data.readme, d = p, 
  bins = 1, at.data = "TRUE")

## HD region
# estimated area
mean(apply(HDCI, 1, diff))
# local coverage
local.coverage(region = HDCI, data = data.readme, d = p, 
  bins = bins, at.data = "TRUE")
# marginal coverage
local.coverage(region = HDCI, data = data.readme, d = p, 
  bins = 1, at.data = "TRUE")
```

To cite this package:
```r
citation("conformal.glm")
```


## Further details

For more details on the parametric conformal prediction region, see:

  Eck, D.J., Crawford, F.W., and Aronow, P.M. (2018+)
  Conformal prediction for exponential families and generalized linear models.
  Preprint available on request (email daniel.eck@yale.edu).

For more details on the prediciton region formed from conformalization of 
residuals, see:

  Lei, J., G'Sell, M., Rinaldo, A., Tibshirani, R., and Wasserman, L. (2016)
  Distribution-Free Predictive Inference for Regression. 
  https://arxiv.org/abs/1604.04173

For more details on the nonparametric conformal prediction region, see:

  Lei, J. and Wasserman, L. (2014)
  Distribution-Free Prediction Bands for Non-parametric Regression. 
  Journal of the Royal Statistical Society: Series B, 76(1), 71-96.

