+++
title = "pyqtgraph实时画图卡顿问题发现及解决"
date = 2018-03-22T22:31:30+08:00
tags = ["pyqtgraph", "卡顿"]
categories = ['programming']
keywords = ['pyqtgraph','卡顿']
markup = "goldmark"
+++
为某设备编写的数据采集及控制软件发现一个随着时间越来越卡顿的问题, 卡到通过QtTimer更新
的LCDNumber组件跳秒, 本该从8s变为7s这样逐秒递减卡成了停留在8s, 突然跳到3s这样. 最终找到
原因, 源自pyqtgraph组件.
<!--more-->
## 原因探索
问题重现: 进行一段较长时间的运行, 可重现卡顿问题.   
怀疑: 由于观察内存用量随时间持续上涨, 猜测原因在于压力实时显示部分   
测试: 在压力采集线程内用time模块记录每次采集的时间； 在压力显示模块用time模块记录每次
plot所用的时间
结果: 采集线程的采集时间长时间不变；压力显示plot所用时间逐步增长.

继续分析: plot操作会对list实例slice, 得到一定长度的数据, 然后进行plot, 猜测有可能是
slice随着list本身的增大会有耗时的增长.    

测试: 将plot()传参和分片操作分开, 用time模块记耗时, 发现分片操作时间极短, 而plot的时间
逐步增长.

## 解决问题
至此, 确定下来问题来自于pyqtgraph的plot, 那么为什么会随着时间逐步增长plot所需时间呢?
并且伴随着内存消耗的持续增长. 查找pyqtgraph源码

```python
def plot(self, *args, **kargs):
    """
    Add and return a new plot.
    See :func:`PlotDataItem.__init__ <pyqtgraph.PlotDataItem.__init__>` for data arguments

    Extra allowed arguments are:
        clear    - clear all plots before displaying new data
        params   - meta-parameters to associate with this data
    """


    clear = kargs.get('clear', False)
    params = kargs.get('params', None)

    if clear:
        self.clear()

    item = PlotDataItem(*args, **kargs)

    if params is None:
        params = {}
    self.addItem(item, params=params)

    return item
```
可以看到每次plot都会生成一个PlotDataItem, 然后添加到Items列表, 视图会自动更新. 由于
plot没有添加clear参数, 每次plot生成一个PlotDataItem, 随着时间的增加, 每次更新视图
所需处理的dataItem越多, 所以耗时增长. 实际在该场景下是不需要保留PlotDataItem的, 每次
plot时将其清空即可. 即使用`plot(x, y, clear=True)`的方式绘制视图.
