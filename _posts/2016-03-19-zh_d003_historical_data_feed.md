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

如果你每天都运行该脚本，你会发现第一根K线的时间戳不改变，因为`spec`中的`start_time`没有变，但是最后一根K线的时间戳
会不断变化，因为越来越多的历史数据被加入到数据库中了。


```python
print(ohlc.dates[0])
print(ohlc.dates[-1])

```
