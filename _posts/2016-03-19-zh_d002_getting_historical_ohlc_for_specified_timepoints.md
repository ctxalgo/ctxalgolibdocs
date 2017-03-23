---
title: 获取给定时间点的历史ohlc数据
layout: post
category: docs_data
language: zh
---

本示例展示如何获取给定时间点的历史ohlc数据，例如获取一定时间段内每日10：00am的k线数据


```python
from ctxalgolib.ohlc.periodicity import Periodicity
from ctxalgolib.data_feed.web_mongo_data_feed import WebMongoDataFeed
from datetime import datetime

feed = WebMongoDataFeed()

# Use actual instrument ids such as cu1703 or you can use dominant instrument id such as cu00 for dominant instrument,
# cu01 for sub-dominant instrument and cu02 for sub-sub-dominant instrument.
instrument_id_list = ['cu00']

# Use options of the ohlc method, such as with_open_interests, profits to specify additional fields that you need,
# besides the usual, opens, highs, lows, closes fields.
ohlc = feed.trading_data_for_timepoints(instrument_id_list, datetime(2014, 1, 1), datetime(2016, 12, 31, 15),
                                        timepoints=['10:00', '14:00'], periodicity=Periodicity.ONE_MINUTE,
                                        with_open_interests=True, profits=True, use_mid_ask_bid=True)

for bars in ohlc:
    if bars.length > 0:
        print bars.to_csv_string()


```
