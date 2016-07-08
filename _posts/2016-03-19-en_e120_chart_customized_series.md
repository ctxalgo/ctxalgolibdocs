---
title: Chart customized series
layout: post
category: charting
language: en
---

```python
from ctxalgolib.ohlc.periodicity import Periodicity
from ctxalgolib.data_feed.web_data_feed import WebFutureDataFeed
from ctxalgolib.ohlc.ohlc_generator import OhlcGeneratorConstants
from ctxalgolib.charting.charts import Charts
from ctxalgolib.data_feed.ohlc_based_data_feed import OhlcBasedDataFeed


# Get some ohlc from web, please replace ohlc with your own ohlc object.
f = WebFutureDataFeed()
instrument_id = 'IF99'
period = Periodicity.WEEKLY
ohlc = f.ohlc(instrument_id, start_time='2014-01-01', end_time='2015-12-31', periodicity=period)


# Construct some ad-hoc time series based on the ohlc. series_t stores x-axis points, series_d stores y-axis points.
series_t = []
series_d = []
length = len(ohlc)
for i in range(1, 10):
    series_t.append(ohlc.dates[length * i/10])
    series_d.append(ohlc.closes[length * i / 10])


# Construct a data feed for charting.
feed = OhlcBasedDataFeed({
    OhlcGeneratorConstants.time_based: {
        instrument_id: {
            period: ohlc
        }
    }
})

# Chart
c = Charts(data_feed=feed)
c.set_instrument(instrument_id)
c.period(period)

# Draw ohlc.
c.height(k=500)        # Set the height of the K-bar area.
c.ohlc(position='k')   # Draw ohlc.
c.volume()             # Draw volume.

# Draw the ad-hoc time series.
c.series(series_t, series_d, position='k', color='red', thick=1)

# Show the chart in browser.
file_path = c.show()
print(file_path)





```
