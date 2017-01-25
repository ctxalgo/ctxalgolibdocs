---
title: Getting historical ohlc data.
layout: post
category: docs_data
language: en
---

This example shows how to get historical ohlc data.

```python
from ctxalgolib.ohlc.periodicity import Periodicity
from ctxalgolib.data_feed.web_mongo_data_feed import WebMongoDataFeed

feed = WebMongoDataFeed()

# Use actual instrument ids such as cu1703 or you can use dominant instrument id such as cu00 for dominant instrument,
# cu01 for sub-dominant instrument and cu02 for sub-sub-dominant instrument.
instrument_id = 'cu00'

# Use options of the ohlc method, such as with_open_interests, profits to specify additional fields that you need,
# besides the usual, opens, highs, lows, closes fields.
ohlc = feed.ohlc(
    instrument_id, start_time='2014-01-01', end_time='2016-12-31',
    periodicity=Periodicity.DAILY, with_open_interests=True, profits=True)

# Access fields from the result ohlc object.
print(len(ohlc.open_interests))



```
