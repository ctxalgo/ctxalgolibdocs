---
title: Stock plate analysis
layout: post
category: docs_data
language: zh
---

```python
from datetime import datetime, date
import numpy as np
from ctxalgolib.data_feed.web_data_feed import WebStockDataFeed
from ctxalgolib.ohlc.periodicity import Periodicity
from ctxalgolib.data_feed.historical_data_fetcher import HistoricalLocalDataFeed
from ctxalgolib.tushare_utils.utils import *
from collections import defaultdict
```

获取所有的股票的日线数据。这里`start_time`指定获取数据的起始时间。`end_time`指定获取数据的结束时间。如果为None，则表示获取到数据库中最新的数据为止。


```python
stock_feed = WebStockDataFeed()
all_stocks = stock_feed.stocks()
all_stock_ids = all_stocks.keys()
folder = '/tmp/stock_ohlc_1d'
period = Periodicity.DAILY
feed = HistoricalLocalDataFeed(
    folder, sids=all_stock_ids, data_feed=stock_feed, period=Periodicity.DAILY,
    start_time=datetime(1990, 1, 1), end_time=None,
    profits=False, dominants=False, text_file=True, enforce_fetch=False)

```

获取股票板块数据。


```python
concepts = plates_of_stocks('concept', path='/tmp/concepts.csv', enforce_fetch=False)
```

把指定日期上上涨的股票提出出来。


```python
start_time = datetime(2016, 6, 1, 15)
end_time = datetime(2016, 6, 2, 15)
sids = []
returns = []
stock_count = len(feed.all_instruments())
for id_, sid in enumerate(feed.all_instruments()):
    print('{}/{} {}'.format(id_, stock_count, sid))
    ohlc = feed.ohlc(sid, start_time=start_time, end_time=end_time, periodicity=period)
    if ohlc.length == 2 and sid in concepts:
        rtn = (ohlc.closes[1]-ohlc.closes[0])/ohlc.closes[0]
        sids.append(sid)
        returns.append(rtn)

```

找到涨幅前100名的股票，并将它们归类到所属的板块中。


```python
returns = np.array(returns)
last_n = min(100, len(returns))

popular_plates = defaultdict(list)
for s_idx in np.argsort(returns)[-last_n:]:
    sid = sids[s_idx]
    if sid in concepts:
        for plate in concepts[sid]:
            popular_plates[plate].append((sid, returns[s_idx]))

p_plates = []
p_stocks = []
p_stock_len = []
for p, stocks in popular_plates.items():
    p_plates.append(p)
    p_stocks.append(stocks)
    p_stock_len.append(len(stocks))

for id_ in reversed(np.argsort(np.array(p_stock_len))):
    print(u'{}\t{}'.format(
        p_plates[id_],
        list(reversed([[x, round(y, 2)] for x, y in sorted(p_stocks[id_], key=lambda s: s[1])]))))




```
