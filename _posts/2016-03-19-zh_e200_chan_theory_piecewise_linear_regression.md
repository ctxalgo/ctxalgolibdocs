---
title: 缠论的分型策略
layout: post
category: docs_ta
language: zh
---

本示例展示如何利用缠论的分型理论来分辨品种cu99在周线上的每一个趋势,并且将其表现在周线图上。
还包括每一个分辨出的趋势的信息,包括(起始日期, 结束日期, 涨跌幅度, 趋势方向)


```python
from ctxalgolib.data_feed.web_data_feed import WebFutureDataFeed
from ctxalgolib.ohlc.ohlc_generator import OhlcGeneratorConstants
from ctxalgolib.charting.charts import Charts
from ctxalgolib.data_feed.ohlc_based_data_feed import OhlcBasedDataFeed
from ctxalgolib.ohlc.periodicity import Periodicity
from ctxalgoctp.ctp.backtesting_utils import *
from ctxalgolib.ta.piecewise_linear_regression.chan_piecewise_linear_regression import ChanPiecewiseLinearRegression


def main():
    data_feed = WebFutureDataFeed()
    ohlc = data_feed.ohlc('cu99', start_time='2012-01-01', end_time='2016-01-01', periodicity=Periodicity.WEEKLY)

    """
    [en]
    ## 1. Search for pattern by using the ChanPiecewiseLinearRegression class instance
    Get all the info of the recognized patterns, the period is WEEKLY.
    pattern_position: includes all the pattern positions in all the bars
    pattern_type: includes the types of all the patterns, +1 means peak pattern, -1 means bottom pattern
    pattern_time: includes the dates of all the patterns

    [zh]
    ## 1. 利用ChanPiecewiseLinearRegression来搜索分型
    得到所有分型的信息, 周期是周线
    pattern_position: 包括了所有分型的位置
    pattern_type: 包括了所有分型的类型, +1表示顶分型, -1表示底分型
    pattern_time: 包括了所有分型的日期和时间
    """

    chan = ChanPiecewiseLinearRegression()
    section = chan.regress_from_ohlc(ohlc)

    # The following is drawing the recognized weekly patterns on the datafeed chart
    # Construct some ad-hoc time series based on the ohlc. series_t stores x-axis points, series_d stores y-axis points.
    series_t = chan.pattern_time
    series_d = []
    for i in range(len(chan.pattern_position)):
        series_d.append(section[i][1])
        # if chan.pattern_type[i] == 1:
        #     series_d.append(ohlc.highs[chan.pattern_position[i]])
        # elif chan.pattern_type[i] == -1:
        #     series_d.append(ohlc.lows[chan.pattern_position[i]])

    # Construct a data feed for charting.
    feed = OhlcBasedDataFeed({
        OhlcGeneratorConstants.time_based: {
            'cu99': {
                Periodicity.DAILY: ohlc
            }
        }
    })

    # Chart
    c = Charts(data_feed=feed)
    c.set_instrument('cu99')

    # Need to tell the chart component under which period the ohlc period is, default is DAILY
    # Or it will connect the empty bar,
    c.period(Periodicity.WEEKLY)

    # Draw ohlc.
    c.height(k=500)  # Set the height of the K-bar area.
    c.ohlc(position='k')  # Draw ohlc.
    c.volume()  # Draw volume.

    # Draw the ad-hoc time series.
    c.series(series_t, series_d, position='k', color='red', thick=1)  # use line to connect the points

    # Show the chart in browser.
    file_path = c.show()
    print(file_path)

    """
    [en]
    ## 2. Obtain the info of each trend by calling the list variable trend_info_set
    trend_info_set: this 2-D list trend_info[][] contains the following info of each trend
                (the start pattern date, the end pattern date, the price change(percentage), the direction),
                which will be used for the machine learning

                The price change(percentage) is: (previous pattern price - current price) / previous price,
                downtrend is negative, uptrend is positive

                When connect from the peak pattern(pattern_type = 1) to the bottom pattern(pattern_type = -1),
                the direction of the trend is down trend, its direction value is -1

                When connect from the peak pattern(pattern_type = -1) to the bottom pattern(pattern_type = 1),
                the direction of the trend is up trend, its direction value is +1
    [zh]
    ## 2. 通过列表变量trend_info_set获得所有趋势的信息
    trend_info_set: 这个二维的列表包括了以下的信息
                    (此趋势起始分型的日期, 此趋势结束分型的日期, 价格变化(百分比), 方向), 整个组合用于机器学习

                    价格变化(百分比)是: (前一个分型的价格-当下分型的价格)/前一个分型的价格, 下降趋势为负值, 上升趋势为正值

                    当连接顶分型(pattern_type = 1)和底分型(pattern_type = -1)的时候, 趋势的方向是下降, 值为-1

                    当连接底分型(pattern_type = -1)和顶分型(pattern_type = 1)的时候, 趋势的方向是上升, 值为1
    """

    trend_info = chan.trend_info_set


if __name__ == '__main__':
    main()

```
