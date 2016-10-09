---
title: Ohlc Generation Semantics
layout: post
category: docs_ohlc_generation
language: en
---

# Ohlc Generation Semantics

## When to generate an ohlc bar
We use one minute bar for illustration, bars of other periods follow the same semantics.

For an instrument, a bar is generated from tick data every minute during the instrument's traded market periods, with the following special rules:

 1. If there is no tick data within this minute, then use the last_price of the last received tick (which arrived before this minute starts) for this instrument as the open, high, low, close prices of the bar, and use 0 for the volume of the bar. So this bar will be a flat bar.
 2. If there is no tick data at all, then use the settlement price of the instrument from the previous trading day as the the open, high, low, close prices of the bar, and use 0 for the volume of the bar. The bar will be flat.
 3. If there is no settlement price for that instrument, which can happen on the instrument's first trading day, then no bar is generated.

## Settlement price retrieving and storing
Settlement price is sent from the CTP server through depth market data after market closes. So we receive those settlement prices and store it in database for using in the next trading day. When initializing an `OhlcGenerator` object, this settlement price (or None in case of no settlement price) is set into the object for bar generation.

## Time to call  `on_time`
`on_time` in `OhlcGenerator` is used to make sure bars are generated timely in the case of no tick. `on_time` is called in a market data broker which runs in a machine different from the CTP server. These two servers will have different local time. We sync our market data broker system local daily, and assume that the market data broker server's local time is very close to that of the CTP server, and when we call `on_time`, we delay by 2 seconds.
