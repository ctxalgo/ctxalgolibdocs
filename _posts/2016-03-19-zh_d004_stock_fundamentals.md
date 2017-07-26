---
title: 获取股票基本面数据
layout: post
category: docs_data
language: zh
---

本示例展示如何获取股票基本面数据，以及如何用这些数据进行因子运算。


```python
from datetime import date
from ctxalgolib.data_feed.data_feed_utils import StockDataFeed
from ctxalgolib.ohlc.trading_days import TradingDays


# Initialize a stock data feed to retrieve data.
feed = StockDataFeed()

# Print all supported financial fields.
all_fields = feed.all_financial_fields()
for k, v in all_fields.items():
    print(u'{}'.format(k))
    for f in v:
        print(u'\t{}.{}'.format(k, f))


# Get a datetime sequence, the fundamental data to be retrieved will be filled in to each of the timestamp.
days = TradingDays().get_days_between(date(2016, 1, 1), date(2016, 12, 31))

# Get a sequence for a single fundamental field.
for dt, value in zip(days, feed.fundamentals('sz000001', days, expr=u'财务.每股主营收入')):
    print(u'{}: {}'.format(dt, value))


# Get a sequence of a fundamental expression, in the following example, we calculate 市值比营运资金, a common factor
# to estimate a company's valuation, whose definition is
# (股本结构.股份总数 * 交易.closes_1d) / (财务.应收账款 + 财务.存货 - 财务.应付账款)
# Since the definition is a a mathematical expression with fundamental terms, we use the fundamental_expression method.
result = feed.fundamental_expression(
    'sz000001', u'(股本结构.股份总数 * 交易.closes_1d_前复权) / (财务.应收账款 + 财务.存货 - 财务.应付账款)', days)

for d, v in zip(result[0], result[1]):
    print(d, v)

```
