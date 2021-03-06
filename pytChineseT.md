#pyalgotrade 中文学习手册 
原文：http://gbeced.github.io/pyalgotrade/docs/v0.17/html/index.html
翻译by迪卡斯
感谢 年糕和电老虎的帮助。
##前言
pyAlgotrade 是一个 基于事件驱动的算法交易 的Python库。它支持：
+ 通过CSV文件和 历史数据 进行回测
+ 模拟交易 用的是Xignite 0（Xignite.com	）和 Bitstamp( Bitstamp.com)的数据
+ 真实交易通过 Birstamp

它还易于用多台电脑 优化策略。


PyAlgotrade 依赖于 ：
+ [ Numpy and  Scipy ](%28http://numpy.scipy.org/)科学计算
+  [pytz ](http://pytz.sourceforge.net/)时区
+ [matplotlib ](http://matplotlib.sourceforge.net/) 绘图库
+  [ws4py](https://github.com/Lawouach/WebSocket-for-Python) WebSocket 的Client和Server端，用来支持 Bitstamp
+ [ tornado](%28http://www.tornadoweb.org/en/stable/)Facebook的开源框架，直接内嵌了HTTP服务器。用来支持Bitstamp
+  [tweepy](https://github.com/tweepy/tweepy)  Python的 推特 API库。

你需要安装以上这些包来使用pyalgotrade.

你可以用pip像这样安装 PyAlgotrade :

    pip install pyalgotrade

##教程
本教程的目的在于给你一个 PyAlgotrade(以下简写成PAt)的简短介绍。如前言所诉，PAt 的目的在于帮你回测证券交易策略。假设你想把一个交易策略通过历史数据评估它。看看它的表现如何。那么PAt容许你用最少的工作量来时实现你的想法。

在这之前，我要感谢巴勃罗·豪尔曾帮我审查初步设计和文档。

这个教程是在UNIX环境下开发的 ，但在window环境中也可以直接上代码。

PAt 有六大主要部件：
+ Strategies
+ Feeds
+ Brokers
+ DataSeries
+ Technicals
+ Optimizer

Strategies
	这些类用来实现你的交易逻辑，啥时买，啥时卖等等。
Feeds
这些数据提供抽象。 例如，你将用一个CSV feed，从CSV文件中加载条形图推送数据给策略。 Feeds 不限于 条形图 格式，比如 一个推特 feed 允许将 推特事件纳入 交易决策
Brokers
Brokers负责执行命令。 和经纪人一样。
DataSeries
用来抽象管理时间序列数据
Technicals
一组用来控制着DataSeries计算的过滤器。如 SMA（简单移动平均），RSI （相对强弱指标）等。这些过滤器等效为DataSeries修饰符。
Optimizer
这组类能让你把回测分配到不同电脑之间，或者不同进程运行在同一台电脑上，或者两者的结合。它们使水平拓展更容易。

话虽如此，首先我们想回测，需要一些数据。让我们用 甲骨文公司在2000年的股价。我们将用下面这条。

    python -c "from pyalgotrade.tools import yahoofinance; yahoofinance.download_daily_bars('orcl', 2000, 'orcl-2000.csv')"

pyalgotrade.tools.yahoofinance 包从Yahoo！ Finance中下载CSV格式数据。 CSV文件看起来像这样：
> Date,Open,High,Low,Close,Volume,Adj Close
> 2000-12-29,30.87,31.31,28.69,29.06,31655500,28.35
> 2000-12-28,30.56,31.12,30.37,31.06,25055600,30.30
> 2000-12-27,30.37,31.06,29.37,30.69,26441700,29.94
> .
> .
> 2000-01-04,115.50,118.62,105.00,107.69,116850000,26.26
> 2000-01-03,124.62,125.19,111.62,118.12,98122000,28.81

来开始写一个简单的策略。打印收盘价 ，像这样：

    from pyalgotrade import strategy
    from pyalgotrade.barfeed import yahoofeed

    class MyStrategy(strategy.BacktestingStrategy):
        def __init__(self, feed, instrument):
            strategy.BacktestingStrategy.__init__(self, feed)
            self.__instrument = instrument

        def onBars(self, bars):
            bar = bars[self.__instrument]
            self.info(bar.getClose())

    # Load the yahoo feed from the CSV file
    feed = yahoofeed.Feed()
    feed.addBarsFromCSV("orcl", "orcl-2000.csv")

    # Evaluate the strategy with the feed's bars.
    myStrategy = MyStrategy(feed, "orcl")
    myStrategy.run()

这段代码一共做了三件事：
1. 说明一个新策略。只有一个方法来定义 *onBars*调用feed中每一条数据。
2. 从CSV文件中加载feed
3. 运行策略 和 条形图。条形图是feed提供的。

如果你运行这个脚本应该看到收盘价:
> 2000-01-03 00:00:00 strategy [INFO] 118.12
> 2000-01-04 00:00:00 strategy [INFO] 107.69
> 2000-01-05 00:00:00 strategy [INFO] 102.0
> .
> .
> .
> 2000-12-27 00:00:00 strategy [INFO] 30.69
> 2000-12-28 00:00:00 strategy [INFO] 31.06
> 2000-12-29 00:00:00 strategy [INFO] 29.06

让我们继续。 写一个打印收盘移动平均价的策略。来说明如何使用technicals。

    from pyalgotrade import strategy 
    from pyalgotrade.barfeed import yahoofeed
    from pyalgotrade.technical import ma


    class MyStrategy(strategy.BacktestingStrategy):
        def __init__(self, feed, instrument):
            strategy.BacktestingStrategy.__init__(self, feed)
            # We want a 15 period SMA over the closing prices.
            self.__sma = ma.SMA(feed[instrument].getCloseDataSeries(), 15)
            self.__instrument = instrument

        def onBars(self, bars):
            bar = bars[self.__instrument]
            self.info("%s %s" % (bar.getClose(), self.__sma[-1]))

    # Load the yahoo feed from the CSV file
    feed = yahoofeed.Feed()
    feed.addBarsFromCSV("orcl", "orcl-2000.csv")

    # Evaluate the strategy with the feed's bars.
    myStrategy = MyStrategy(feed, "orcl")
    myStrategy.run() 

这个非常类似于前面的例子。除了：
1. 我们在收盘价数据序列上初始化一个SMA过滤器。
2. 我们随着收盘价打印当前的移动平均价。

如果你运行这个脚本，就能看到收盘价和相应的SMA指标。但在这种情况下，第一个14 SMA的值为None。
> 2000-01-03 00:00:00 strategy [INFO] 118.12 None
2000-01-04 00:00:00 strategy [INFO] 107.69 None
2000-01-05 00:00:00 strategy [INFO] 102.0 None
2000-01-06 00:00:00 strategy [INFO] 96.0 None
2000-01-07 00:00:00 strategy [INFO] 103.37 None
2000-01-10 00:00:00 strategy [INFO] 115.75 None
2000-01-11 00:00:00 strategy [INFO] 112.37 None
2000-01-12 00:00:00 strategy [INFO] 105.62 None
2000-01-13 00:00:00 strategy [INFO] 105.06 None
2000-01-14 00:00:00 strategy [INFO] 106.81 None
2000-01-18 00:00:00 strategy [INFO] 111.25 None
2000-01-19 00:00:00 strategy [INFO] 57.13 None
2000-01-20 00:00:00 strategy [INFO] 59.25 None
2000-01-21 00:00:00 strategy [INFO] 59.69 None
2000-01-24 00:00:00 strategy [INFO] 54.19 94.2866666667
2000-01-25 00:00:00 strategy [INFO] 56.44 90.1746666667
.
.
.
2000-12-27 00:00:00 strategy [INFO] 30.69 29.9866666667

在给定的时间内无法算出数值时，所有的technicals都会返回None值。
technicals 一个重要的特性就是联合。因为他们 同样建模为 DataSeries。例如，很容易的做到在收盘价上的RSI来得到一个移动平均SMA：

    from pyalgotrade import strategy
    from pyalgotrade.barfeed import yahoofeed
    from pyalgotrade.technical import ma
    from pyalgotrade.technical import rsi


    class MyStrategy(strategy.BacktestingStrategy):
        def __init__(self, feed, instrument):
            strategy.BacktestingStrategy.__init__(self, feed)
            self.__rsi = rsi.RSI(feed[instrument].getCloseDataSeries(), 14)
            self.__sma = ma.SMA(self.__rsi, 15)
            self.__instrument = instrument

        def onBars(self, bars):
            bar = bars[self.__instrument]
            self.info("%s %s %s" % (bar.getClose(), self.__rsi[-1], self.__sma[-1]))

    # Load the yahoo feed from the CSV file
    feed = yahoofeed.Feed()
    feed.addBarsFromCSV("orcl", "orcl-2000.csv")

    # Evaluate the strategy with the feed's bars.
    myStrategy = MyStrategy(feed, "orcl")
    myStrategy.run()
如果你运行这个脚本，你会看到屏幕上出现一堆点：
+ 第一行14 RSI值为 None。那是因为我们需要至少15个值才能到到一个RSI指标。
+ 第一行28 RSI值为 None。那是因为第一行14RSI的值为None，第十五行则是第一个RSI不为None的值。这个值被SMA过滤器 接收。我们只有当我们有15个不是None值时，才能计算出SMA(15)。

###交易
让我们继续，写一个简单的策略。这次模拟实际的交易。想法非常简单：
+ 如果 收盘价调整到SMA（15）之上，我们挂一市价买单。
+ 有多头仓位的情况下，如果收盘价调整到SMA（15）之下。那我们挂一个市价卖单。

		from pyalgotrade import strategy
		from pyalgotrade.barfeed import yahoofeed
		from pyalgotrade.technical import ma
    
        class MyStrategy(strategy.BacktestingStrategy):
            def __init__(self, feed, instrument, smaPeriod):
               strategy.BacktestingStrategy.__init__(self, feed, 1000)
               self.__position = None
               self.__instrument = instrument
               # We'll use adjusted close values instead of regular close values.
               self.setUseAdjustedValues(True)
               self.__sma = ma.SMA(feed[instrument].getPriceDataSeries(), smaPeriod)

            def onEnterOk(self, position):
                execInfo = position.getEntryOrder().getExecutionInfo()
                self.info("BUY at $%.2f" % (execInfo.getPrice()))

            def onEnterCanceled(self, position):
                self.__position = None

            def onExitOk(self, position):
                execInfo = position.getExitOrder().getExecutionInfo()
                self.info("SELL at $%.2f" % (execInfo.getPrice()))
                self.__position = None

            def onExitCanceled(self, position):
                # If the exit was canceled, re-submit it.
                self.__position.exitMarket()

            def onBars(self, bars):
                # Wait for enough bars to be available to calculate a SMA.
                if self.__sma[-1] is None:
                    return

                bar = bars[self.__instrument]
                # If a position was not opened, check if we should enter a long position.
		        if self.__position is None:
	                if bar.getPrice() > self.__sma[-1]:
		                # Enter a buy market order for 10 shares. The order is good till canceled.
		                self.__position = self.enterLong(self.__instrument, 10, True)
		        # Check if we have to exit the position.
				        elif bar.getPrice() < self.__sma[-1] and not self.__position.exitActive():
			            self.__position.exitMarket()


			def run_strategy(smaPeriod):
		        # Load the yahoo feed from the CSV file
			    feed = yahoofeed.Feed()
			    feed.addBarsFromCSV("orcl", "orcl-2000.csv")

			    # Evaluate the strategy with the feed.
			    myStrategy = MyStrategy(feed, "orcl", smaPeriod)
			    myStrategy.run()
			    print "Final portfolio value: $%.2f" % myStrategy.getBroker().getEquity()
	     run_strategy(15)

运行脚本后 ，会看到这样的结果：
> 2000-01-26 00:00:00 strategy [INFO] BUY at $27.26
2000-01-28 00:00:00 strategy [INFO] SELL at $24.74
2000-02-03 00:00:00 strategy [INFO] BUY at $26.60
2000-02-22 00:00:00 strategy [INFO] SELL at $28.40
2000-02-23 00:00:00 strategy [INFO] BUY at $28.91
2000-03-31 00:00:00 strategy [INFO] SELL at $38.51
2000-04-07 00:00:00 strategy [INFO] BUY at $40.19
2000-04-12 00:00:00 strategy [INFO] SELL at $37.44
2000-04-19 00:00:00 strategy [INFO] BUY at $37.76
2000-04-20 00:00:00 strategy [INFO] SELL at $35.45
2000-04-28 00:00:00 strategy [INFO] BUY at $37.70
2000-05-05 00:00:00 strategy [INFO] SELL at $35.54
2000-05-08 00:00:00 strategy [INFO] BUY at $36.17
2000-05-09 00:00:00 strategy [INFO] SELL at $35.39
2000-05-16 00:00:00 strategy [INFO] BUY at $37.28
2000-05-19 00:00:00 strategy [INFO] SELL at $34.58
2000-05-31 00:00:00 strategy [INFO] BUY at $35.18
2000-06-23 00:00:00 strategy [INFO] SELL at $38.81
2000-06-27 00:00:00 strategy [INFO] BUY at $39.56
2000-06-28 00:00:00 strategy [INFO] SELL at $39.42
2000-06-29 00:00:00 strategy [INFO] BUY at $39.41
2000-06-30 00:00:00 strategy [INFO] SELL at $38.60
2000-07-03 00:00:00 strategy [INFO] BUY at $38.96
2000-07-05 00:00:00 strategy [INFO] SELL at $36.89
2000-07-21 00:00:00 strategy [INFO] BUY at $37.19
2000-07-24 00:00:00 strategy [INFO] SELL at $37.04
2000-07-26 00:00:00 strategy [INFO] BUY at $35.93
2000-07-28 00:00:00 strategy [INFO] SELL at $36.08
2000-08-01 00:00:00 strategy [INFO] BUY at $36.11
2000-08-02 00:00:00 strategy [INFO] SELL at $35.06
2000-08-04 00:00:00 strategy [INFO] BUY at $37.61
2000-09-11 00:00:00 strategy [INFO] SELL at $41.34
2000-09-29 00:00:00 strategy [INFO] BUY at $39.07
2000-10-02 00:00:00 strategy [INFO] SELL at $38.30
2000-10-20 00:00:00 strategy [INFO] BUY at $34.71
2000-10-31 00:00:00 strategy [INFO] SELL at $31.34
2000-11-20 00:00:00 strategy [INFO] BUY at $23.35
2000-11-21 00:00:00 strategy [INFO] SELL at $23.83
2000-12-01 00:00:00 strategy [INFO] BUY at $25.33
2000-12-21 00:00:00 strategy [INFO] SELL at $26.72
2000-12-22 00:00:00 strategy [INFO] BUY at $29.17

> Final portfolio value: $979.44

但我们使用周期为30的SMA来计算的结果比使用SMA（15）的情况更好吗？我们可以这样试一下：

    for i in range(10, 30):
        run_strategy(i)
于是，我们得到更好的结果， SMA（20）：

    Final portfolio value: $1075.38
当我们只有一组参数时，这样做还不错。但如果我们要测试的策略有多组参数。串行绝不会随着策略而变得更复杂

###优化
开始 接触优化器部件。一个简单的想法：
+ 一个服务系统负责：
	+ 提供条形图 用来运行策略。
	+ 提供相应的参数运行策略。
	+  从每个工作部件那里记录策略运算的结果。
+ 很多个工作部件负责：
	+ 运行策略的条数据和服务器提供的参数
为了说明这一点：我们将使用一个名为[RSI2](http://stockcharts.com/school/doku.php?id=chart_school:trading_strategies:rsi2)的策略，该策略需要以下参数：
+ 用SMA周期来趋势识别。我们用 entrySMA表示。 SMA介于150到250之间。
+ 用更小SMA周期 来确认退出点。我们用 exitSMA表示 。SMA介于 15到20之间。
+ 用RSI周期来看多/看空。用 risPeroid命名同时RSI介于2到10之间
+ 当RSI	超卖买入。用 overSoldThrehold命名。RSI介于5到25之间
+ 当RSI	超买卖出。用 overBoughtThreshold命名  RSI介于75到95

这些参数，可以构成4409559不同的组合。

测试含这些参数的策略用了0.16秒。如果我执行并评估所有的组合找到最佳的一组参数，需要约8.5天。这个时间非常长,但如果我能获得10台8核电脑来做这项工作的总时间就只需约2.5小时。

简言之，我们需要并行。

让我们先下载3年的 道琼斯工业指数的日线数据：

    python -c "from pyalgotrade.tools import yahoofinance; yahoofinance.download_daily_bars('dia', 2009, 'dia-2009.csv')"
	python -c "from pyalgotrade.tools import yahoofinance; yahoofinance.download_daily_bars('dia', 2010, 'dia-2010.csv')"
	python -c "from pyalgotrade.tools import yahoofinance; yahoofinance.download_daily_bars('dia', 2011, 'dia-2011.csv')"

编写这段代码，并用rsi2.py保存：

    from pyalgotrade import strategy
    from pyalgotrade.technical import ma
    from pyalgotrade.technical import rsi
    from pyalgotrade.technical import cross

    class RSI2(strategy.BacktestingStrategy):
        def __init__(self, feed, instrument, entrySMA, exitSMA, rsiPeriod, overBoughtThreshold, overSoldThreshold):
            strategy.BacktestingStrategy.__init__(self, feed)
            self.__instrument = instrument
            # We'll use adjusted close values, if available, instead of regular close values.
            if feed.barsHaveAdjClose():
                self.setUseAdjustedValues(True)
            self.__priceDS = feed[instrument].getPriceDataSeries()
            self.__entrySMA = ma.SMA(self.__priceDS, entrySMA)
            self.__exitSMA = ma.SMA(self.__priceDS, exitSMA)
            self.__rsi = rsi.RSI(self.__priceDS, rsiPeriod)
            self.__overBoughtThreshold = overBoughtThreshold
            self.__overSoldThreshold = overSoldThreshold
            self.__longPos = None
            self.__shortPos = None

        def getEntrySMA(self):
            return self.__entrySMA

        def getExitSMA(self):
            return self.__exitSMA

        def getRSI(self):
            return self.__rsi

        def onEnterCanceled(self, position):
            if self.__longPos == position:
                self.__longPos = None
            elif self.__shortPos == position:
                self.__shortPos = None
            else:
                assert(False)

        def onExitOk(self, position):
            if self.__longPos == position:
                self.__longPos = None
            elif self.__shortPos == position:
                self.__shortPos = None
            else:
                assert(False)

        def onExitCanceled(self, position):
            # If the exit was canceled, re-submit it.
            position.exitMarket()

        def onBars(self, bars):
            # Wait for enough bars to be available to calculate SMA and RSI.
            if self.__exitSMA[-1] is None or self.__entrySMA[-1] is None or self.__rsi[-1] is None:
                return

            bar = bars[self.__instrument]
            if self.__longPos is not None:
                if self.exitLongSignal():
                    self.__longPos.exitMarket()
            elif self.__shortPos is not None:
                if self.exitShortSignal():
                    self.__shortPos.exitMarket()
            else:
                if self.enterLongSignal(bar):
                    shares = int(self.getBroker().getCash() * 0.9 / bars[self.__instrument].getPrice())
                    self.__longPos = self.enterLong(self.__instrument, shares, True)
                elif self.enterShortSignal(bar):
                    shares = int(self.getBroker().getCash() * 0.9 / bars[self.__instrument].getPrice())
                    self.__shortPos = self.enterShort(self.__instrument, shares, True)

        def enterLongSignal(self, bar):
            return bar.getPrice() > self.__entrySMA[-1] and self.__rsi[-1] <= self.__overSoldThreshold

        def exitLongSignal(self):
            return cross.cross_above(self.__priceDS, self.__exitSMA) and not self.__longPos.exitActive()

        def enterShortSignal(self, bar):
            return bar.getPrice() < self.__entrySMA[-1] and self.__rsi[-1] >= self.__overBoughtThreshold

        def exitShortSignal(self):
            return cross.cross_below(self.__priceDS, self.__exitSMA) and not self.__shortPos.exitActive()

下面是服务脚本：

    import itertools
    from pyalgotrade.barfeed import yahoofeed
    from pyalgotrade.optimizer import server


    def parameters_generator():
        instrument = ["dia"]
        entrySMA = range(150, 251)
        exitSMA = range(5, 16)
        rsiPeriod = range(2, 11)
        overBoughtThreshold = range(75, 96)
        overSoldThreshold = range(5, 26)
        return itertools.product(instrument, entrySMA, exitSMA, rsiPeriod, overBoughtThreshold, overSoldThreshold)

    # The if __name__ == '__main__' part is necessary if running on Windows.
    if __name__ == '__main__':
        # Load the feed from the CSV files.
        feed = yahoofeed.Feed()
        feed.addBarsFromCSV("dia", "dia-2009.csv")
        feed.addBarsFromCSV("dia", "dia-2010.csv")
        feed.addBarsFromCSV("dia", "dia-2011.csv")

        # Run the server.
        server.serve(feed, parameters_generator(), "localhost", 5000)

这段服务代码做了3件事：
1. 声明一个函数，产生不同的参数组合
2. 通过我们下载的CSV文件 加载 feed
3. 使用    pyalgotrade.optimizer.local 并行运算策略找出最好结果

但你运行脚本时，你会看到：
> 2014-05-03 15:08:06,587 server [INFO] Loading bars
2014-05-03 15:08:06,910 server [INFO] Waiting for workers
2014-05-03 15:08:58,347 server [INFO] Partial result 1242173.28754 with parameters: ('dia', 150, 5, 2, 91, 19) from worker-95583
2014-05-03 15:08:58,967 server [INFO] Partial result 1203266.33502 with parameters: ('dia', 150, 5, 2, 81, 19) from worker-95584
2014-05-03 15:09:52,097 server [INFO] Partial result 1220763.1579 with parameters: ('dia', 150, 5, 3, 83, 24) from worker-95584
2014-05-03 15:09:52,921 server [INFO] Partial result 1221627.50793 with parameters: ('dia', 150, 5, 3, 80, 24) from worker-95583
2014-05-03 15:10:40,826 server [INFO] Partial result 1142162.23912 with parameters: ('dia', 150, 5, 4, 76, 17) from worker-95584
2014-05-03 15:10:41,318 server [INFO] Partial result 1107487.03214 with parameters: ('dia', 150, 5, 4, 83, 17) from worker-95583
.
.

正式答案是 该策略最高收益是$2314.40。其参数为：
1. entrySMA: 154
2. exitSMA: 5
3. rsiPeriod: 2
4. overBoughtThreshold: 91
5. overSoldThreshold: 18
###绘图

PyAlgoTrade 绘制一个 策略走势非常容易

建立一个文件 ，名为 sma_crossover.py：

    from pyalgotrade import strategy
    from pyalgotrade.technical import ma
    from pyalgotrade.technical import cross

    class SMACrossOver(strategy.BacktestingStrategy):
        def __init__(self, feed, instrument, smaPeriod):
            strategy.BacktestingStrategy.__init__(self, feed)
            self.__instrument = instrument
            self.__position = None
            # We'll use adjusted close values instead of regular close values.
            self.setUseAdjustedValues(True)
            self.__prices = feed[instrument].getPriceDataSeries()
            self.__sma = ma.SMA(self.__prices, smaPeriod)

        def getSMA(self):
            return self.__sma

        def onEnterCanceled(self, position):
            self.__position = None

        def onExitOk(self, position):
            self.__position = None

        def onExitCanceled(self, position):
            # If the exit was canceled, re-submit it.
            self.__position.exitMarket()

        def onBars(self, bars):
            # If a position was not opened, check if we should enter a long position.
            if self.__position is None:
                if cross.cross_above(self.__prices, self.__sma) > 0:
                    shares = int(self.getBroker().getCash() * 0.9 / bars[self.__instrument].getPrice())
                    # Enter a buy market order. The order is good till canceled.
                    self.__position = self.enterLong(self.__instrument, shares, True)
            # Check if we have to exit the position.
            elif not self.__position.exitActive() and cross.cross_below(self.__prices, self.__sma) > 0:
                self.__position.exitMarket()

然后建另一个文件。保存下面的代码：

    from pyalgotrade import plotter
    from pyalgotrade.barfeed import yahoofeed
    from pyalgotrade.stratanalyzer import returns
    import sma_crossover

    # Load the yahoo feed from the CSV file
    feed = yahoofeed.Feed()
    feed.addBarsFromCSV("orcl", "orcl-2000.csv")

    # Evaluate the strategy with the feed's bars.
    myStrategy = sma_crossover.SMACrossOver(feed, "orcl", 20)

    # Attach a returns analyzers to the strategy.
    returnsAnalyzer = returns.Returns()
    myStrategy.attachAnalyzer(returnsAnalyzer)

    # Attach the plotter to the strategy.
    plt = plotter.StrategyPlotter(myStrategy)
    # Include the SMA in the instrument's subplot to get it displayed along with the closing prices.
    plt.getInstrumentSubplot("orcl").addDataSeries("SMA", myStrategy.getSMA())
    # Plot the simple returns on each bar.
    plt.getOrCreateSubplot("returns").addDataSeries("Simple returns", returnsAnalyzer.getReturns())

    # Run the strategy.
    myStrategy.run()
    myStrategy.info("Final portfolio value: $%.2f" % myStrategy.getResult())

    # Plot the strategy.
    plt.plot()

这段代码做了三件事：
1. 从 CSV文件中加载feed:
2. 运行策略与feed提供的条形图。和运行附加的StrategyPlotter
3. 绘制策略

结果是这样的
![enter image description here](http://gbeced.github.io/pyalgotrade/docs/v0.17/html/_images/tutorial-5.png)

我希望你喜欢这个快速教程。也希望你下载PyAlgotrade 来编写你自己的教程