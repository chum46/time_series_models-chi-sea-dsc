
# Time Series Models

If we think back to our lecture on the bias-variance tradeoff, a perfect model is not possible.  There will always be inexplicable error. In time series modeling, we call that noise.  A timeseries that is completely random is called whitenoise, and is written mathematically as:

$$\Large Y_t =  \epsilon_t$$


```python
# Let's make some white noise!
from random import gauss as gs
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import warnings
warnings.filterwarnings('ignore')

rands = []
for _ in range(1000):
    rands.append(gs(0, 1))
series = pd.Series(rands)
```


```python
X = np.linspace(-10, 10, 1000)
plt.figure(figsize=(10, 7))
plt.plot(X, series);
```


![png](index_files/index_3_0.png)


We know this data has no true pattern governing its fluctuations (because we coded it with a random function).

Any attempt at a model would be fruitless.  The next point in the series could be any value, completely independent of the previous value.

We will assume that the timeseries data that we are working with is more than just white noise.

# Train Test Split

Let's reimport our chicago gun crime data, and prepare it in the same manner as the last notebook.



```python
ts = pd.read_csv('data/Gun_Crimes_Heat_Map.csv')
ts['Date'] = pd.to_datetime(ts.Date)
ts_minute = ts.groupby('Date').count()['ID']
daily_count = ts_minute.resample('D').sum()
daily_count = daily_count[daily_count < 90]

ts_dr = pd.date_range(daily_count.index[0], daily_count.index[-1])
ts_daily = np.empty(shape=len(ts_dr))
ts_daily = pd.Series(ts_daily)
ts_daily = ts_daily.reindex(ts_dr)
ts_daily = ts_daily.fillna(daily_count)
ts_daily = ts_daily.interpolate()
```

We are going to resample to the week level for this notebook.


```python
# Resample to week
```


```python
# Code
```


```python
ts_weekly = ts_daily.resample('W').mean()
fig, ax = plt.subplots()
ax.plot(ts_weekly)
```




    [<matplotlib.lines.Line2D at 0x118194ac8>]




![png](index_files/index_12_1.png)


Train test split for a time series is a little different than what we are used to.  Because chronological order matters, we cannot randomly sample points in our data.  Instead, we cut off a portion of our data at the end, and reserve it as our test set.


```python
round(ts_weekly.shape[0]*.8)
```




    271




```python
train = ts_weekly[:round(ts_weekly.shape[0]*.8)]
test = ts_weekly[round(ts_weekly.shape[0]*.8):]

fig, ax = plt.subplots()
ax.plot(train)
ax.plot(test)
ax.set_title('Train test Split')
```




    Text(0.5, 1.0, 'Train test Split')




![png](index_files/index_15_1.png)


We will now set aside our test set, and build our model on the train.

# Random Walk

A next logical step would be to simply predict the next data point with the point previous to it.  

We call this type of time series a random walk, and it is written mathematicall like so.

$$\Large Y_t = Y_{t-1} + \epsilon_t$$

$\epsilon$ represents white noise error. 

$$\Large Y_t - Y_{t-1}=  \epsilon_t$$

Let's bring back our Chicago gun crime data and make a simple random walk model.


```python
# we can perform this with the shift operator

random_walk = train.shift(1)

import matplotlib.pyplot as plt
fig, ax = plt.subplots()

train[0:30].plot(ax=ax, c='r', label='original')
random_walk[0:30].plot(ax=ax, c='b', label='shifted')
plt.legend()
```




    <matplotlib.legend.Legend at 0x1187c75c0>




![png](index_files/index_19_1.png)


For a baseline to compare our later models, lets calculate our RMSE for the random walk


```python
residuals = random_walk - train
np.sqrt((residuals.dropna()**2).sum())
```




    77.18876411837702



Now, lets plot the residuals.


```python
plt.plot(residuals.index, residuals)
plt.plot(residuals.index, residuals.rolling(30).std())
```




    [<matplotlib.lines.Line2D at 0x1188f0e80>]




![png](index_files/index_23_1.png)



```python
plt.plot(residuals.index, residuals.rolling(30).var())
```




    [<matplotlib.lines.Line2D at 0x1188954a8>]




![png](index_files/index_24_1.png)


If we look at the rolling standard deviation of our errors, we can see that the performance of our model varies at different points in time.

That is a result of the trends in our data.

In the previous notebook, we were able to make our series stationary by differencing our data. 

Let's repeat that process here. 

In order to make our life easier, we will use statsmodels to difference our data via the ARIMA class. 

We will break down what ARIMA is shortly, but for now, we will focus on the I, which stands for integrated.  A time series which needs to be differenced is said to be integrated [1](https://people.duke.edu/~rnau/411arim.htm). 

There is an order parameter in ARIMA with three slots: (p, d, q).  d represents our order of differencing, so putting a one there in our model will apply a first order difference.





```python
from statsmodels.tsa.arima_model import ARIMA
```


```python
rw = ARIMA(train, (0,1,0)).fit()
rw.predict(typ='levels')
```




    2014-01-12    31.187619
    2014-01-19    18.987619
    2014-01-26    24.559048
    2014-02-02    24.559048
    2014-02-09    22.273333
                    ...    
    2019-02-10    24.559048
    2019-02-17    27.844762
    2019-02-24    30.701905
    2019-03-03    30.844762
    2019-03-10    28.701905
    Freq: W-SUN, Length: 270, dtype: float64



We can see that the differenced predictions (d=1) are just a random walk


```python
random_walk
```




    2014-01-05          NaN
    2014-01-12    31.200000
    2014-01-19    19.000000
    2014-01-26    24.571429
    2014-02-02    24.571429
                    ...    
    2019-02-10    24.571429
    2019-02-17    27.857143
    2019-02-24    30.714286
    2019-03-03    30.857143
    2019-03-10    28.714286
    Freq: W-SUN, Length: 271, dtype: float64




```python
Visually, our differenced data looks more like white noise:
```


```python
fig, ax = plt.subplots()
ax.plot(train.diff())
ax.set_title('Weekly differenced data')
```




    Text(0.5, 1.0, 'Weekly differenced data')




![png](index_files/index_32_1.png)


By removing the trend from our data, we assume that our data's mean and variance are constant throughout.  But it is not just white noise.  If it were, our models could do no better than random predictions around the mean.  

Our task now is to find more patterns in the series.  

We will focus on the data points near to the point in question.  We can attempt to find patterns to how much influence previous points in the sequence have. 

If that made you think of regression, great! What we will be doing is assigning weights, like our betas, to previous points.

# The Autoregressive Model (AR)

Our next attempt at a model is the autoregressive model, which is a timeseries regressed on its previous values

### $y_{t} = c + \phi_{1}y_{t-1} + \varepsilon_{t}$

The above formula is a first order autoregressive model (AR1), which finds the best fit weight $\phi$ which, multiplied by the point previous to a point in question, yields the best fit model. 

In our ARIMA model, the p variable of the order (p,d,q) represents the AR term.  For a first order AR model, we put a 1 there.


```python
ar_1 = ARIMA(train, (1,1,0)).fit()

# We put a typ='levels' to convert our predictions to remove the differencing performed.
ar_1.predict(typ='levels')
```




    2014-01-12    31.198536
    2014-01-19    22.556496
    2014-01-26    22.944514
    2014-02-02    24.569538
    2014-02-09    22.950500
                    ...    
    2019-02-10    26.027893
    2019-02-17    26.896905
    2019-02-24    29.879050
    2019-03-03    30.813585
    2019-03-10    29.337404
    Freq: W-SUN, Length: 270, dtype: float64



The ARIMA class comes with a nice summary table.  


```python
ar_1.summary()
```




<table class="simpletable">
<caption>ARIMA Model Results</caption>
<tr>
  <th>Dep. Variable:</th>        <td>D.y</td>       <th>  No. Observations:  </th>    <td>270</td>  
</tr>
<tr>
  <th>Model:</th>          <td>ARIMA(1, 1, 0)</td>  <th>  Log Likelihood     </th> <td>-789.078</td>
</tr>
<tr>
  <th>Method:</th>             <td>css-mle</td>     <th>  S.D. of innovations</th>   <td>4.497</td> 
</tr>
<tr>
  <th>Date:</th>          <td>Thu, 09 Jul 2020</td> <th>  AIC                </th> <td>1584.156</td>
</tr>
<tr>
  <th>Time:</th>              <td>07:57:30</td>     <th>  BIC                </th> <td>1594.952</td>
</tr>
<tr>
  <th>Sample:</th>           <td>01-12-2014</td>    <th>  HQIC               </th> <td>1588.491</td>
</tr>
<tr>
  <th></th>                 <td>- 03-10-2019</td>   <th>                     </th>     <td> </td>   
</tr>
</table>
<table class="simpletable">
<tr>
      <td></td>         <th>coef</th>     <th>std err</th>      <th>z</th>      <th>P>|z|</th>  <th>[0.025</th>    <th>0.975]</th>  
</tr>
<tr>
  <th>const</th>     <td>   -0.0015</td> <td>    0.212</td> <td>   -0.007</td> <td> 0.994</td> <td>   -0.417</td> <td>    0.414</td>
</tr>
<tr>
  <th>ar.L1.D.y</th> <td>   -0.2917</td> <td>    0.059</td> <td>   -4.954</td> <td> 0.000</td> <td>   -0.407</td> <td>   -0.176</td>
</tr>
</table>
<table class="simpletable">
<caption>Roots</caption>
<tr>
    <td></td>   <th>            Real</th>  <th>         Imaginary</th> <th>         Modulus</th>  <th>        Frequency</th>
</tr>
<tr>
  <th>AR.1</th> <td>          -3.4285</td> <td>          +0.0000j</td> <td>           3.4285</td> <td>           0.5000</td>
</tr>
</table>



But, as you may notice, the output does not include RMSE.

It does include AIC. We briefly touched on AIC with linear regression.  It is a metrics that we used to penalize models for having too many features.  A better model has a lower AIC.

Let's compare the first order autoregressive model to our Random Walk.


```python
rw_model = ARIMA(train, (0,1,0)).fit()
rw_model.summary()
```




<table class="simpletable">
<caption>ARIMA Model Results</caption>
<tr>
  <th>Dep. Variable:</th>        <td>D.y</td>       <th>  No. Observations:  </th>    <td>270</td>  
</tr>
<tr>
  <th>Model:</th>          <td>ARIMA(0, 1, 0)</td>  <th>  Log Likelihood     </th> <td>-800.814</td>
</tr>
<tr>
  <th>Method:</th>               <td>css</td>       <th>  S.D. of innovations</th>   <td>4.698</td> 
</tr>
<tr>
  <th>Date:</th>          <td>Thu, 09 Jul 2020</td> <th>  AIC                </th> <td>1605.628</td>
</tr>
<tr>
  <th>Time:</th>              <td>07:57:33</td>     <th>  BIC                </th> <td>1612.825</td>
</tr>
<tr>
  <th>Sample:</th>           <td>01-12-2014</td>    <th>  HQIC               </th> <td>1608.518</td>
</tr>
<tr>
  <th></th>                 <td>- 03-10-2019</td>   <th>                     </th>     <td> </td>   
</tr>
</table>
<table class="simpletable">
<tr>
    <td></td>       <th>coef</th>     <th>std err</th>      <th>z</th>      <th>P>|z|</th>  <th>[0.025</th>    <th>0.975]</th>  
</tr>
<tr>
  <th>const</th> <td>   -0.0124</td> <td>    0.286</td> <td>   -0.043</td> <td> 0.965</td> <td>   -0.573</td> <td>    0.548</td>
</tr>
</table>



Our AIC for the AR(1) model is lower than the random walk, indicating improvement.  

Before abandoning it for AIC, let's just make sure the RMSE is lower as well.


```python
rmse = np.sqrt((ar_1.resid**2).sum())
print(rmse)
```

    73.97870623211666



```python
np.sqrt(sum(rw.resid**2))
```




    77.18849602348361



Checks out. RMSE is lower as well.

Autoregression, as we said before, is a regression of a time series on lagged values of itself.  

From the summary, we see the coefficient of the 1st lag:


```python
ar_1.arparams
```




    array([-0.29167099])



We come close to reproducing this coefficients with linear regression, with slight differences due to how statsmodels performs the regression. 


```python
from sklearn.linear_model import LinearRegression

lr = LinearRegression()
lr.fit(np.array(train.diff().shift(1).dropna()).reshape(-1,1), train[1:].diff().dropna())
print(lr.coef_)
```

    [-0.28545641]


We can also factor in more than just the most recent point.
$$\large y_{t} = c + \phi_{1}y_{t-1} + \phi_{2}y_{t-2}+ \varepsilon_{t}$$

We refer to the order of our AR model by the number of lags back we go.  The above formula refers to an AR(2) model.  We put a 2 in the p position of the ARIMA class order


```python
ar_2 = ARIMA(train, (2,1,0)).fit()

# We put a typ='levels' to convert our predictions to remove the differencing performed.
ar_2.predict(typ='levels')
```




    2014-01-12    31.205021
    2014-01-19    22.538712
    2014-01-26    26.234072
    2014-02-02    22.860489
    2014-02-09    23.160197
                    ...    
    2019-02-10    27.929185
    2019-02-17    28.163787
    2019-02-24    28.626233
    2019-03-03    29.929740
    2019-03-10    29.490555
    Freq: W-SUN, Length: 270, dtype: float64




```python
print(rw.aic)
print(ar_1.aic)
print(ar_2.aic)
```

    1605.628111571946
    1584.1562780341596
    1559.6786866515151


Our AIC improves with more lagged terms.

# Moving Average Model (MA)

The next type of model is based on error.  The idea behind the moving average model is to make a prediciton based on how far off we were the day before.

$$\large Y_t = \mu +\epsilon_t + \theta * \epsilon_{t-1}$$

The moving average model is a pretty cool idea. We make a prediction, see how far off we were, then adjust our next prediction by a factor of how far off our pervious predicion was.

In our ARIMA model, the q term of our order (p,d,q) refers to the MA component. To use one lagged error, we put 1 in the q position.



```python
ma_1 = ARIMA(train, (0,0,1)).fit()
y_hat = ma_1.predict(typ='levels')
y_hat
```




    2014-01-05    35.118190
    2014-01-12    33.285298
    2014-01-19    26.563703
    2014-01-26    33.823562
    2014-02-02    28.899878
                    ...    
    2019-02-10    32.311460
    2019-02-17    32.038346
    2019-02-24    34.202697
    2019-03-03    32.804978
    2019-03-10    32.289767
    Freq: W-SUN, Length: 271, dtype: float64




```python
ma_1.summary()
```




<table class="simpletable">
<caption>ARMA Model Results</caption>
<tr>
  <th>Dep. Variable:</th>         <td>y</td>        <th>  No. Observations:  </th>    <td>271</td>  
</tr>
<tr>
  <th>Model:</th>            <td>ARMA(0, 1)</td>    <th>  Log Likelihood     </th> <td>-850.315</td>
</tr>
<tr>
  <th>Method:</th>             <td>css-mle</td>     <th>  S.D. of innovations</th>   <td>5.571</td> 
</tr>
<tr>
  <th>Date:</th>          <td>Thu, 09 Jul 2020</td> <th>  AIC                </th> <td>1706.631</td>
</tr>
<tr>
  <th>Time:</th>              <td>08:05:21</td>     <th>  BIC                </th> <td>1717.437</td>
</tr>
<tr>
  <th>Sample:</th>           <td>01-05-2014</td>    <th>  HQIC               </th> <td>1710.970</td>
</tr>
<tr>
  <th></th>                 <td>- 03-10-2019</td>   <th>                     </th>     <td> </td>   
</tr>
</table>
<table class="simpletable">
<tr>
     <td></td>        <th>coef</th>     <th>std err</th>      <th>z</th>      <th>P>|z|</th>  <th>[0.025</th>    <th>0.975]</th>  
</tr>
<tr>
  <th>const</th>   <td>   35.1182</td> <td>    0.572</td> <td>   61.446</td> <td> 0.000</td> <td>   33.998</td> <td>   36.238</td>
</tr>
<tr>
  <th>ma.L1.y</th> <td>    0.6914</td> <td>    0.037</td> <td>   18.488</td> <td> 0.000</td> <td>    0.618</td> <td>    0.765</td>
</tr>
</table>
<table class="simpletable">
<caption>Roots</caption>
<tr>
    <td></td>   <th>            Real</th>  <th>         Imaginary</th> <th>         Modulus</th>  <th>        Frequency</th>
</tr>
<tr>
  <th>MA.1</th> <td>          -1.4463</td> <td>          +0.0000j</td> <td>           1.4463</td> <td>           0.5000</td>
</tr>
</table>



Let's see if we can reproduce the predictions above


```python
y_hat_manual = ((train - y_hat)*ma_1.maparams[0] +train.mean()).shift()
y_hat_manual
```




    2014-01-05          NaN
    2014-01-12    32.409142
    2014-01-19    25.241024
    2014-01-26    33.740776
    2014-02-02    28.721099
                    ...    
    2019-02-10    32.311562
    2019-02-17    32.038448
    2019-02-24    34.202798
    2019-03-03    32.805079
    2019-03-10    32.289869
    Freq: W-SUN, Length: 271, dtype: float64



Let's look at the 1st order MA model with a 1st order difference


```python
ma_1 = ARIMA(train, (0,1,1)).fit()

print(rw.aic)
print(ar_1.aic)
print(ar_2.aic)
print(ma_1.aic)

```

    1605.628111571946
    1584.1562780341596
    1559.6786866515151
    1561.6598781567423


It performs better than a 1st order AR, but worse than a 2nd order

Just like our AR models, we can lag back as far as we want. Our MA(2) model would use the past two lagged terms:

$$\large Y_t = \mu +\epsilon_t + \theta_{t-1} * \epsilon_{t-1} + \theta_2 * \epsilon_{t-2}$$

and our MA term would be two.


```python
ma_2 = ARIMA(train, (0,1,2)).fit()
y_hat = ma_2.predict(typ='levels')
y_hat
```




    2014-01-12    31.214963
    2014-01-19    22.331461
    2014-01-26    25.804439
    2014-02-02    24.659319
    2014-02-09    23.462409
                    ...    
    2019-02-10    29.178170
    2019-02-17    29.956661
    2019-02-24    30.679592
    2019-03-03    30.652317
    2019-03-10    29.480736
    Freq: W-SUN, Length: 270, dtype: float64




```python
print(rw.aic)
print(ar_1.aic)
print(ar_2.aic)
print(ma_1.aic)
print(ma_2.aic)
```

    1605.628111571946
    1584.1562780341596
    1559.6786866515151
    1561.6598781567423
    1555.9026879742858


# ARMA

We don't have to limit ourselves to just AR or MA.  We can use both AR terms and MA terms.

for example, an ARMA(2,1) model is given by:

 $$\large Y_t = \mu + \phi_1 Y_{t-1}+\phi_2 Y_{t-2}+ \theta \epsilon_{t-1}+\epsilon_t$$



```python
arma_21 = ARIMA(train, (2,1,1)).fit()
```


```python
print(rw.aic)
print(ar_1.aic)
print(ar_2.aic)
print(ma_1.aic)
print(arma_21.aic)
```

    1605.628111571946
    1584.1562780341596
    1559.6786866515151
    1561.6598781567423
    1558.0010402540224



```python
Best performance so far.
```

# ACF and PACF

We have been able to reduce our AIC by chance, adding fairly random p,d,q terms.

We have two tools to help guide us in these decisions: the autocorrelation and partial autocorrelation functions.

## ACF

The autocorrelation plot of our time series is simply a version of the correlation plots we used in linear regression.  Our features this time are prior points in the time series, or the lags. 

We can calculate a specific $\gamma_k$ with:

${\displaystyle \gamma_k = \frac 1 n \sum\limits_{t=1}^{n-k} (y_t - \bar{y})(y_{t+k}-\bar{y})}$


```python
from statsmodels.graphics.tsaplots import plot_acf, plot_pacf
```


```python

df = pd.DataFrame(train)
df.columns = ['lag_0']
df['lag_1'] = train.shift()
df['lag_2'] = train.shift(2)
df['lag_3'] = train.shift(3)
df['lag_4'] = train.shift(4)
df['lag_5'] = train.shift(5)
list(df.corr()['lag_0'].index)
plt.bar(list(df.corr()['lag_0'].index), list(df.corr()['lag_0']))
```




    <BarContainer object of 6 artists>




![png](index_files/index_77_1.png)



```python
# Original data

plot_acf(train);
```


![png](index_files/index_78_0.png)


The shaded area of the graph is the convidence interval.  When the autocorrelation drops into the shaded area, that means there is no longer statistically significant correlation between lags. 

The above autocorrelation shows that there is correlation between lags up to about 12 weeks back.  

When Looking at the ACF graph for the original data, we see a strong persistent correlation with higher order lags. This is evidence that we should take a first diefference of the data to remove this autocorrelation.

This makes sense, since we are trying to capture the effect of recent lags in our ARMA models, and with high correlation between distant lags, our models will not come close to the true process.


```python
plot_acf(train.diff().dropna());
```


![png](index_files/index_80_0.png)


Some rules of thumb:
  - If the autocorrelation shows positive correlation at the first lag, then try adding an AR term.
    
  - If the autocorrelatuion shows negative correlation at the first lag, try adding MA terms.
    
    

This autocorrelation plot can now be used to get an idea of a potential MA term.  Our differenced series shows negative significant correlation at lag of 1 suggests adding 1 MA term.  There is also a statistically significant 2nd, term, so adding another MA is another possibility.


> If the ACF of the differenced series displays a sharp cutoff and/or the lag-1 autocorrelation is negative--i.e., if the series appears slightly "overdifferenced"--then consider adding an MA term to the model. The lag at which the ACF cuts off is the indicated number of MA terms. [Duke](https://people.duke.edu/~rnau/411arim3.htm#signatures)

The ACF can be used to identify the possible structure of time series data. That can be tricky going forward as there often isn’t a single clear-cut interpretation of a sample autocorrelation function.

## PACF

In general, a partial correlation is a conditional correlation. It is the  amount of correlation between a variable and a lag of itself that is not explained by correlations at all lower-order-lags. The autocorrelation of a time series $Y$ at lag 1 is the coefficient of correlation between $Y_t$ and $Y_{t-1}$, which is presumably also the correlation between $Y_{t-1}$ and $Y_{t-2}$. But if $Y_t$ is correlated with $Y_{t-1}$, and $Y_{t-1}$ is equally correlated with $Y_{t-2}$, then we should also expect to find correlation between $Y_t$ and $Y_{t-2}$. Thus, the correlation at lag 1 "propagates" to lag 2 and presumably to higher-order lags. The partial autocorrelation at lag 2 is therefore the difference between the actual correlation at lag 2 and the expected correlation due to the propagation of correlation at lag 1.




```python
plot_pacf(train.diff().dropna());
```


![png](index_files/index_86_0.png)


When we run a linear regression on our lags, the coefficients calculated factor in the influence of the other variables.  This reminds us of our autoregressive model.  Since the PACF shows the direct effect of previous lags, it helps us choose AR terms.  If there is a significant positive value at a lag, consider adding an AR term according to the number that you see.

Some rules of thumb: 

    - A sharp drop after lag "k" suggests an AR-K model.
    - A gradual decline suggests an MA.

![alt text](./img/armaguidelines.png)

The plots above suggest that we should try a 1st order differenced MA(1) or MA(2) model on our weekly gun offense data.

This aligns with our AIC scores from above.

# auto_arima

Luckily for us, we have a Python package that will help us determine optimal terms.


```python
from pmdarima import auto_arima

auto_arima(train, start_p=0, start_q=0, max_p=6, max_q=3, seasonal=False, trace=True)
```

    Fit ARIMA: order=(0, 1, 0); AIC=1605.628, BIC=1612.825, Fit time=0.003 seconds
    Fit ARIMA: order=(1, 1, 0); AIC=1584.156, BIC=1594.952, Fit time=0.014 seconds
    Fit ARIMA: order=(0, 1, 1); AIC=1561.660, BIC=1572.455, Fit time=0.016 seconds
    Fit ARIMA: order=(1, 1, 1); AIC=1558.132, BIC=1572.526, Fit time=0.048 seconds
    Fit ARIMA: order=(1, 1, 2); AIC=1553.146, BIC=1571.138, Fit time=0.069 seconds
    Fit ARIMA: order=(2, 1, 3); AIC=1555.821, BIC=1581.010, Fit time=0.206 seconds
    Fit ARIMA: order=(0, 1, 2); AIC=1555.903, BIC=1570.296, Fit time=0.025 seconds
    Fit ARIMA: order=(2, 1, 2); AIC=1555.144, BIC=1576.735, Fit time=0.083 seconds
    Fit ARIMA: order=(1, 1, 3); AIC=1555.145, BIC=1576.735, Fit time=0.099 seconds
    Total fit time: 0.565 seconds





    ARIMA(callback=None, disp=0, maxiter=None, method=None, order=(1, 1, 2),
          out_of_sample_size=0, scoring='mse', scoring_args={}, seasonal_order=None,
          solver='lbfgs', start_params=None, suppress_warnings=False,
          transparams=True, trend=None, with_intercept=True)



According to auto_arima, our optimal model is a first order differenced, AR(1)MA(2) model.

Let's plot our training predictions.


```python
aa_model = ARIMA(train, (1,1,2)).fit()
y_hat_train = aa_model.predict(typ='levels')

fig, ax = plt.subplots()
ax.plot(y_hat_train)
ax.plot(train)
```




    [<matplotlib.lines.Line2D at 0x1a222af978>]




![png](index_files/index_96_1.png)



```python
# Let's zoom in:

fig, ax = plt.subplots()
ax.plot(y_hat_train[50:70])
ax.plot(train[50:70])
```




    [<matplotlib.lines.Line2D at 0x1a20e540f0>]




![png](index_files/index_97_1.png)



```python
aa_model.summary()
```




<table class="simpletable">
<caption>ARIMA Model Results</caption>
<tr>
  <th>Dep. Variable:</th>        <td>D.y</td>       <th>  No. Observations:  </th>    <td>270</td>  
</tr>
<tr>
  <th>Model:</th>          <td>ARIMA(1, 1, 2)</td>  <th>  Log Likelihood     </th> <td>-771.573</td>
</tr>
<tr>
  <th>Method:</th>             <td>css-mle</td>     <th>  S.D. of innovations</th>   <td>4.212</td> 
</tr>
<tr>
  <th>Date:</th>          <td>Thu, 09 Jul 2020</td> <th>  AIC                </th> <td>1553.146</td>
</tr>
<tr>
  <th>Time:</th>              <td>10:20:33</td>     <th>  BIC                </th> <td>1571.138</td>
</tr>
<tr>
  <th>Sample:</th>           <td>01-12-2014</td>    <th>  HQIC               </th> <td>1560.371</td>
</tr>
<tr>
  <th></th>                 <td>- 03-10-2019</td>   <th>                     </th>     <td> </td>   
</tr>
</table>
<table class="simpletable">
<tr>
      <td></td>         <th>coef</th>     <th>std err</th>      <th>z</th>      <th>P>|z|</th>  <th>[0.025</th>    <th>0.975]</th>  
</tr>
<tr>
  <th>const</th>     <td>    0.0121</td> <td>    0.112</td> <td>    0.108</td> <td> 0.914</td> <td>   -0.207</td> <td>    0.231</td>
</tr>
<tr>
  <th>ar.L1.D.y</th> <td>   -0.5158</td> <td>    0.152</td> <td>   -3.397</td> <td> 0.001</td> <td>   -0.813</td> <td>   -0.218</td>
</tr>
<tr>
  <th>ma.L1.D.y</th> <td>    0.1080</td> <td>    0.141</td> <td>    0.767</td> <td> 0.443</td> <td>   -0.168</td> <td>    0.384</td>
</tr>
<tr>
  <th>ma.L2.D.y</th> <td>   -0.4519</td> <td>    0.068</td> <td>   -6.614</td> <td> 0.000</td> <td>   -0.586</td> <td>   -0.318</td>
</tr>
</table>
<table class="simpletable">
<caption>Roots</caption>
<tr>
    <td></td>   <th>            Real</th>  <th>         Imaginary</th> <th>         Modulus</th>  <th>        Frequency</th>
</tr>
<tr>
  <th>AR.1</th> <td>          -1.9389</td> <td>          +0.0000j</td> <td>           1.9389</td> <td>           0.5000</td>
</tr>
<tr>
  <th>MA.1</th> <td>          -1.3728</td> <td>          +0.0000j</td> <td>           1.3728</td> <td>           0.5000</td>
</tr>
<tr>
  <th>MA.2</th> <td>           1.6119</td> <td>          +0.0000j</td> <td>           1.6119</td> <td>           0.0000</td>
</tr>
</table>



# Test

Now that we have chosen our parameters, let's try our model on the test set.


```python
test
```




    2019-03-17    31.285714
    2019-03-24    33.571429
    2019-03-31    36.571429
    2019-04-07    35.571429
    2019-04-14    34.857143
                    ...    
    2020-05-31    55.157143
    2020-06-07    59.771429
    2020-06-14    50.142857
    2020-06-21    53.857143
    2020-06-28    54.500000
    Freq: W-SUN, Length: 68, dtype: float64




```python
y_hat_test = aa_model.predict(start=test.index[0], end=test.index[-1],typ='levels')

fig, ax = plt.subplots()
ax.plot(y_hat_test)

```




    [<matplotlib.lines.Line2D at 0x1a2646d710>]




![png](index_files/index_102_1.png)



```python
fig, ax = plt.subplots()
ax.plot(y_hat_test)
ax.plot(test)
```




    [<matplotlib.lines.Line2D at 0x1a20ee6780>]




![png](index_files/index_103_1.png)


Our predictions on the test set certainly leave something to be desired.  

Let's take another look at our autocorrelation function of the original series.


```python
plot_acf(ts_weekly);
```


![png](index_files/index_105_0.png)


Let's increase the lags


```python
plot_acf(ts_weekly, lags=75);
```


![png](index_files/index_107_0.png)


There seems to be a wave of correlation at around 50 lags.
What is going on?

![verkempt](https://media.giphy.com/media/l3vRhBz4wCpJ9aEuY/giphy.gif)

# SARIMA

Looks like we may have some other forms of seasonality.  Luckily, we have SARIMA, which stands for Seasonal Auto Regressive Integrated Moving Average.  That is a lot.  The statsmodels package is actually called SARIMAX.  The X stands for exogenous, and we are only dealing with endogenous variables, but we can use SARIMAX as a SARIMA.



```python
from statsmodels.tsa.statespace.sarimax import SARIMAX
```

A seasonal ARIMA model is classified as an **ARIMA(p,d,q)x(P,D,Q)** model, 

    **p** = number of autoregressive (AR) terms 
    **d** = number of differences 
    **q** = number of moving average (MA) terms
     
    **P** = number of seasonal autoregressive (SAR) terms 
    **D** = number of seasonal differences 
    **Q** = number of seasonal moving average (SMA) terms


```python
import itertools
p = q = range(0, 2)
pdq = list(itertools.product(p, [1], q))
seasonal_pdq = [(x[0], x[1], x[2], 52) for x in list(itertools.product(p, [1], q))]
print('Examples of parameter for SARIMA...')
for i in pdq:
    for s in seasonal_pdq:
        print('SARIMAX: {} x {}'.format(i, s))

```

    Examples of parameter for SARIMA...
    SARIMAX: (0, 1, 0) x (0, 1, 0, 52)
    SARIMAX: (0, 1, 0) x (0, 1, 1, 52)
    SARIMAX: (0, 1, 0) x (1, 1, 0, 52)
    SARIMAX: (0, 1, 0) x (1, 1, 1, 52)
    SARIMAX: (0, 1, 1) x (0, 1, 0, 52)
    SARIMAX: (0, 1, 1) x (0, 1, 1, 52)
    SARIMAX: (0, 1, 1) x (1, 1, 0, 52)
    SARIMAX: (0, 1, 1) x (1, 1, 1, 52)
    SARIMAX: (1, 1, 0) x (0, 1, 0, 52)
    SARIMAX: (1, 1, 0) x (0, 1, 1, 52)
    SARIMAX: (1, 1, 0) x (1, 1, 0, 52)
    SARIMAX: (1, 1, 0) x (1, 1, 1, 52)
    SARIMAX: (1, 1, 1) x (0, 1, 0, 52)
    SARIMAX: (1, 1, 1) x (0, 1, 1, 52)
    SARIMAX: (1, 1, 1) x (1, 1, 0, 52)
    SARIMAX: (1, 1, 1) x (1, 1, 1, 52)



```python
for param in pdq:
    for param_seasonal in seasonal_pdq:
        try:
            mod =SARIMAX(train,order=param,seasonal_order=param_seasonal,enforce_stationarity=False,enforce_invertibility=False)
            results = mod.fit()
            print('ARIMA{}x{} - AIC:{}'.format(param,param_seasonal,results.aic))
        except: 
            print('hello')
            continue
```

    ARIMA(0, 1, 0)x(0, 1, 0, 52)12 - AIC:1406.5914910305441
    ARIMA(0, 1, 0)x(0, 1, 1, 52)12 - AIC:1039.6273546182113
    ARIMA(0, 1, 0)x(1, 1, 0, 52)12 - AIC:1055.4512482807704
    ARIMA(0, 1, 0)x(1, 1, 1, 52)12 - AIC:1038.419076265242
    ARIMA(0, 1, 1)x(0, 1, 0, 52)12 - AIC:1326.266685805947
    ARIMA(0, 1, 1)x(0, 1, 1, 52)12 - AIC:978.1309943041055
    ARIMA(0, 1, 1)x(1, 1, 0, 52)12 - AIC:1005.6947621192127
    ARIMA(0, 1, 1)x(1, 1, 1, 52)12 - AIC:980.71332272061
    ARIMA(1, 1, 0)x(0, 1, 0, 52)12 - AIC:1373.0472465290327
    ARIMA(1, 1, 0)x(0, 1, 1, 52)12 - AIC:1019.1651883767508
    ARIMA(1, 1, 0)x(1, 1, 0, 52)12 - AIC:1024.5423831365738
    ARIMA(1, 1, 0)x(1, 1, 1, 52)12 - AIC:1018.4857412836736
    ARIMA(1, 1, 1)x(0, 1, 0, 52)12 - AIC:1320.7264572171368
    ARIMA(1, 1, 1)x(0, 1, 1, 52)12 - AIC:973.5518935855749
    ARIMA(1, 1, 1)x(1, 1, 0, 52)12 - AIC:988.5066193103887
    ARIMA(1, 1, 1)x(1, 1, 1, 52)12 - AIC:975.3115937856937


Let's try the third from the bottom, ARIMA(1, 1, 1)x(0, 1, 1, 52)12 - AIC:973.5518935855749


```python
sari_mod =SARIMAX(train,order=(1,1,1),seasonal_order=(0,1,1,52),enforce_stationarity=False,enforce_invertibility=False).fit()

```


```python
y_hat_train = sari_mod.predict(typ='levels')
y_hat_test = sari_mod.predict(start=test.index[0], end=test.index[-1],typ='levels')

fig, ax = plt.subplots()
ax.plot(train)
ax.plot(test)
ax.plot(y_hat_train)
ax.plot(y_hat_test)
```




    [<matplotlib.lines.Line2D at 0x1a2f783b38>]




![png](index_files/index_118_1.png)



```python
# Let's zoom in on test
fig, ax = plt.subplots()

ax.plot(test)
ax.plot(y_hat_test)

```




    [<matplotlib.lines.Line2D at 0x1a26b82710>]




![png](index_files/index_119_1.png)


# Forecast

Lastly, let's predict into the future.

To do so, we refit to our entire training set.


```python
sari_mod =SARIMAX(ts_weekly,order=(1,1,1),seasonal_order=(0,1,1,52),enforce_stationarity=False,enforce_invertibility=False).fit()
```


```python
forecast = sari_mod.forecast(steps = 52)
```


```python
fig, ax = plt.subplots()

ax.plot(ts_weekly)
ax.plot(forecast)
ax.set_title('Chicago Gun Crime Predictions\n One Year out')
```




    Text(0.5, 1.0, 'Chicago Gun Crime Predictions\n One Year out')




![png](index_files/index_124_1.png)

