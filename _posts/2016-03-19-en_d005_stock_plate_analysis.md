---
title: Stock plate movement analysis using both CTX and tushare data.
layout: post
category: docs_data
language: en
---

[zh]:
---
title: 通过同时使用CTX和tushare数据来分析热门板块。
layout: post
category: docs_data
language: en
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

Prepare ohlc data for all stocks.

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

Get stock plates.

```python
concepts = plates_of_stocks('concept', path='/tmp/concepts.csv', enforce_fetch=False)
```

Get the stocks with return on the specified date.

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

Find the 100 top stocks with the highest returns and find their plates.

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
