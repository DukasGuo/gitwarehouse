#pyalgotrade 中文学习手册 
##前言
pyAlgotrade 是一个 基于事件驱动的算法交易 的Python库。它支持：
+ 通过CSV文件和 历史数据 进行回测
+ 模拟交易 用的是Xignite 0（Xignite.com	）和 Bitstamp( Bitstamp.com)的数据
+ 真实交易通过 Birstamp

它还易于用多台电脑 优化策略。


PyAlgotrade 依赖于 ：
+ [Numpy and  Scipy](%28http://numpy.scipy.org/)科学计算
+ [pytz](http://pytz.sourceforge.net/)时区
+ [matplotlib](http://matplotlib.sourceforge.net/) 绘图库
+ [ws4py](https://github.com/Lawouach/WebSocket-for-Python) WebSocket 的Client和Server端，用来支持 Bitstamp
+ [tornado](%28http://www.tornadoweb.org/en/stable/)Facebook的开源框架，直接内嵌了HTTP服务器。用来支持Bitstamp
+ [tweepy](https://github.com/tweepy/tweepy)  Python的 推特 API库。

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
这些数据提供抽象。 例如，你将用一个CSV feed，从CSV文件中加载bars推送数据给策略。 Feeds 不限于 bars 格式，比如 一个推特 feed 允许将 推特事件纳入 交易决策
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
3. 运行策略 和 bars。bars是feed提供的。

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

运行脚本的话，你看到这样的结果：
> 2000-01-26 00:00:00 strategy [INFO] BUY at $27.26
> 2000-01-28 00:00:00 strategy [INFO] SELL at $24.74
> 2000-02-03 00:00:00 strategy [INFO] BUY at $26.60
> 2000-02-22 00:00:00 strategy [INFO] SELL at $28.40
> 2000-02-23 00:00:00 strategy [INFO] BUY at $28.91
> 2000-03-31 00:00:00 strategy [INFO] SELL at $38.51
> 2000-04-07 00:00:00 strategy [INFO] BUY at $40.19
> 2000-04-12 00:00:00 strategy [INFO] SELL at $37.44
> 2000-04-19 00:00:00 strategy [INFO] BUY at $37.76
> 2000-04-20 00:00:00 strategy [INFO] SELL at $35.45
> 2000-04-28 00:00:00 strategy [INFO] BUY at $37.70
> 2000-05-05 00:00:00 strategy [INFO] SELL at $35.54
> 2000-05-08 00:00:00 strategy [INFO] BUY at $36.17
> 2000-05-09 00:00:00 strategy [INFO] SELL at $35.39
> 2000-05-16 00:00:00 strategy [INFO] BUY at $37.28
> 2000-05-19 00:00:00 strategy [INFO] SELL at $34.58
> 2000-05-31 00:00:00 strategy [INFO] BUY at $35.18
> 2000-06-23 00:00:00 strategy [INFO] SELL at $38.81
> 2000-06-27 00:00:00 strategy [INFO] BUY at $39.56
> 2000-06-28 00:00:00 strategy [INFO] SELL at $39.42
> 2000-06-29 00:00:00 strategy [INFO] BUY at $39.41
> 2000-06-30 00:00:00 strategy [INFO] SELL at $38.60
> 2000-07-03 00:00:00 strategy [INFO] BUY at $38.96
> 2000-07-05 00:00:00 strategy [INFO] SELL at $36.89
> 2000-07-21 00:00:00 strategy [INFO] BUY at $37.19
> 2000-07-24 00:00:00 strategy [INFO] SELL at $37.04
> 2000-07-26 00:00:00 strategy [INFO] BUY at $35.93
> 2000-07-28 00:00:00 strategy [INFO] SELL at $36.08
> 2000-08-01 00:00:00 strategy [INFO] BUY at $36.11
> 2000-08-02 00:00:00 strategy [INFO] SELL at $35.06
> 2000-08-04 00:00:00 strategy [INFO] BUY at $37.61
> 2000-09-11 00:00:00 strategy [INFO] SELL at $41.34
> 2000-09-29 00:00:00 strategy [INFO] BUY at $39.07
> 2000-10-02 00:00:00 strategy [INFO] SELL at $38.30
> 2000-10-20 00:00:00 strategy [INFO] BUY at $34.71
> 2000-10-31 00:00:00 strategy [INFO] SELL at $31.34
> 2000-11-20 00:00:00 strategy [INFO] BUY at $23.35
> 2000-11-21 00:00:00 strategy [INFO] SELL at $23.83
> 2000-12-01 00:00:00 strategy [INFO] BUY at $25.33
> 2000-12-21 00:00:00 strategy [INFO] SELL at $26.72
> 2000-12-22 00:00:00 strategy [INFO] BUY at $29.17