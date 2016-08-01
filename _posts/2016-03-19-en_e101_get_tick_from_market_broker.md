---
title: Registration for using real time market data
layout: post
category: market_broker
language: en
---

This example shows how to register the data sources to receive real time market data. The example includes
the way to register tick data (dominant contracts or non-dominant contracts).

```python
import zmq
from ctxalgocore.market_data.market_data_broker_utils import MarketDataBrokerUtils

# Create your own market data receiver (a zmq subscriber)
sub_context = zmq.Context()
subscriber = sub_context.socket(zmq.SUB)

# If you want to get the TICK market data, use the following way to generate filter string.
instrument_type = "FUTURE"  # define the type of the instrument id, only support 'FUTURE' for now.
instrument_id = 'IF00'  #  For dominant contract, use 'IF00', second dominant contract 'IF01', and so on.
                        #  For general contracts, use name like 'IF1603', and so on.
filter_str = instrument_type + '#' + instrument_id

# If you want to register all instrument id from the market, you can simple leave the filter_str empty
# filter_str = ''

# Based on the filter string defined above, we set the correct filter to receive the market data you want.
subscriber.setsockopt(zmq.SUBSCRIBE, filter_str)

# To accept the tick data, need to register 6556 port.
subscriber.connect("tcp://127.0.0.1" + ':' + "6556")

while True:
    try:
        [filter_str, contents] = subscriber.recv_multipart(flags=zmq.NOBLOCK)
        tick_data = MarketDataBrokerUtils.parse_subscribed_data('tick', contents)
        print str(tick_data)
        # ... You can apply any processing on the received bar data or tick data here in the loop.

    except zmq.ZMQError, e:
        if e.errno == zmq.EAGAIN:
            pass
        else:
            raise e

```
