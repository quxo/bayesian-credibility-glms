<h1>Bayesian Credibility</h1>

    
<p> This page contains the R code used for generating the applied
      example of the article Bayesian Credibility. The published
      version can be found <a
      href="https://doi.org/10.1016/j.insmatheco.2018.05.001">in this
      link</a> and a preprint <a href="https://arxiv.org/pdf/1710.08553.pdf">here </a> It is first necessary to install the
      package <code>rstan</code>. If you have not installed it run
      first the command <code> install.packages("rstan") </code>
      before running what follows.  </p>
    
<hr>
<h3>1. Downloading and preparing the data </h3>
    
<p>
      The following code downloads the data and modifies the variables
      as indicated in the article. Note that the code above downloads
      the file <code>car.csv</code> to your working directory.
</p>

<pre>
download.file("http://www.businessandeconomics.mq.edu.au/our_departments/Applied_Finance_and_Actuarial_Studies/acst_docs/glms_for_insurance_data/data/car.csv",destfile="car.csv")

cars.data <- read.csv("car.csv",header=TRUE)
cars.data$veh_body <- factor(cars.data$veh_body)
cars.data$veh_age <- factor(cars.data$veh_age)
cars.data$gender <- factor(cars.data$gender)
cars.data$area <- factor(cars.data$area)
cars.data$agecat <- factor(cars.data$agecat)

cars.data$veh_value <- cut(cars.data$veh_value,breaks=c(0,1.2,1.86,Inf),labels=c("P1","P2","P3"),include.lowest=T)

 ## Put together area A,B,C,D
cars.data$area <- factor(cars.data$area,levels=c(levels(cars.data$area),"ABCD") )
cars.data$area[cars.data$area %in% c("A","B","C","D")] <- "ABCD"
cars.data$area <- factor(cars.data$area)
cars.data$area <- relevel(cars.data$area,"ABCD")
</pre>

<hr>

<h3> 2. Using Stan</h3>    

<p>
      In order to use Stan we first load the library and detect the
      cores of the computer.
</p>

<pre>
library(rstan)

rstan_options(auto_write = TRUE)
options(mc.cores = parallel::detectCores())
</pre>
    
Now we create a string with the stan code of the model and store in a variable.

<pre>
stan.severity.code="
data {
  int<lower=1> M; // Number of rows of the design matrix
  int<lower=1> P; // Number of columns of the design matrix
  matrix[M,P] X; // Design matrix
  vector<lower=0>[M] y; // response variable
  vector<lower=0>[M] ws; // durations
}

parameters {
  vector<lower=-20,upper=20>[P] beta;
  real<lower=0,upper=1000> phi;
}

transformed parameters{
  vector[M] mu;
  mu = exp(X * beta);
}

model{
  beta ~ uniform(-20,20);
  phi ~ uniform(0,1000);
  for (n in 1:M){
     y[n] ~ gamma(ws[n]/phi,ws[n]/(phi*mu[n]));
 }
}"

</pre>

    
<p>
      Now we aggregate the data, get the corresponding design matrix
      create a list with the data for Stan, and fit the stan model.
</p>

<pre >
agg.data <- aggregate(cbind(claimcst0,numclaims) ~  agecat + gender + area + veh_value,
                        subset=clm==1,
                        data=cars.data,
                        FUN=sum)

X <- model.matrix( ~ agecat + gender + area + veh_value,
                  data=agg.data)


model.data <- list(M=dim(X)[1],
                       P=dim(X)[2],
                       y=agg.data$claimcst0/agg.data$numclaims,
                       X=X,
                       ws=agg.data$numclaims)

sev.stanfit <- stan(model_code=stan.severity.code,
                    data=model.data,
                    iter=30000,
                    warmup=2000,
                    chains=3)
</pre>

<hr>
<h3>3. Entropic Estimates</h3>

<p>
The following computes the posterior mean, entropic beta
and entropic mean respectively.
</p>

<pre>
## Posterior mean
post.mu <- colMeans(extract(sev.stanfit,pars="mu" )$mu)

## Entropic betas
entropic.beta <- coef(glm(post.mu ~ agecat + gender + area + veh_value,
                              family=Gamma(link="log"),
                              weights=numclaims,
                          data=agg.data ))

## Entropic credibility
		 entropic.mu <-exp( X %*% entropic.beta)
</pre>

<hr>
<h3>4. Non Credibility Estimates</h3>

<p>
Now we compute the premium using frequentist GLMs.
</p>

<pre>
agg.data$obs.premium <- agg.data$claimcst0 / agg.data$numclaims

non.cred.model <- glm(obs.premium ~ agecat + gender + area + veh_value,
                      family=Gamma(link="log"),
                      weights=numclaims,
                      data=agg.data )

non.cred.premium <- predict(non.cred.model ,type="response")
</pre>
				    

				    
<hr>
<h3> 4. Classes table</h3>


<table border=1>
<tr> <th> agecat </th> <th> gender </th> <th> area </th> <th> veh_value </th> <th> claimcst0 </th> <th> numclaims </th> <th> Credibility Premium </th> <th> Premium </th>  </tr>
  <tr> <td> 5 </td> <td> F </td> <td> ABCD </td> <td> P3 </td> <td align="right"> 143286.00 </td> <td align="right"> 103.00 </td> <td align="right"> 1373.24 </td> <td align="right"> 1362.23 </td> </tr>
  <tr> <td> 5 </td> <td> F </td> <td> ABCD </td> <td> P2 </td> <td align="right"> 190286.65 </td> <td align="right"> 99.00 </td> <td align="right"> 1428.23 </td> <td align="right"> 1416.31 </td> </tr>
  <tr> <td> 6 </td> <td> F </td> <td> ABCD </td> <td> P3 </td> <td align="right"> 49604.06 </td> <td align="right"> 33.00 </td> <td align="right"> 1475.97 </td> <td align="right"> 1458.01 </td> </tr>
  <tr> <td> 3 </td> <td> F </td> <td> ABCD </td> <td> P3 </td> <td align="right"> 342153.16 </td> <td align="right"> 210.00 </td> <td align="right"> 1517.42 </td> <td align="right"> 1508.93 </td> </tr>
  <tr> <td> 4 </td> <td> F </td> <td> ABCD </td> <td> P3 </td> <td align="right"> 175800.70 </td> <td align="right"> 157.00 </td> <td align="right"> 1520.57 </td> <td align="right"> 1512.69 </td> </tr>
  <tr> <td> 6 </td> <td> F </td> <td> ABCD </td> <td> P2 </td> <td align="right"> 90709.05 </td> <td align="right"> 74.00 </td> <td align="right"> 1535.08 </td> <td align="right"> 1515.90 </td> </tr>
  <tr> <td> 3 </td> <td> F </td> <td> ABCD </td> <td> P2 </td> <td align="right"> 300949.94 </td> <td align="right"> 202.00 </td> <td align="right"> 1578.18 </td> <td align="right"> 1568.84 </td> </tr>
  <tr> <td> 4 </td> <td> F </td> <td> ABCD </td> <td> P2 </td> <td align="right"> 427309.04 </td> <td align="right"> 226.00 </td> <td align="right"> 1581.46 </td> <td align="right"> 1572.75 </td> </tr>
  <tr> <td> 5 </td> <td> F </td> <td> E </td> <td> P3 </td> <td align="right"> 14694.00 </td> <td align="right"> 16.00 </td> <td align="right"> 1598.67 </td> <td align="right"> 1572.53 </td> </tr>
  <tr> <td> 5 </td> <td> F </td> <td> ABCD </td> <td> P1 </td> <td align="right"> 166165.91 </td> <td align="right"> 95.00 </td> <td align="right"> 1604.94 </td> <td align="right"> 1590.78 </td> </tr>
  <tr> <td> 5 </td> <td> M </td> <td> ABCD </td> <td> P3 </td> <td align="right"> 107384.24 </td> <td align="right"> 112.00 </td> <td align="right"> 1649.01 </td> <td align="right"> 1635.28 </td> </tr>
  <tr> <td> 5 </td> <td> F </td> <td> E </td> <td> P2 </td> <td align="right"> 5494.55 </td> <td align="right"> 9.00 </td> <td align="right"> 1662.68 </td> <td align="right"> 1634.97 </td> </tr>
  <tr> <td> 2 </td> <td> F </td> <td> ABCD </td> <td> P3 </td> <td align="right"> 260872.70 </td> <td align="right"> 151.00 </td> <td align="right"> 1670.48 </td> <td align="right"> 1660.47 </td> </tr>
  <tr> <td> 5 </td> <td> M </td> <td> ABCD </td> <td> P2 </td> <td align="right"> 93941.60 </td> <td align="right"> 76.00 </td> <td align="right"> 1715.05 </td> <td align="right"> 1700.21 </td> </tr>
  <tr> <td> 6 </td> <td> F </td> <td> E </td> <td> P3 </td> <td align="right"> 1985.14 </td> <td align="right"> 3.00 </td> <td align="right"> 1718.27 </td> <td align="right"> 1683.11 </td> </tr>
  <tr> <td> 6 </td> <td> F </td> <td> ABCD </td> <td> P1 </td> <td align="right"> 91259.62 </td> <td align="right"> 67.00 </td> <td align="right"> 1725.01 </td> <td align="right"> 1702.64 </td> </tr>
  <tr> <td> 2 </td> <td> F </td> <td> ABCD </td> <td> P2 </td> <td align="right"> 243282.20 </td> <td align="right"> 164.00 </td> <td align="right"> 1737.38 </td> <td align="right"> 1726.39 </td> </tr>
  <tr> <td> 3 </td> <td> F </td> <td> E </td> <td> P3 </td> <td align="right"> 47338.40 </td> <td align="right"> 25.00 </td> <td align="right"> 1766.52 </td> <td align="right"> 1741.88 </td> </tr>
  <tr> <td> 4 </td> <td> F </td> <td> E </td> <td> P3 </td> <td align="right"> 28446.99 </td> <td align="right"> 20.00 </td> <td align="right"> 1770.19 </td> <td align="right"> 1746.22 </td> </tr>
  <tr> <td> 6 </td> <td> M </td> <td> ABCD </td> <td> P3 </td> <td align="right"> 150576.26 </td> <td align="right"> 71.00 </td> <td align="right"> 1772.38 </td> <td align="right"> 1750.27 </td> </tr>
  <tr> <td> 3 </td> <td> F </td> <td> ABCD </td> <td> P1 </td> <td align="right"> 416263.74 </td> <td align="right"> 204.00 </td> <td align="right"> 1773.45 </td> <td align="right"> 1762.09 </td> </tr>
  <tr> <td> 4 </td> <td> F </td> <td> ABCD </td> <td> P1 </td> <td align="right"> 382979.87 </td> <td align="right"> 221.00 </td> <td align="right"> 1777.14 </td> <td align="right"> 1766.48 </td> </tr>
  <tr> <td> 6 </td> <td> F </td> <td> E </td> <td> P2 </td> <td align="right"> 33129.19 </td> <td align="right"> 7.00 </td> <td align="right"> 1787.07 </td> <td align="right"> 1749.93 </td> </tr>
  <tr> <td> 3 </td> <td> M </td> <td> ABCD </td> <td> P3 </td> <td align="right"> 271928.40 </td> <td align="right"> 162.00 </td> <td align="right"> 1822.15 </td> <td align="right"> 1811.38 </td> </tr>
  <tr> <td> 4 </td> <td> M </td> <td> ABCD </td> <td> P3 </td> <td align="right"> 268648.71 </td> <td align="right"> 165.00 </td> <td align="right"> 1825.94 </td> <td align="right"> 1815.90 </td> </tr>
  <tr> <td> 3 </td> <td> F </td> <td> E </td> <td> P2 </td> <td align="right"> 43056.17 </td> <td align="right"> 25.00 </td> <td align="right"> 1837.26 </td> <td align="right"> 1811.04 </td> </tr>
  <tr> <td> 4 </td> <td> F </td> <td> E </td> <td> P2 </td> <td align="right"> 28805.81 </td> <td align="right"> 18.00 </td> <td align="right"> 1841.08 </td> <td align="right"> 1815.55 </td> </tr>
  <tr> <td> 6 </td> <td> M </td> <td> ABCD </td> <td> P2 </td> <td align="right"> 108135.99 </td> <td align="right"> 63.00 </td> <td align="right"> 1843.35 </td> <td align="right"> 1819.76 </td> </tr>
  <tr> <td> 5 </td> <td> F </td> <td> E </td> <td> P1 </td> <td align="right"> 14554.23 </td> <td align="right"> 8.00 </td> <td align="right"> 1868.41 </td> <td align="right"> 1836.37 </td> </tr>
  <tr> <td> 3 </td> <td> M </td> <td> ABCD </td> <td> P2 </td> <td align="right"> 198017.50 </td> <td align="right"> 112.00 </td> <td align="right"> 1895.11 </td> <td align="right"> 1883.30 </td> </tr>
  <tr> <td> 4 </td> <td> M </td> <td> ABCD </td> <td> P2 </td> <td align="right"> 260961.26 </td> <td align="right"> 131.00 </td> <td align="right"> 1899.06 </td> <td align="right"> 1888.00 </td> </tr>
  <tr> <td> 5 </td> <td> M </td> <td> E </td> <td> P3 </td> <td align="right"> 39418.93 </td> <td align="right"> 15.00 </td> <td align="right"> 1919.71 </td> <td align="right"> 1887.74 </td> </tr>
  <tr> <td> 5 </td> <td> M </td> <td> ABCD </td> <td> P1 </td> <td align="right"> 184712.55 </td> <td align="right"> 75.00 </td> <td align="right"> 1927.25 </td> <td align="right"> 1909.65 </td> </tr>
  <tr> <td> 2 </td> <td> F </td> <td> E </td> <td> P3 </td> <td align="right"> 18717.61 </td> <td align="right"> 13.00 </td> <td align="right"> 1944.71 </td> <td align="right"> 1916.81 </td> </tr>
  <tr> <td> 2 </td> <td> F </td> <td> ABCD </td> <td> P1 </td> <td align="right"> 341669.44 </td> <td align="right"> 166.00 </td> <td align="right"> 1952.34 </td> <td align="right"> 1939.06 </td> </tr>
  <tr> <td> 5 </td> <td> M </td> <td> E </td> <td> P2 </td> <td align="right"> 8078.37 </td> <td align="right"> 7.00 </td> <td align="right"> 1996.59 </td> <td align="right"> 1962.69 </td> </tr>
  <tr> <td> 5 </td> <td> F </td> <td> F </td> <td> P3 </td> <td align="right"> 16626.50 </td> <td align="right"> 7.00 </td> <td align="right"> 2002.05 </td> <td align="right"> 1962.81 </td> </tr>
  <tr> <td> 2 </td> <td> M </td> <td> ABCD </td> <td> P3 </td> <td align="right"> 415682.61 </td> <td align="right"> 142.00 </td> <td align="right"> 2005.95 </td> <td align="right"> 1993.30 </td> </tr>
  <tr> <td> 6 </td> <td> F </td> <td> E </td> <td> P1 </td> <td align="right"> 9685.72 </td> <td align="right"> 6.00 </td> <td align="right"> 2008.19 </td> <td align="right"> 1965.50 </td> </tr>
  <tr> <td> 2 </td> <td> F </td> <td> E </td> <td> P2 </td> <td align="right"> 27660.32 </td> <td align="right"> 17.00 </td> <td align="right"> 2022.59 </td> <td align="right"> 1992.92 </td> </tr>
  <tr> <td> 1 </td> <td> F </td> <td> ABCD </td> <td> P3 </td> <td align="right"> 127635.32 </td> <td align="right"> 55.00 </td> <td align="right"> 2054.96 </td> <td align="right"> 2033.95 </td> </tr>
  <tr> <td> 6 </td> <td> M </td> <td> E </td> <td> P3 </td> <td align="right"> 988.43 </td> <td align="right"> 2.00 </td> <td align="right"> 2063.33 </td> <td align="right"> 2020.48 </td> </tr>
  <tr> <td> 3 </td> <td> F </td> <td> E </td> <td> P1 </td> <td align="right"> 31886.61 </td> <td align="right"> 16.00 </td> <td align="right"> 2064.58 </td> <td align="right"> 2034.13 </td> </tr>
  <tr> <td> 4 </td> <td> F </td> <td> E </td> <td> P1 </td> <td align="right"> 46814.15 </td> <td align="right"> 19.00 </td> <td align="right"> 2068.87 </td> <td align="right"> 2039.20 </td> </tr>
  <tr> <td> 6 </td> <td> M </td> <td> ABCD </td> <td> P1 </td> <td align="right"> 117751.87 </td> <td align="right"> 48.00 </td> <td align="right"> 2071.43 </td> <td align="right"> 2043.93 </td> </tr>
  <tr> <td> 5 </td> <td> F </td> <td> F </td> <td> P2 </td> <td align="right"> 1210.95 </td> <td align="right"> 1.00 </td> <td align="right"> 2082.22 </td> <td align="right"> 2040.74 </td> </tr>
  <tr> <td> 2 </td> <td> M </td> <td> ABCD </td> <td> P2 </td> <td align="right"> 216822.03 </td> <td align="right"> 114.00 </td> <td align="right"> 2086.28 </td> <td align="right"> 2072.44 </td> </tr>
  <tr> <td> 3 </td> <td> M </td> <td> E </td> <td> P3 </td> <td align="right"> 56767.03 </td> <td align="right"> 24.00 </td> <td align="right"> 2121.27 </td> <td align="right"> 2091.03 </td> </tr>
  <tr> <td> 4 </td> <td> M </td> <td> E </td> <td> P3 </td> <td align="right"> 49861.02 </td> <td align="right"> 14.00 </td> <td align="right"> 2125.68 </td> <td align="right"> 2096.24 </td> </tr>
  <tr> <td> 3 </td> <td> M </td> <td> ABCD </td> <td> P1 </td> <td align="right"> 204610.22 </td> <td align="right"> 108.00 </td> <td align="right"> 2129.60 </td> <td align="right"> 2115.30 </td> </tr>
  <tr> <td> 4 </td> <td> M </td> <td> ABCD </td> <td> P1 </td> <td align="right"> 273757.03 </td> <td align="right"> 144.00 </td> <td align="right"> 2134.03 </td> <td align="right"> 2120.57 </td> </tr>
  <tr> <td> 1 </td> <td> F </td> <td> ABCD </td> <td> P2 </td> <td align="right"> 229982.48 </td> <td align="right"> 122.00 </td> <td align="right"> 2137.25 </td> <td align="right"> 2114.71 </td> </tr>
  <tr> <td> 6 </td> <td> M </td> <td> E </td> <td> P2 </td> <td align="right"> 20723.49 </td> <td align="right"> 4.00 </td> <td align="right"> 2145.96 </td> <td align="right"> 2100.70 </td> </tr>
  <tr> <td> 6 </td> <td> F </td> <td> F </td> <td> P3 </td> <td align="right"> 4993.37 </td> <td align="right"> 3.00 </td> <td align="right"> 2151.82 </td> <td align="right"> 2100.83 </td> </tr>
  <tr> <td> 3 </td> <td> M </td> <td> E </td> <td> P2 </td> <td align="right"> 5918.78 </td> <td align="right"> 5.00 </td> <td align="right"> 2206.22 </td> <td align="right"> 2174.05 </td> </tr>
  <tr> <td> 4 </td> <td> M </td> <td> E </td> <td> P2 </td> <td align="right"> 52690.12 </td> <td align="right"> 11.00 </td> <td align="right"> 2210.80 </td> <td align="right"> 2179.47 </td> </tr>
  <tr> <td> 3 </td> <td> F </td> <td> F </td> <td> P3 </td> <td align="right"> 46057.12 </td> <td align="right"> 34.00 </td> <td align="right"> 2212.25 </td> <td align="right"> 2174.19 </td> </tr>
  <tr> <td> 4 </td> <td> F </td> <td> F </td> <td> P3 </td> <td align="right"> 34487.43 </td> <td align="right"> 20.00 </td> <td align="right"> 2216.85 </td> <td align="right"> 2179.61 </td> </tr>
  <tr> <td> 5 </td> <td> M </td> <td> E </td> <td> P1 </td> <td align="right"> 7157.38 </td> <td align="right"> 7.00 </td> <td align="right"> 2243.62 </td> <td align="right"> 2204.46 </td> </tr>
  <tr> <td> 2 </td> <td> F </td> <td> E </td> <td> P1 </td> <td align="right"> 13886.76 </td> <td align="right"> 9.00 </td> <td align="right"> 2272.84 </td> <td align="right"> 2238.42 </td> </tr>
  <tr> <td> 3 </td> <td> F </td> <td> F </td> <td> P2 </td> <td align="right"> 56496.29 </td> <td align="right"> 19.00 </td> <td align="right"> 2300.84 </td> <td align="right"> 2260.52 </td> </tr>
  <tr> <td> 4 </td> <td> F </td> <td> F </td> <td> P2 </td> <td align="right"> 32559.16 </td> <td align="right"> 7.00 </td> <td align="right"> 2305.62 </td> <td align="right"> 2266.15 </td> </tr>
  <tr> <td> 2 </td> <td> M </td> <td> E </td> <td> P3 </td> <td align="right"> 44224.71 </td> <td align="right"> 32.00 </td> <td align="right"> 2335.25 </td> <td align="right"> 2301.03 </td> </tr>
  <tr> <td> 5 </td> <td> F </td> <td> F </td> <td> P1 </td> <td align="right"> 2783.49 </td> <td align="right"> 2.00 </td> <td align="right"> 2339.85 </td> <td align="right"> 2292.13 </td> </tr>
  <tr> <td> 2 </td> <td> M </td> <td> ABCD </td> <td> P1 </td> <td align="right"> 168426.88 </td> <td align="right"> 80.00 </td> <td align="right"> 2344.42 </td> <td align="right"> 2327.74 </td> </tr>
  <tr> <td> 1 </td> <td> F </td> <td> E </td> <td> P3 </td> <td align="right"> 8017.28 </td> <td align="right"> 5.00 </td> <td align="right"> 2392.30 </td> <td align="right"> 2347.96 </td> </tr>
  <tr> <td> 1 </td> <td> F </td> <td> ABCD </td> <td> P1 </td> <td align="right"> 163074.98 </td> <td align="right"> 76.00 </td> <td align="right"> 2401.69 </td> <td align="right"> 2375.21 </td> </tr>
  <tr> <td> 5 </td> <td> M </td> <td> F </td> <td> P3 </td> <td align="right"> 46239.91 </td> <td align="right"> 9.00 </td> <td align="right"> 2404.10 </td> <td align="right"> 2356.25 </td> </tr>
  <tr> <td> 6 </td> <td> M </td> <td> E </td> <td> P1 </td> <td align="right"> 4026.32 </td> <td align="right"> 9.00 </td> <td align="right"> 2411.47 </td> <td align="right"> 2359.48 </td> </tr>
  <tr> <td> 2 </td> <td> M </td> <td> E </td> <td> P2 </td> <td align="right"> 7568.42 </td> <td align="right"> 11.00 </td> <td align="right"> 2428.76 </td> <td align="right"> 2392.39 </td> </tr>
  <tr> <td> 2 </td> <td> F </td> <td> F </td> <td> P3 </td> <td align="right"> 101384.81 </td> <td align="right"> 43.00 </td> <td align="right"> 2435.40 </td> <td align="right"> 2392.54 </td> </tr>
  <tr> <td> 1 </td> <td> M </td> <td> ABCD </td> <td> P3 </td> <td align="right"> 213348.43 </td> <td align="right"> 78.00 </td> <td align="right"> 2467.64 </td> <td align="right"> 2441.65 </td> </tr>
  <tr> <td> 3 </td> <td> M </td> <td> E </td> <td> P1 </td> <td align="right"> 2177.30 </td> <td align="right"> 2.00 </td> <td align="right"> 2479.19 </td> <td align="right"> 2441.86 </td> </tr>
  <tr> <td> 4 </td> <td> M </td> <td> E </td> <td> P1 </td> <td align="right"> 18164.05 </td> <td align="right"> 9.00 </td> <td align="right"> 2484.35 </td> <td align="right"> 2447.95 </td> </tr>
  <tr> <td> 1 </td> <td> F </td> <td> E </td> <td> P2 </td> <td align="right"> 5835.85 </td> <td align="right"> 5.00 </td> <td align="right"> 2488.10 </td> <td align="right"> 2441.19 </td> </tr>
  <tr> <td> 5 </td> <td> M </td> <td> F </td> <td> P2 </td> <td align="right"> 12066.20 </td> <td align="right"> 5.00 </td> <td align="right"> 2500.37 </td> <td align="right"> 2449.80 </td> </tr>
  <tr> <td> 2 </td> <td> F </td> <td> F </td> <td> P2 </td> <td align="right"> 35546.70 </td> <td align="right"> 13.00 </td> <td align="right"> 2532.93 </td> <td align="right"> 2487.54 </td> </tr>
  <tr> <td> 1 </td> <td> M </td> <td> ABCD </td> <td> P2 </td> <td align="right"> 151417.25 </td> <td align="right"> 62.00 </td> <td align="right"> 2566.46 </td> <td align="right"> 2538.60 </td> </tr>
  <tr> <td> 3 </td> <td> F </td> <td> F </td> <td> P1 </td> <td align="right"> 1792.19 </td> <td align="right"> 3.00 </td> <td align="right"> 2585.52 </td> <td align="right"> 2538.98 </td> </tr>
  <tr> <td> 3 </td> <td> M </td> <td> F </td> <td> P3 </td> <td align="right"> 36945.01 </td> <td align="right"> 26.00 </td> <td align="right"> 2656.51 </td> <td align="right"> 2610.00 </td> </tr>
  <tr> <td> 4 </td> <td> M </td> <td> F </td> <td> P3 </td> <td align="right"> 55052.36 </td> <td align="right"> 15.00 </td> <td align="right"> 2662.04 </td> <td align="right"> 2616.50 </td> </tr>
  <tr> <td> 2 </td> <td> M </td> <td> E </td> <td> P1 </td> <td align="right"> 38062.92 </td> <td align="right"> 6.00 </td> <td align="right"> 2729.27 </td> <td align="right"> 2687.10 </td> </tr>
  <tr> <td> 3 </td> <td> M </td> <td> F </td> <td> P2 </td> <td align="right"> 21796.48 </td> <td align="right"> 7.00 </td> <td align="right"> 2762.89 </td> <td align="right"> 2713.63 </td> </tr>
  <tr> <td> 4 </td> <td> M </td> <td> F </td> <td> P2 </td> <td align="right"> 8965.32 </td> <td align="right"> 8.00 </td> <td align="right"> 2768.64 </td> <td align="right"> 2720.39 </td> </tr>
  <tr> <td> 1 </td> <td> F </td> <td> E </td> <td> P1 </td> <td align="right"> 16903.95 </td> <td align="right"> 9.00 </td> <td align="right"> 2795.95 </td> <td align="right"> 2741.90 </td> </tr>
  <tr> <td> 5 </td> <td> M </td> <td> F </td> <td> P1 </td> <td align="right"> 7310.71 </td> <td align="right"> 2.00 </td> <td align="right"> 2809.74 </td> <td align="right"> 2751.58 </td> </tr>
  <tr> <td> 2 </td> <td> F </td> <td> F </td> <td> P1 </td> <td align="right"> 1401.77 </td> <td align="right"> 3.00 </td> <td align="right"> 2846.33 </td> <td align="right"> 2793.96 </td> </tr>
  <tr> <td> 1 </td> <td> M </td> <td> E </td> <td> P3 </td> <td align="right"> 87116.26 </td> <td align="right"> 17.00 </td> <td align="right"> 2872.73 </td> <td align="right"> 2818.60 </td> </tr>
  <tr> <td> 1 </td> <td> M </td> <td> ABCD </td> <td> P1 </td> <td align="right"> 94418.41 </td> <td align="right"> 51.00 </td> <td align="right"> 2884.00 </td> <td align="right"> 2851.31 </td> </tr>
  <tr> <td> 2 </td> <td> M </td> <td> F </td> <td> P3 </td> <td align="right"> 24323.16 </td> <td align="right"> 26.00 </td> <td align="right"> 2924.49 </td> <td align="right"> 2872.12 </td> </tr>
  <tr> <td> 1 </td> <td> M </td> <td> E </td> <td> P2 </td> <td align="right"> 28476.67 </td> <td align="right"> 7.00 </td> <td align="right"> 2987.76 </td> <td align="right"> 2930.51 </td> </tr>
  <tr> <td> 1 </td> <td> F </td> <td> F </td> <td> P3 </td> <td align="right"> 8258.59 </td> <td align="right"> 9.00 </td> <td align="right"> 2995.93 </td> <td align="right"> 2930.70 </td> </tr>
  <tr> <td> 2 </td> <td> M </td> <td> F </td> <td> P2 </td> <td align="right"> 18357.16 </td> <td align="right"> 6.00 </td> <td align="right"> 3041.60 </td> <td align="right"> 2986.15 </td> </tr>
  <tr> <td> 3 </td> <td> M </td> <td> F </td> <td> P1 </td> <td align="right"> 47952.73 </td> <td align="right"> 5.00 </td> <td align="right"> 3104.75 </td> <td align="right"> 3047.90 </td> </tr>
  <tr> <td> 1 </td> <td> F </td> <td> F </td> <td> P2 </td> <td align="right"> 10839.49 </td> <td align="right"> 7.00 </td> <td align="right"> 3115.90 </td> <td align="right"> 3047.06 </td> </tr>
  <tr> <td> 1 </td> <td> M </td> <td> E </td> <td> P1 </td> <td align="right"> 490.00 </td> <td align="right"> 1.00 </td> <td align="right"> 3357.44 </td> <td align="right"> 3291.51 </td> </tr>
  <tr> <td> 2 </td> <td> M </td> <td> F </td> <td> P1 </td> <td align="right"> 6950.54 </td> <td align="right"> 4.00 </td> <td align="right"> 3417.93 </td> <td align="right"> 3354.00 </td> </tr>
  <tr> <td> 1 </td> <td> F </td> <td> F </td> <td> P1 </td> <td align="right"> 14113.60 </td> <td align="right"> 6.00 </td> <td align="right"> 3501.43 </td> <td align="right"> 3422.41 </td> </tr>
  <tr> <td> 1 </td> <td> M </td> <td> F </td> <td> P3 </td> <td align="right"> 127855.12 </td> <td align="right"> 13.00 </td> <td align="right"> 3597.58 </td> <td align="right"> 3518.14 </td> </tr>
  <tr> <td> 1 </td> <td> M </td> <td> F </td> <td> P2 </td> <td align="right"> 11469.09 </td> <td align="right"> 1.00 </td> <td align="right"> 3741.64 </td> <td align="right"> 3657.83 </td> </tr>
  <tr> <td> 1 </td> <td> M </td> <td> F </td> <td> P1 </td> <td align="right"> 8120.13 </td> <td align="right"> 1.00 </td> <td align="right"> 4204.60 </td> <td align="right"> 4108.42 </td> </tr>
   </table>
