---
title: Incrementally getting historical data.
layout: post
category: docs_data
language: en
---

This example demonstrates how to get historical data incrementally.

```python
from datetime import datetime
from ctxalgolib.ohlc.periodicity import Periodicity
from ctxalgolib.data_feed.historical_data_fetcher import HistoricalLocalDataFeed
from ctxalgolib.ohlc.ohlc_generator import OhlcGeneratorConstants

folder = 'c:\\tmp\\ohlc'
# Specify the instruments, the period and the time range to download trading data.
# time_time set to None means to download until the last available data.
feed = HistoricalLocalDataFeed(
    folder, sids=['cu00', 'i00'], period=Periodicity.ONE_MINUTE,
    start_time=datetime(2017, 1, 1), end_time=None,
    profits=True, dominants=True)
ohlc = feed.ohlc('cu00', periodicity=Periodicity.ONE_MINUTE)
```

If you run the script day after day, the timestamp of the first ohlc bar does not change because the `start_time`
in `spec` does not change, but the timestamp of the last bar changes as more and more data is available in database.


```python
print(ohlc.dates[0])
print(ohlc.dates[-1])

```
