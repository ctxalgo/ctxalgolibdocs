---
title: 增量获取历史ohlc数据
layout: post
category: docs_data
language: zh
---

本示例展示如何增量获取历史ohlc数据


```python
from datetime import datetime
from ctxalgolib.ohlc.periodicity import Periodicity
from ctxalgolib.data_feed.historical_data_fetcher import HistoricalLocalDataFeed

folder = 'c:\\tmp\\ohlc'
# Specify the instruments, the period and the time range to download trading data.
# time_time set to None means to download until the last available data.
# Use `profits`, `dominants` to specify additional fields in downloaded ohlc bars.
# Use `text_file` to switch between text and binary data storage. Binary data storage is much faster.
# If `enforce_fetch` is set to False, then no new data will be downloaded. It is faster when you already
# synced `folder` with the latest data and just want to retrieve and consume data.
feed = HistoricalLocalDataFeed(
    folder, sids=['cu00', 'i00', 'ag00'], period=Periodicity.ONE_MINUTE,
    start_time=datetime(2017, 1, 1), end_time=None,
    profits=True, dominants=True, text_file=False, enforce_fetch=True)

# Use the `ohlc` method to retrieve bars you want. the `start_time` and `end_time` here should be within the
# range specified in HistoricalLocalDataFeed above.
ohlc_1m = feed.ohlc('cu00', periodicity=Periodicity.ONE_MINUTE, start_time=datetime(2017, 1, 10), end_time=None)

# You can also retrieve ohlcs whose period is different from the period of the downloaded data.
ohlc_5m = feed.ohlc('cu00', periodicity=Periodicity.FIVE_MINUTE, start_time=datetime(2017, 1, 10), end_time=None)
```

如果你每天都运行该脚本，你会发现第一根K线的时间戳不改变，因为`spec`中的`start_time`没有变，但是最后一根K线的时间戳
会不断变化，因为越来越多的历史数据被加入到数据库中了。


```python
print(ohlc_1m.dates[0])
print(ohlc_1m.dates[-1])

```
