---
title: Simple pattern recognition strategy from the Chan Theory
layout: post
category: ta
language: en
---

This example shows how to use this strategy for a single instrument(cu99) and present the results on the weekly chart.
It also contains the info of each recognized trend(the start pattern date,the end pattern date,
the price change(percentage), the direction), which will be used for the machine learning

```python
from ctxalgolib.data_feed.web_data_feed import WebFutureDataFeed
from ctxalgolib.ohlc.ohlc_generator import OhlcGeneratorConstants
from ctxalgolib.charting.charts import Charts
from ctxalgolib.data_feed.ohlc_based_data_feed import OhlcBasedDataFeed
from ctxalgolib.ohlc.periodicity import Periodicity
# from ctxalgoctp.ctp.backtesting_utils import *
from ctxalgolib.ta.piecewise_linear_regression.chan_piecewise_linear_regression import ChanPiecewiseLinearRegression
from statsmodels.tsa.stattools import acf, pacf
import matplotlib.pylab as plt
import rpy2.robjects as robjects


instrument = 'cu99'
chan_period = Periodicity.HOURLY

data_feed = WebFutureDataFeed()
```

## 1. Search for pattern by using the ChanPiecewiseLinearRegression class instance
Get all the info of the recognized patterns, the period is WEEKLY.
pattern_position: includes all the pattern positions in all the bars
pattern_type: includes the types of all the patterns, +1 means peak pattern, -1 means bottom pattern
pattern_time: includes the dates of all the patterns


```python
chan = ChanPiecewiseLinearRegression()
section = chan.regress_from_ohlc(ohlc)

# The following is drawing the recognized weekly patterns on the datafeed chart
# Construct some ad-hoc time series based on the ohlc. series_t stores x-axis points, series_d stores y-axis points.
series_t = chan.pattern_time
series_d = []
for i in range(len(chan.pattern_position)):
    series_d.append(section[i][1])
    # if chan.pattern_type[i] == 1:
    #     series_d.append(ohlc.highs[chan.pattern_position[i]])
    # elif chan.pattern_type[i] == -1:
    #     series_d.append(ohlc.lows[chan.pattern_position[i]])

# Construct a data feed for charting.
feed = OhlcBasedDataFeed({
    OhlcGeneratorConstants.time_based: {
        instrument: {
            chan_period: ohlc
        }
    }
})

# Chart
c = Charts(data_feed=feed)
c.set_instrument(instrument)

# Need to tell the chart component under which period the ohlc period is, default is DAILY
# Or it will connect the empty bar,
c.period(chan_period)

# Draw ohlc.
c.height(k=500)  # Set the height of the K-bar area.
c.ohlc(position='k')  # Draw ohlc.
c.volume()  # Draw volume.

# Draw the ad-hoc time series.
c.series(series_t, series_d, position='k', color='red', thick=1)  # use line to connect the points

# Show the chart in browser.
file_path = c.show()
print(file_path)

```

## 2. Obtain the info of each trend by calling the list variable trend_info_set
trend_info_set: this 2-D list trend_info[][] contains the following info of each trend
            (the pattern time range, the price change(percentage), the direction),
            which is used for the statistical time series analysis(ARIMA+GARCH models to fit and predict)
            The price change(percentage) is: (previous pattern price - current price) / previous price,
            downtrend is negative, uptrend is positive

            When connect from the peak pattern(pattern_type = 1) to the bottom pattern(pattern_type = -1),
            the direction of the trend is down trend, its direction value is -1

            When connect from the peak pattern(pattern_type = -1) to the bottom pattern(pattern_type = 1),
            the direction of the trend is up trend, its direction value is +1

```python
trend_info = chan.trend_info_set
print trend_info

# about trading days
print chan.trading_days
trading_days_difference = []
trading_days_difference_difference = []

for n in range(len(chan.trading_days)-1):
    trading_days_difference.append(chan.trading_days[n+1] - chan.trading_days[n])
# print trading_days_difference

for n in range(len(trading_days_difference)-1):
    trading_days_difference_difference.append(trading_days_difference[n+1] - trading_days_difference[n])
# print trading_days_difference_difference

# lag_acf_time = acf(chan.trading_days, nlags=30)
# lag_pacf_time = pacf(chan.trading_days, nlags=30)

lag_acf_time = acf(trading_days_difference, nlags=30)
lag_pacf_time = pacf(trading_days_difference, nlags=30)

# lag_acf_time = acf(trading_days_difference_difference, nlags=30)
# lag_pacf_time = pacf(trading_days_difference_difference, nlags=30)

# about trading_price_change
price_change_abs = []
price_change_difference = []
price_change_difference_difference = []
for n in range(len(chan.price_change)-1):
    price_change_abs.append(abs(chan.price_change[n]))

for n in range(len(price_change_abs)-1):
    price_change_difference.append(price_change_abs[n+1] - price_change_abs[n])

for n in range(len(price_change_difference)-1):
    price_change_difference_difference.append(price_change_difference[n+1] - price_change_difference[n])

# lag_acf_price = acf(price_change_abs, nlags=30)
# lag_pacf_price = pacf(price_change_abs, nlags=30)

lag_acf_price = acf(price_change_difference, nlags=30)
lag_pacf_price = pacf(price_change_difference, nlags=30)

# lag_acf_price = acf(price_change_difference_difference, nlags=30)
# lag_pacf_price = pacf(price_change_difference_difference, nlags=30)

# print lag_acf_time
# print lag_pacf_time
# print lag_acf_price
# print lag_pacf_price

plt.subplot(321)
plt.stem(lag_acf_time, marker='o', linestyle='--')
# plt.plot(lag_acf_time, marker='o', linestyle='--')
plt.grid(True)
plt.axhline(0, color='black', lw=2)
plt.title('acf_time')

plt.subplot(322)
plt.stem(lag_pacf_time, marker='o', linestyle='--')
plt.grid(True)
plt.axhline(0, color='black', lw=2)
plt.title('pacf_time')

plt.subplot(323)
plt.stem(lag_acf_price, marker='o', linestyle='--')
plt.grid(True)
plt.axhline(0, color='black', lw=2)
plt.title('acf_price')

plt.subplot(324)
plt.stem(lag_pacf_price, marker='o', linestyle='--')
plt.grid(True)
plt.axhline(0, color='black', lw=2)
plt.title('pacf_price')

# Open the file for writing
dataFile = open('writetest.txt', 'w')
# Loop through each item in the list
# and write it to the output file.
for eachitem in chan.price_change:
    dataFile.write(str(eachitem)+'\n')
# Close the output file
dataFile.close()

"""
trading_days_difference_float = []
for n in range(len(trading_days_difference)-1):
    trading_days_difference_float.append(float(trading_days_difference[n]))
arima811_days = sm.tsa.ARIMA(trading_days_difference_float, (8, 1, 1)).fit()
print arima811_days.summary()
"""
in_sample_size = 100
price_change_difference_in_sample = []
price_change_difference_predict_out_of_sample = []

for n in range(in_sample_size-1):
    price_change_difference_in_sample.append(price_change_difference[n])

lag_acf_price_in_sample = acf(price_change_difference_in_sample, nlags=30)
lag_pacf_price_in_sample = pacf(price_change_difference_in_sample, nlags=30)

# print arima511_price.plot_predict()

plt.subplot(325)
plt.stem(lag_acf_price_in_sample, marker='o', linestyle='--')
plt.grid(True)
plt.axhline(0, color='black', lw=2)
plt.title('acf_price_in_sample')
plt.xlabel('lag')

plt.subplot(326)
plt.stem(lag_pacf_price_in_sample, marker='o', linestyle='--')
plt.grid(True)
plt.axhline(0, color='black', lw=2)
plt.title('pacf_price_in_sample')
plt.xlabel('lag')

plt.draw()
plt.show()






```
