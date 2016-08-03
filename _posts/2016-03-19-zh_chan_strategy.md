---
title: 缠论趋势策略
layout: post
category: ta
language: zh
---

本示例包括基于缠论来分辨趋势, 并且在python中调用R软件利用统计的时间序列ARIMA+GARCH模型来预测接下来趋势的幅度,并且利用其进行交易
(趋势跟随和趋势反转)的策略，回测以及查看结果的代码。


```python
from ctxalgolib.ta.piecewise_linear_regression.chan_piecewise_linear_regression import ChanPiecewiseLinearRegression
from ctxalgolib.charting.charts import ChartsWithReport
from ctxalgoctp.ctp.backtesting_utils import *
from ctxalgolib.data_feed.web_data_feed import WebFutureDataFeed
from ctxalgolib.ohlc.ohlc_generator import OhlcGeneratorConstants
from ctxalgolib.charting.charts import Charts
from ctxalgolib.data_feed.ohlc_based_data_feed import OhlcBasedDataFeed
from ctxalgolib.ohlc.periodicity import Periodicity
# from ctxalgoctp.ctp.backtesting_utils import *
from ctxalgolib.ta.piecewise_linear_regression.chan_piecewise_linear_regression import ChanPiecewiseLinearRegression
import rpy2.robjects as robjects

```

以下代码段展示了完整的交易策略


```python
class ChanStrategy(AbstractStrategy):
    """
    A single double moving average trend following strategy:
    1. If the fast moving average up-cross the slow moving average, change position to 1.
    2. If the fast moving average down-cross the slow moving average, change position to -1.
    That is, the position can be 0, -1 and 1.
    """
    def __init__(self, instrument_ids, strategy_period, parameters, base_folder,
                 periods=None, description=None, logger=None):
        AbstractStrategy.__init__(
            self, instrument_ids, parameters, base_folder, strategy_period=strategy_period,
            periods=periods, description=description, logger=logger)

        # here we use empty list because we use append. function
        self.pattern_type = []  # 记录分型点的类型，1为顶分型，-1为底分型
        self.pattern_time = []  # 记录分型点的时间
        self.pattern_value = []  # 记录点的数值，为顶分型去high值，为底分型去low值
        self.pattern_position = []  # 记录pattern的位置
        self.price_change = []  # store the price change of each trend in the list
        self.trading_days = []  # store the time range of each trend in the list
        # this 2-D list trend_info_set[][] contains the info of each trend
        # (the start pattern date,the end pattern date, the price change(percentage), the direction),
        # which will be used for the machine learning
        self.trend_info_set = []
        # self.predict_price_change = []  # used to store all the predictions of the price_change by ARIMA+GARCH model

    def count_effective_bars_up(self, previous_pattern_position, current_pattern_position, bars_solid_low,
                                bars_solid_high):

        # this function used to count the effective bars between previous pattern position and current pattern position
        # in the up trend. It should be larger than 3 effective bars

        initial_bars_between_patterns = current_pattern_position - (previous_pattern_position + 1)
        effective_bars_between_patterns_up = current_pattern_position - (previous_pattern_position + 1)

        if initial_bars_between_patterns >= 3:
            for i in range(previous_pattern_position,
                           current_pattern_position):  # loop over the bars between peak&&bottom pattern-pair
                case1_right_merging = bars_solid_high[i] <= bars_solid_high[i + 1] and bars_solid_low[i] >= \
                                                                                       bars_solid_low[
                                                                                           i + 1]  # 往右边包含, i+1包含i
                case2_left_merging = bars_solid_high[i] > bars_solid_high[i + 1] and bars_solid_low[i] < bars_solid_low[
                    i + 1]  # 往左边包含, i包含i+1

                # 不断把merge之后的bar算作第i+1个bar,下一轮即变成ith bar,再继续和下一个i+1比较,每merge一次即effective bar减一
                if case1_right_merging:
                    bars_solid_low[i + 1] = bars_solid_low[i]
                    effective_bars_between_patterns_up -= 1
                elif case2_left_merging:
                    bars_solid_high[i + 1] = bars_solid_high[i]
                    effective_bars_between_patterns_up -= 1

        # print('between previous bottom pattern position {} and current peak pattern position {} there are
        # {} effective bars \n '.format( previous_pattern_position, current_pattern_position,
        # effective_bars_between_patterns_up))
        return effective_bars_between_patterns_up

    def count_effective_bars_down(self, previous_pattern_position, current_pattern_position, bars_solid_low,
                                  bars_solid_high):

        # this function used to count the effective bars between previous pattern position and current pattern position
        # in the downtrend. It should be larger than 3 effective bars

        initial_bars_between_patterns = current_pattern_position - (previous_pattern_position + 1)
        effective_bars_between_patterns_down = current_pattern_position - (previous_pattern_position + 1)

        if initial_bars_between_patterns >= 3:
            for i in range(previous_pattern_position,
                           current_pattern_position):  # loop over the bars between pattern-pair
                case1_right_merging = (bars_solid_high[i] <= bars_solid_high[i + 1] and
                                       bars_solid_low[i] >= bars_solid_low[i + 1])  # 往右边包含, i+1包含i
                case2_left_merging = (bars_solid_high[i] > bars_solid_high[i + 1] and
                                      bars_solid_low[i] < bars_solid_low[i + 1])  # 往左边包含, i包含i+1

                # 不断把merge之后的bar算作第i+1个bar,下一轮即变成ith bar,再继续和下一个i+1比较,每merge一次即effective bar减一
                if case1_right_merging:
                    bars_solid_high[i + 1] = bars_solid_high[i]
                    effective_bars_between_patterns_down -= 1
                elif case2_left_merging:
                    bars_solid_low[i + 1] = bars_solid_low[i]
                    effective_bars_between_patterns_down -= 1

        # print('between previous peak pattern position {} and current bottom pattern position {} there are
        # {} effective bars \n '.format( previous_pattern_position, current_pattern_position,
        # effective_bars_between_patterns_down))
        return effective_bars_between_patterns_down

    def on_bar(self, instrument_id, bars, tick):

        # write the trends percentage change into the txt file, which will be used for the R tools
        chan = ChanPiecewiseLinearRegression()
        chan.regress_from_ohlc(self.ohlc())
        # print(chan.all_pattern_type)
        # Open the file for writing
        dataFile = open('trends_percentage_change_result.txt', 'w')
        # print len(chan.price_change)
        # Loop through each item in the list and write it to the output file.
        if len(chan.pattern_position) >= 1:
            print('ohlc length: {}  pattern position: {}   '
                  'price_change length: {}  pattern_type length: {}'
                  .format(len(self.ohlc().opens), chan.pattern_position[-1],
                          len(chan.price_change), len(chan.pattern_type)))
        for eachitem in chan.price_change:
            dataFile.write(str(eachitem) + '\n')
        # Close the output file
        dataFile.close()

        num_lines = sum(1 for line in open('trends_percentage_change_result.txt') if line.rstrip())
        # print num_lines
        # with open('trends_percentage_change_result.txt') as f:
        #    print len(f.readlines())
        if num_lines >= 300:  # to ensure that the in-sample data is enough, then apply the strategy
            robjects.r('''
                f1 <- function() {
                    pattern_length = length(readLines("~/Trading/Quant/ctx_project/ctxalgoctp/trends_percentage_change_result.txt"))
                    dataset = read.table("~/Trading/Quant/ctx_project/ctxalgoctp/trends_percentage_change_result.txt",sep = \'\t\',header=FALSE)
                    price = abs(dataset)
                    x=c(1:pattern_length)
                    for(i in 1:pattern_length)
                    {
                     x[i] = price[i,]
                    }
                    z = log(x)
                    z_diff = diff(z)

                    library(rugarch)
                    roll_pred = function(series,parameter=c(p,q,a,b),start=300)
                    {
                        pred = c(1:1)
                        arma.garch.norm = ugarchspec(mean.model=list(armaOrder=parameter[1:2]),variance.model=list(garchOrder=parameter[3:4]))
                        datafit = ugarchfit(data=series[1:length(series)],spec=arma.garch.norm)
                        forc = ugarchforecast(datafit,n.ahead=1)
                        pred = fitted(forc)[1]
                        pred
                    }
                    predict = roll_pred(z_diff,parameter=c(0,1,0,3),start=300)
                    ratio_pred = exp(predict)
                    x_pred = x[pattern_length]*ratio_pred
                }
                ''')
            func = robjects.r['f1']
            result = func()
            # self.context['prediction_list'] used to store all the predictions
            if 'prediction_list' not in self.context:
                self.context['prediction_list'] = []
            self.context['prediction_list'].append(result[0])

            # self.context['effective_prediction_list'] used to store the effective predictions
            if 'effective_prediction_list' not in self.context:
                self.context['effective_prediction_list'] = []

            if len(self.context['prediction_list']) >= 2:
                # when from peak pattern to bottom pattern,
                # fill in the effective_prediction_list with the prediction[-2]
                if chan.all_pattern_type[-2] == 1 and chan.all_pattern_type[-1] == -1 \
                        and (len(self.ohlc().opens) == chan.pattern_position[-1] + 2):
                    # Here use position to ensure that we choose the prediction right after the switch of the patterns
                    # If not, the effective prediction will be the uptrend prediction value from the current downtrend
                    # This self.context['prediction_list'][-2] element is downtrend prediction calculated from the
                    # past uptrend before the current peak pattern, so should multiply -1 to it,
                    # and this is the prediction we should use for the short position in the downtrend, not [-1]
                    self.context['prediction_list'][-2] *= -1
                    # the current result[0] is the next uptrend prediction calculated from the current recognized
                    # downtrend and then add it to the prediction list so can not use context['prediction_list'][-1]
                    self.context['effective_prediction_list'].append(self.context['prediction_list'][-2])
                # when from bottom pattern to peak pattern,
                # fill in the effective_prediction_list with the prediction[-2]
                if chan.all_pattern_type[-2] == -1 and chan.all_pattern_type[-1] == 1 \
                        and (len(self.ohlc().opens) == chan.pattern_position[-1] + 2):
                    # Same way, this chan.all_pattern_type[-2] element is uptrend prediction calculated from the
                    # downtrend before the bottom pattern
                    self.context['prediction_list'][-2] *= 1
                    # the current result[0] is the next downtrend prediction calculated from the current recognized
                    # uptrend and then add it to the prediction list so can not use context['prediction_list'][-1]
                    self.context['effective_prediction_list'].append(self.context['prediction_list'][-2])

                print('last trend type: {}  last trend change: {}  effective prediction length: {} '
                      .format(chan.pattern_type[-1], chan.price_change[-1], len(self.context['effective_prediction_list'])))
                if len(self.context['effective_prediction_list']) >= 1:
                    print('prediction_list: {}  effective_prediction_list: {}  \n'
                          .format(self.context['prediction_list'][-1], self.context['effective_prediction_list'][-1]))
            # self.ohlc() contains all the ohlcs which are generated till now, the ith ohlc bar info has the elements:
            # self.ohlc().closes[i]
            # self.ohlc().opens[i]
            # self.ohlc().lows[i]
            # self.ohlc().highs[i]
            # self.ohlc().volumes[i]
            # self.ohlc().dates[i]
            # self.ohlc() data has the components of open, close, high, low, volume, date. choose the high and low only
            # high of the solid body ith bar is put in bars_solid_high[i]
            # low of the solid body ith bar is put in bars_solid_low[i]

            bars_solid_high = [-1 for i in
                               range(len(self.ohlc().opens))]  # this is the way to define a new list with own values
            bars_solid_low = [-1 for i in range(len(self.ohlc().opens))]

            # define the solid bar
            for j in xrange(len(self.ohlc().opens)):
                # use the high and low of one bar to define the solid body of the bar(solid bar)
                # print('data_source {} opens  {} '.format( j, ohlc.opens[j]) )
                # print('data_source {} closes {} '.format( j, ohlc.closes[j]) )
                bars_solid_high[j] = self.ohlc().highs[j]
                bars_solid_low[j] = self.ohlc().lows[j]

                # search from the start along with the moving of ohlc
                # 返回值:
                # self.pattern_type 记录分型点的类型, 1为顶分型, -1为底分型
                # self.pattern_time 记录分型点的时间
                # self.pattern_value 记录点的数值, 为顶分型去high值, 为底分型去low值
                # append:(the number of the patterns)
                # self.pattern_type.append(+1)
                # self.pattern_time.append(ohlc.dates[i])
                # self.pattern_value.append(bars_solid_high[i])
                # self.pattern_position.append(i)
                # full values:(the number of bars)
                # temp_pattern_high = bars_solid_high[i]
                # temp_pattern_low = bars_solid_low[i]

            # 找出顶和底
            temp_pattern_position = 0  # 上一个顶或底的位置
            temp_pattern_high = 0  # 上一个顶的high值
            temp_pattern_low = 0  # 上一个底的low值
            temp_pattern_type = 0  # 上一个记录位置的类型, 1为顶分型，-1为底分型
            length = len(self.ohlc().opens)  # the length of the ohlcs used

            r0 = 2.0  # this is the profit/risk ratio

            for i in range(length - 1): # loop over all the ohlcs we have till now
                case1 = bars_solid_high[i - 1] < bars_solid_high[i] and bars_solid_high[i + 1] < bars_solid_high[
                    i]  # i is 准顶分型 quasi-peak pattern
                case2 = bars_solid_low[i - 1] > bars_solid_low[i] and bars_solid_low[i + 1] > bars_solid_low[
                    i]  # i is the 准底分型 quasi-bottom pattern

                if case1:  # 当前是一个准顶分型
                    if temp_pattern_type == 1:  # 上一个是顶分型
                        if bars_solid_high[i] <= temp_pattern_high:  # 当这个bar的high不高于前一个顶分型的high,跳过这个准顶分型
                            continue  # jump this one bar and search for the next bar in the loop
                        else:  # 当这个bar的high高于前一个顶分型的high,这个准顶分型变为counted顶分型,上一个顶分型删掉
                            temp_pattern_type = 1
                            del self.pattern_type[-1]
                            self.pattern_type.append(+1)
                            del self.pattern_time[-1]
                            self.pattern_time.append(self.ohlc().dates[i])
                            temp_pattern_high = bars_solid_high[i]
                            del self.pattern_value[-1]
                            self.pattern_value.append(bars_solid_high[i])
                            temp_pattern_position = i
                            del self.pattern_position[-1]
                            self.pattern_position.append(i)
                            if not self.has_pending_order():
                                if self.has_position():
                                    self.change_position_to(0)

                    elif temp_pattern_type == -1:  # 上一个是底分型
                        if bars_solid_high[i] <= temp_pattern_low:  # 如果上一个底分型的low还高于这一个准顶分型的high, 则跳过这根bar
                            continue
                        else:  # 如果其他情况,这个准顶分型变为counted顶分型,并且顶底分型之间至少隔四根合并后有效的bar
                            if i >= temp_pattern_position + 4:  # 顶和底之间至少3k线
                                # 判断两个分型之间"有效"(包含merge之后)bar的数目
                                effective_bars = self.count_effective_bars_up(temp_pattern_position, i, bars_solid_low,
                                                                              bars_solid_high)
                                if effective_bars >= 3:
                                    temp_pattern_type = 1
                                    self.pattern_type.append(+1)
                                    self.pattern_time.append(self.ohlc().dates[i])
                                    temp_pattern_high = bars_solid_high[i]
                                    self.pattern_value.append(bars_solid_high[i])
                                    temp_pattern_position = i
                                    self.pattern_position.append(i)
                                    # The change_position_to method handles open and close positions for you.
                                    # It will maintain one direction positions: either long or short, not both.
                                    # As an example, if you start with 0 position, and then do the following sequence:
                                    # change_position_to(1)   # open a long position
                                    # change_position_to(0)   # close the long position
                                    # change_position_to(-1)  # open a short position
                                    # change_position_to(1)   # close the short position, and open a long position.
                                    # So at the end, you have 1 long position.
                                    if len(self.pattern_type) >= 2 and len(self.context['effective_prediction_list']) >= 1:
                                        # after the peak pattern we find the first bottom pattern
                                        if self.pattern_type[-1] == 1 and self.pattern_type[-2] == -1:
                                            # s0 is the price change already happened between the newly found
                                            # peak pattern(pattern[-1]) and the bottom pattern(pattern[-2])
                                            s0 = (-self.pattern_value[-2]+self.pattern_value[-1])/self.pattern_value[-2]
                                            # r1 is the ratio which is the space between the predicted price change and
                                            # the happened price change s0, divided by the happened price change s0
                                            r1 = (self.context['prediction_list'][-1] - s0) / s0
                                            # then compare r1 with the r0, if r1 >= r0, it means still have room to
                                            # short or ignore this opportunity
                                            if r1 >= r0:
                                                if not self.has_pending_order():
                                                    if self.has_position():
                                                        self.change_position_to(0)  # sell the short position
                                                    else:
                                                        self.change_position_to(1)  # or open the long position
                                            else:
                                                if not self.has_pending_order():
                                                    if self.has_position():
                                                        self.change_position_to(0)  # sell the long position

                    else:  # find the first pattern(the value of temp_pattern_type not determined before it)
                        temp_pattern_type = 1  # 如果找到第一个准顶分型,即当做第一个顶分型
                        self.pattern_type.append(+1)
                        self.pattern_time.append(self.ohlc().dates[i])
                        temp_pattern_high = bars_solid_high[i]
                        self.pattern_value.append(bars_solid_high[i])
                        temp_pattern_position = i
                        self.pattern_position.append(i)

                elif case2:  # 当前是一个准底分型
                    if temp_pattern_type == -1:  # 上一个是底分型
                        if bars_solid_low[i] >= temp_pattern_low:  # 当这个bar的low不低于前一个顶分型的low,跳过这个准底分型
                            continue  # jump this one bar and search for the next bar in the loop
                        else:  # 当这个bar的low低于前一个底分型的low,这个准底分型变为counted顶分型
                            temp_pattern_type = -1
                            del self.pattern_type[-1]
                            self.pattern_type.append(-1)
                            del self.pattern_time[-1]
                            self.pattern_time.append(self.ohlc().dates[i])
                            temp_pattern_low = bars_solid_low[i]
                            del self.pattern_value[-1]
                            self.pattern_value.append(bars_solid_low[i])
                            temp_pattern_position = i
                            del self.pattern_position[-1]
                            self.pattern_position.append(i)

                            if not self.has_pending_order():
                                if self.has_position():
                                    self.change_position_to(0)

                    elif temp_pattern_type == 1:  # 上一个是顶分型
                        if bars_solid_low[i] >= temp_pattern_high:  # 如果上一个顶分型的high还低于这一个底分型的low, 则跳过这根bar
                            continue

                        else:  # 如果其他情况,这个准底分型变为counted底分型,并且顶底分型之间至少隔四根合并后有效的bar,
                            # 此为顶之后的第一个底分型, 需要根据prediction评估是否开空单
                            if i >= temp_pattern_position + 4:  # 顶和底之间至少3k线

                                effective_bars = self.count_effective_bars_down(temp_pattern_position, i, bars_solid_low,
                                                                                bars_solid_high)
                                if effective_bars >= 3:
                                    temp_pattern_type = -1
                                    self.pattern_type.append(-1)
                                    self.pattern_time.append(self.ohlc().dates[i])
                                    temp_pattern_low = bars_solid_high[i]
                                    self.pattern_value.append(bars_solid_low[i])
                                    temp_pattern_position = i
                                    self.pattern_position.append(i)
                                    if len(self.pattern_type) >= 2 and len(self.context['effective_prediction_list']) >= 1:
                                        # after the peak pattern we find the first bottom pattern
                                        if self.pattern_type[-1] == -1 and self.pattern_type[-2] == 1:
                                            # s0 is the price change already happened between the newly found
                                            # bottom pattern(pattern[-1]) and the peak pattern(pattern[-2])
                                            s0 = (self.pattern_value[-2]-self.pattern_value[-1])/self.pattern_value[-2]
                                            # r1 is the ratio which is the space between the predicted price change and
                                            # the happened price change s0, divided by the happened price change s0
                                            r1 = (- self.context['prediction_list'][-1] - s0) / s0
                                            # then compare r1 with the r0, if r1 >= r0, it means still have room to
                                            # short or ignore this opportunity
                                            if r1 >= r0:
                                                if not self.has_pending_order():
                                                    if self.has_position():
                                                        self.change_position_to(0)  # sell the long position
                                                    else:
                                                        self.change_position_to(-1)  # or open the short position
                                            else:
                                                if not self.has_pending_order():
                                                    if self.has_position():
                                                        self.change_position_to(0)  # sell the long position

                    else:  # find the first pattern(the value of temp_pattern_type not determined before it)
                        temp_pattern_type = -1
                        self.pattern_type.append(-1)
                        self.pattern_time.append(self.ohlc().dates[i])
                        temp_pattern_low = bars_solid_high[i]
                        self.pattern_value.append(bars_solid_low[i])
                        temp_pattern_position = i
                        self.pattern_position.append(i)
                else:
                    continue
```

现在我们对以上交易策略进行历史数据的回测。回测之后，我们生成一个网页，包括所有产生的交易记录。
你可以在浏览器中打开该页面查看具体的交易信息。


```python
def main():
    start_date = '2006-01-01'  # Backtesting start date.
    end_date = '2011-01-01'    # Backtesting end date.

    # The values in config will be used to instantiate the strategy objects by the backtest method.
    config = {
        'instrument_ids': ['cu99'],                      # We are trading this future instrument.
        'strategy_period': Periodicity.HOURLY,   # The ohlc bar granularity on which trading happens.
        'parameters': {
            'fast_ma_period': 8,   # The parameter for the fast moving average.
            'slow_ma_period': 15,  # The parameter for the slow moving average.
        }
    }
    start_time = datetime.now()

    report, data_source = backtest(ChanStrategy, config, start_date, end_date)
    end_time = datetime.now()
    print('Backtesting duration: ' + str(end_time - start_time))

    # Use charting facility to visualize trades.
    c = ChartsWithReport(report, data_source, folder=report.base_folder, open_in_browser=True)
    c.set_instrument(config['instrument_ids'][0])
    c.period(config['strategy_period'])

    c.ohlc()                                        # Draw ohlc bars.
    c.volume()                                      # Draw volumes bars.

    c.drawdowns(max_drawdown_color='pink')          # Visualize the max-drawdown period(s).
    c.instrument_positions()                        # Draw the current positions that we are holding.
    c.transactions()                                # Draw all the transactions (open and close of a position).
    c.balance()                                     # Draw the balance.
    c.show()                                        # Open the html page in browser.

if __name__ == '__main__':
    main()

```
