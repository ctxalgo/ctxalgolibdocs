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
spec = {
    OhlcGeneratorConstants.time_based: {
        'cu00': {
            Periodicity.ONE_MINUTE: {'start_time': datetime(2017, 1, 1), 'end_time': None},
        },
        'i00': {
            Periodicity.ONE_MINUTE: {'start_time': datetime(2017, 1, 1), 'end_time': None},
        }
    }
}

feed = HistoricalLocalDataFeed(spec, folder, profits=True, dominants=True, check_start_time=False)
ohlc = feed.ohlc('cu00')
```

If you run the script day after day, the timestamp of the first ohlc bar does not change because the `start_time`
in `spec` does not change, but the timestamp of the last bar changes as more and more data is available in database.


```python
print(ohlc.dates[0])
print(ohlc.dates[-1])

```
