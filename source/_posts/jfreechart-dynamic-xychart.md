---
title: JfreeChart实现数据的实时动态曲线显示
date: 2010-08-08 10:55:18
description: 
categories:
- Java
tags:
- Java
toc: true
author: John Tang
comments:
original:
permalink: 
---

## 目标： 

通过JfreeChart实现数据的实时动态曲线显示 
需要解决的问题 

- JfreeChart如何生成折线图？ 
- JfreeChart如何生成动态图？ 
- 如何改变JfreeChart的动态图的X轴？
- JfreeChart如何生成折线图？ 

JfreeChart提供了一个MemoryUsage的demo, 

1. 配置开发环境 
2. 下载Jfreechart的lib包 [http://sourceforge.net/projects/jfreechart/files/ ](http://sourceforge.net/projects/jfreechart/files/ )
3. 创建eclipse工程，引入jcommon-*.jar,jfreechart-*.jar 

MemoryUsage的源码可以在下面的打包文件里找到，将这个demo运行起来你就可以看到一个JVM 内存消耗的实时数据显示。 

分析源码后可以发现生成这样的图表主要用到了 

	org.jfree.data.time.TimeSeriesCollection org.jfree.data.time.TimeSeries 
	
	这个主要的功能是实时的收集数据，API文档是这样描述的 
	
	A collection of time series objects. 
	This class implements the XYDataset interface, as well as the extended IntervalXYDataset interface. 
	This makes it a convenient dataset for use with the XYPlot class. 


	org.jfree.chart.plot.XYPlot 
	
	是一个曲线图，通过指定XY的坐标来表示数据点，任何实现了XYDataset接口的类都可以通过它来显示，
	它通过XYItemRenderer来设置点数据的显示样式，从而生成各种不同的图表。
	
	A general class for plotting data in the form of (x, y) pairs. 
	This plot can use data from any class that implements the XYDataset interface. 
	XYPlot makes use of an XYItemRenderer to draw each point on the plot. 
	By using different renderers, various chart types can be produced. 

## 问题 

### 现在X轴是以时间线来显示的，可以设置以刻度或次序(1,2,3,4...)的格式显示吗？ 
可以， 将 

	org.jfree.data.time.TimeSeries org.jfree.data.time.TimeSeriesCollection 

换成 

	org.jfree.data.xy.XYSeries org.jfree.data.xy.XYSeriesCollection 

就可以了。 

### 那他们还是动态的图吗？ 

是的。
 
### 那是为什么呢？ 

因为XYSeriesCollection 和 TimeSeriesCollection都实现了这样一个接口 

	org.jfree.data.general.SeriesChangeListener 

它继承自EventListener SeriesChangeListener extends java.util.EventListener 

它又定义了数据更改时发出通知的方法

>  Methods for receiving notification of changes to a data series. 
>  
>  void seriesChanged(SeriesChangeEvent event) Called when an observed series changes in some way. 

实现了这个接口的SeriesCollection都可以实时的更新数据 

添加数据可以使用 series.add(X,Y); 

更新可以使用 series.update(X,Y); 

还有清空之前的所有数据可以使用 series.clear(); 


### 如何实时的显示特定点的数据呢？ 

当然JfreeChart有一个tip的功能，当鼠标移上去的时候显示数据点的数据 

### 如何让它按照我们想要的格式显示呢？ 

这里你就需要使用 

	org.jfree.chart.annotations.XYTextAnnotation
	 A text annotation that can be placed at a particular (x, y) location on an XYPlot. 

这里有一小段的代码(更多的jfreechart的demo代码可以在附件里找到，相信看过之后，基本上都能满足你的要求了) 

	XYTextAnnotation textpointer = new XYTextAnnotation(X+","+Y, X+15, Y-15); 
	textpointer.setBackgroundPaint(Color.YELLOW); 
	textpointer.setPaint(Color.CYAN); 
	dynamicJfreeChart.getXYPlot().addAnnotation(textpointer); 

### 示例源码

下面的代码未整理，收集来源于网络

 [http://d.download.csdn.net/down/2605527/tangjunchf](http://d.download.csdn.net/down/2605527/tangjunchf)