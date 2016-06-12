---
title: Simple trend following strategy using two moving averages
layout: post
category: en
---

This example shows how to write trading strategies for a single instrument. The example includes
a double moving average trend following strategy, the code to perform backtesting of the strategy, and
the code to investigate trades through generated charts.

```python
from ctxalgolib.ta.online_indicators.moving_averages import MovingAveragesIndicator
from ctxalgolib.charting.charts import ChartsWithReport
from ctxalgolib.ta.cross import cross_direction
from ctxalgoctp.ctp.backtesting_utils import *
```

The following listing shows the strategy class.

```python
class TrendFollowingStrategy(AbstractStrategy):
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

        # Setup the moving average calculators. This strategy needs two moving averages, a fast one and a slow one.
        # The fast moving average use the fast_ma_period as its parameter, the slow moving average uses slow_ma_period.
        # Note that you can directly reference the parameters, such as fast_ma_period through self.fast_ma_period.
        # You can also use the notation self.parameters.fast_ma_period to reference the parameter.
        self.ma_ind1 = MovingAveragesIndicator(period=self.fast_ma_period)
        self.ma_ind2 = MovingAveragesIndicator(period=self.slow_ma_period)

    def on_bar(self, instrument_id, bars, tick):
        # Check if there is enough ohlc bars to calculate the slow moving average because it needs
        # more bars than the fast moving average.
        if self.ohlc().length >= self.slow_ma_period:
            # Calculate the two moving averages.
            ma_fast = self.ma_ind1.calculate(self.ohlc())
            ma_slow = self.ma_ind2.calculate(self.ohlc())

            # Check if fast moving average up-crosses or down-crosses slow moving average.
            # cross_direction return 1 if ma_fast up-crosses ma_slow, -1 if ma_fast down-crosses ma_fast,
            # and 0 otherwise. The strategy uses this signal to change its position of the traded instrument.
            signal = cross_direction(ma_fast, ma_slow)
            if signal != 0:
                # The change_position_to method handles open and close positions for you.
                # It will maintain one direction positions: either long or short, not both.
                # As an example, if you start with 0 position, and then do the following sequence:
                # change_position_to(1)   # open a long position
                # change_position_to(0)   # close the long position
                # change_position_to(-1)  # open a short position
                # change_position_to(1)   # close the short position, and open a long position.
                # So at the end, you have 1 long position.
                self.change_position_to(signal)
```

Now, we create configurations for backtesting the strategy. After backtesting, we generate a chart in form of
HTML page to view all the traded. You can review the trades inside a browser.

```python
def main():
    start_date = '2014-01-01'  # Backtesting start date.
    end_date = '2014-12-31'    # Backtesting end date.

    # The values in config will be used to instantiate the strategy objects by the backtest method.
    config = {
        'instrument_ids': ['IF99'],                      # We are trading this future instrument.
        'strategy_period': Periodicity.FIFTEEN_MINUTE,   # The ohlc bar granularity on which trading happens.
        'parameters': {
            'fast_ma_period': 8,   # The parameter for the fast moving average.
            'slow_ma_period': 15,  # The parameter for the slow moving average.
        }
    }

    report, data_source = backtest(TrendFollowingStrategy, config, start_date, end_date)

    # Use charting facility to visualize trades.
    c = ChartsWithReport(report, data_source, folder=report.base_folder, open_in_browser=True)
    c.set_instrument(config['instrument_ids'][0])
    c.period(config['strategy_period'])

    c.ohlc()                                        # Draw ohlc bars.
    c.volume()                                      # Draw volumes bars.

    c.ma(config['parameters']['fast_ma_period'])    # Draw the fast moving average.
    c.ma(config['parameters']['slow_ma_period'])    # Draw the slow moving average.

    c.drawdowns(max_drawdown_color='pink')          # Visualize the max-drawdown period(s).
    c.instrument_positions()                        # Draw the current positions that we are holding.
    c.transactions()                                # Draw all the transactions (open and close of a position).
    c.balance()                                     # Draw the balance.
    c.show()                                        # Open the html page in browser.


if __name__ == '__main__':
    main()

```
