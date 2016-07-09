---
title: Registration for using real time market data
layout: post
category: market_broker
language: en
---

This example shows how to register the data sources to receive real time market data. The example includes
the way to register tick data, bar data (dominant contracts or non-dominant contracts).

```python
import zmq
import json
from bson import json_util
from ctxalgolib.data_feed.zeromq_feed_utils import ZeromqFeedUtils
from ctxalgolib.ohlc.periodicity import Periodicity
from ctxalgolib.proto.proto_gen.ctp_pb2 import DepthMarketData

# Create your own market data receiver (a zmq subscriber)
sub_context = zmq.Context()
subscriber = sub_context.socket(zmq.SUB)

# If you want to get the BAR market data, use the following way to generate filter string.
instrument_type = 'future'  # define the type of the instrument id, only support 'future' for now.
instrument_id = 'IF00'  #  For dominant contract, use 'IF00', second dominant contract 'IF01', and so on.
                        #  For general contracts, use name like 'IF1603', and so on.
periodicity = Periodicity.ONE_MINUTE  # the target Periodicity to register.
filter_str = ZeromqFeedUtils.get_time_based_ohlc_bar_filter(instrument_type, instrument_id, periodicity,
                                                            use_mid_ask_bid=True)

# If you want to get the TICK market data, use the following way to generate filter string.
instrument_type = "FUTURE"  # define the type of the instrument id, only support 'FUTURE' for now.
instrument_id = 'IF00'  #  For dominant contract, use 'IF00', second dominant contract 'IF01', and so on.
                        #  For general contracts, use name like 'IF1603', and so on.
filter_str = instrument_type + '#' + instrument_id

# If you want to register all instrument id from the market, you can simple leave the filter_str empty
filter_str = ''

# Based on the filter string defined above, we set the correct filter to receive the market data you want.
subscriber.setsockopt(zmq.SUBSCRIBE, filter_str)
subscriber.connect("localhost" + ':' + "27017")

while True:
    try:
        [filter_str, contents] = subscriber.recv_multipart(flags=zmq.NOBLOCK)
        # If you registered bar data.
        bar_data = json.loads(contents, object_hook=json_util.object_hook)
        # If you registered tick data.
        tick_data = DepthMarketData()
        tick_data.ParseFromString(contents)

        # ... You can apply any processing on the received bar data or tick data here in the loop.

    except zmq.ZMQError, e:
        if e.errno == zmq.EAGAIN:
            pass
        else:
            raise e

```
