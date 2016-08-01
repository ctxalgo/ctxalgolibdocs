---
title: 使用实时市场行情数据的方法
layout: post
category: market_broker
language: zh
---

本示例展示如何注册market data broker 并获取实时市场ohlc bar行情数据（主力合约以及非主力合约）的方法。


```python
import zmq
from ctxalgocore.market_data.market_data_broker_utils import MarketDataBrokerUtils
from ctxalgolib.data_feed.zeromq_feed_utils import ZeromqFeedUtils
from ctxalgolib.ohlc.periodicity import Periodicity

# Create your own market data receiver (a zmq subscriber)
sub_context = zmq.Context()
subscriber = sub_context.socket(zmq.SUB)

# If you want to get the BAR market data, use the following way to generate filter string.
instrument_type = 'future'  # define the type of the instrument id, only support 'future' for now.
instrument_id = 'IF1609'  #  For dominant contract, use 'IF00', second dominant contract 'IF01', and so on.
                        #  For general contracts, use name like 'IF1603', and so on.
periodicity = Periodicity.ONE_MINUTE  # the target Periodicity to register.
filter_str = ZeromqFeedUtils.get_time_based_ohlc_bar_filter(instrument_type, instrument_id, periodicity,
                                                            use_mid_ask_bid=True)

# If you want to register all instrument id from the market, you can simple leave the filter_str empty
# filter_str = ''

# Based on the filter string defined above, we set the correct filter to receive the market data you want.
subscriber.setsockopt(zmq.SUBSCRIBE, filter_str)

# To accept the bar data, need to register 6557 port.
subscriber.connect("tcp://139.196.203.113" + ':' + "6557")

while True:
    try:
        [filter_str, contents] = subscriber.recv_multipart(flags=zmq.NOBLOCK)
        bar_data = MarketDataBrokerUtils.parse_subscribed_data('bar', contents)
        print str(bar_data)
        # ... You can apply any processing on the received bar data or tick data here in the loop.

    except zmq.ZMQError, e:
        if e.errno == zmq.EAGAIN:
            pass
        else:
            raise e

```
