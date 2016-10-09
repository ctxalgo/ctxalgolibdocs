---
title: Registration for using real time market data
layout: post
category: docs_market_broker
language: en
---

This example shows how to register the data sources to receive real time market data. The example includes
the way to register bar data (dominant contracts or non-dominant contracts).

```python
import zmq
from ctxalgolib.data_feed.zeromq_feed_utils import ZeromqFeedUtils
from ctxalgolib.ohlc.periodicity import Periodicity

# Create your own market data receiver (a zmq subscriber)
sub_context = zmq.Context()
subscriber = sub_context.socket(zmq.SUB)

# If you want to get the BAR market data, use the following way to generate filter string.
instrument_type = 'future'  # define the type of the instrument id, only support 'future' for now.
instrument_id = 'cu00'  #  For dominant contract, use 'IF00', second dominant contract 'IF01', and so on.
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

# Market data processor
# subscriber.connect("tcp://139.196.234.169" + ':' + "6557")


while True:
    try:
        content = subscriber.recv_multipart(flags=zmq.NOBLOCK)
        if len(content) == 2:
            filter_str, contents = content[0], content[1]
            bar_data = MarketDataBrokerUtils.parse_subscribed_data('bar', contents)
            print str(bar_data)
            assert bar_data['timestamp'].second == 0
        else:
            print(content)
            raise Exception('should not happen.')
        # ... You can apply any processing on the received bar data or tick data here in the loop.

    except zmq.ZMQError as e:
        if e.errno == zmq.EAGAIN:
            pass
        else:
            raise e



```
