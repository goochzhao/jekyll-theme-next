---
title: MPAndroidChart使用分组功能小记
date: 2018-04-21 16：55
categories:
- 问题记录
tags:
- 图表框架使用
description:  使用分组功能时对于自定义X轴标签一直对不齐问题记录
---

## MPAndroidChart 使用分组功能遇到的问题

> 先说在使用的过程中遇到的问题：

- 简单介绍框架
- 如何接入该框架
- 简单使用
- 如何自定义X轴标签
- X轴标签和对应的 刻度对不齐问题

![mark](http://p7fpzn7qh.bkt.clouddn.com/goochzhao/180421/gHE4aI8bb1.png?imageslim)

###  简单介绍框架

MPAndroidChart是一款功能强大且易于使用的Android图表库。它运行在[API级别8](http://developer.android.com/guide/topics/manifest/uses-sdk-element.html#ApiLevels)以上。

作为附加功能，该库允许Android和iOS之间的跨平台开发，因为此库的iOS版本也可用，（:smile: 官网上抄的，不要打我！）

- 许多不同的图表类型：LineChart，BarChart（垂直，水平，堆叠，分组），PieChart，ScatterChart，CandleStickChart（用于财务数据），RadarChart（蜘蛛网图），BubbleChart
- 组合图表（例如一行中的线条和条形）
- 在两个轴上进行缩放（使用触摸手势，单独坐标轴或捏合缩放）
- 拖动/平移（带触摸手势）
- 分离（双）y轴
- 突出显示值（使用可定制的**弹出视图**）
- 将图表保存到SD卡（如图）
- 预定义的颜色模板
- 图例（自动生成，可定制）
- 可定制的轴（x轴和y轴）
- 动画（在x轴和y轴上构建动画）
- 限制行数（提供附加信息，最大值...）
- 监听触摸，手势和选择回调
- 完全可定制（油漆，字体，图例，颜色，背景，虚线......）
- [Realm.io](https://realm.io/)移动数据库支持通过[MPAndroidChart-Realm](https://github.com/PhilJay/MPAndroidChart-Realm)库
- 在Line-和BarChart中平滑渲染多达**10.000个**数据点
- 轻量级（方法计数〜1.4K）
- 可使用**.jar文件**（只有500kb大小）
- 作为**gradle依赖**和通过**maven提供**
- 文档清晰
- [Google-PlayStore演示应用程序](https://play.google.com/store/apps/details?id=com.xxmassdeveloper.mpchartexample)
- 广泛使用，对[GitHub](https://github.com/PhilJay/MPAndroidChart/issues)和stackoverflow 都有很好的支持- [mpandroidchart](https://stackoverflow.com/questions/tagged/mpandroidchart)
- 也适用于**iOS**：[图表](https://github.com/danielgindi/Charts)（API以同样的方式工作）
- 也可用于**Xamarin**：[MPAndroidChart.Xamarin](https://github.com/Flash3001/MPAndroidChart.Xamarin)

### 如何接入该框架

- [MPAndroidChart](https://github.com/PhilJay/MPAndroidChart)  github 地址 star数已经达到21K左右了还是很强大的

- 添加下列依赖到工程中的build.gradle文件

  ```gr
  allprojects {
  	repositories {
  		maven { url "https://jitpack.io" }
  	}
  }
  ```

- 添加下列依赖到项目中的build.gradle

  ```gro
  dependencies {
  	implementation 'com.github.PhilJay:MPAndroidChart:v3.0.3'
  }
  ```

添加好依赖就可以使用了，但是在引入改库的3.0+版本时要注意他的一些API较之前的版本有一些改动，详情还请仔细参考[wiki](https://github.com/PhilJay/MPAndroidChart/wiki)，

### 简单使用

在这里就以BarChart(条形统计图)为例 其他的类似

先看效果：

![mark](http://p7fpzn7qh.bkt.clouddn.com/goochzhao/180421/3BJ71b9h1d.png?imageslim)

这里使用的方法很简单，详情看官网

> 初始化样式

```java
private void initBarChartStyle(BarChart barChart) {
        mBarChart.setDescription(null);
        mBarChart.setPinchZoom(false);
        mBarChart.setScaleEnabled(false);
        mBarChart.setDrawBarShadow(false);
        mBarChart.setDrawGridBackground(false);
        mBarChart.setNoDataText("正在加载数据！");
        //设置是否支持拖拽
        mBarChart.setDragEnabled(false);
        //设置能否缩放
        mBarChart.setScaleEnabled(false);
        initXaxis(barChart);
        initYaxis(barChart);
        initLegent(barChart);
    }
```

> 初始化Y轴

```java
private void initYaxis(BarChart barChart) {
        //Y-axis
        barChart.getAxisRight().setEnabled(false);
        YAxis leftAxis = barChart.getAxisLeft();
        leftAxis.setValueFormatter(new LargeValueFormatter());
        leftAxis.setDrawGridLines(false);
        leftAxis.setSpaceTop(35f);
        leftAxis.setAxisMinimum(0f);
    //在这里可以自定义y轴便签
        leftAxis.setValueFormatter(new IAxisValueFormatter() {
            @Override
            public String getFormattedValue(float value, AxisBase axis) {
                return value + "%";
            }
        });
    }

```

> 初始化X轴

```java
 /**
     * 设置x轴
     * @param barChart
     */
    private void initXaxis(BarChart barChart) {
        XAxis xAxis = barChart.getXAxis();
        xAxis.setGranularity(1f);
        xAxis.setGranularityEnabled(true);
        xAxis.setCenterAxisLabels(true);
        xAxis.setDrawGridLines(false);
        xAxis.setAxisMaximum(3);
        xAxis.setPosition(XAxis.XAxisPosition.BOTTOM);
    }
```

> 初始化图例

```java
 /**
     * 设置图例
     * @param barChart
     */
    private void initLegent(BarChart barChart) {
        //设置图例样式，默认可以显示，设置setEnabled（false）；可以不绘制
        Legend l = barChart.getLegend();
        l.setVerticalAlignment(Legend.LegendVerticalAlignment.TOP);
        l.setHorizontalAlignment(Legend.LegendHorizontalAlignment.LEFT);
        l.setOrientation(Legend.LegendOrientation.HORIZONTAL);
        l.setDrawInside(false);
        l.setForm(Legend.LegendForm.SQUARE);
        l.setFormSize(9f);
        l.setTextSize(11f);
        l.setXEntrySpace(4f);
        l.setWordWrapEnabled(true);
    }
```

> 最重要的是分组设置

```java
/**
     * 设置数据
     * @param data 数据
     * @param xVals x轴标签
     * @throws Throwable
     */
    public void setData(BarData data, List<String> xVals) throws Throwable {
        data.setDrawValues(false);
        //这里有三个参数需要解释一下，见下文
        if (((barSpace + barWidth) * groupCount + groupSpace) != 1.0f) {
            throw new Throwable("不满足为1的条件");
        }
        mBarChart.setData(data);
        mBarChart.getBarData().setBarWidth(barWidth);
        mBarChart.getXAxis().setGranularity(1f);
        mBarChart.getXAxis().setCenterAxisLabels(true);

        mBarChart.getXAxis().setAxisMinimum(0);
        mBarChart.getXAxis().setAxisMaximum(0 + mBarChart.getBarData().getGroupWidth(groupSpace, barSpace) * groupCount);
        mBarChart.groupBars(0, groupSpace, barSpace);
        mBarChart.getData().setHighlightEnabled(false);
        //自定义x轴便签在这里设置即可
        mBarChart.getXAxis().setValueFormatter(new IndexAxisValueFormatter(xVals));
        mBarChart.invalidate();
    }
```

上述代码中描述中有几个参数：

- groupCount：一组内有几个条形图
- groupSpace：各组之间的空隙
- barWidth：条形图的自身宽度
- barSpace：组内各个条形图的间隙

在这里有一个等式：`(barSpace + barWidth)* groupCount + groupSpace =1.0f` 如果这个等式不满足则就会出现标签对不齐的问题。

> 设置数据

```jav
 List<Integer> colors = new ArrayList<>();
        colors.add(Color.BLACK);
        colors.add(Color.GRAY);
        colors.add(Color.RED);
        colors.add(Color.GREEN);
        float barWidth = 0.2f;
        float barSpace = 0f;
        float groupSpace = 0.2f;
        int groupCount = 4;

        ArrayList<BarEntry> yVals = new ArrayList<>();//Y轴方向第一组数组
        ArrayList<BarEntry> yVals2 = new ArrayList<>();//Y轴方向第二组数组
        ArrayList<BarEntry> yVals3 = new ArrayList<>();//Y轴方向第三组数组
        ArrayList<BarEntry> yVals4 = new ArrayList<>();//Y轴方向第三组数组
        final ArrayList<String> xVals = new ArrayList<>();//X轴数据

        for (int i = 0; i <= 3; i++) {//添加数据源
            xVals.add((i + 1) + "月");
            yVals.add(new BarEntry(i, (i + 1) * 100));
            yVals2.add(new BarEntry(i, (i + 1) * 100));
            yVals3.add(new BarEntry(i, (i + 1) * 100));
            yVals4.add(new BarEntry(i, (i + 1) * 100));
        }
        BarDataSet barDataSet = new BarDataSet(yVals, "小明每月支出");
        barDataSet.setColor(Color.RED);//设置第一组数据颜色

        BarDataSet barDataSet2 = new BarDataSet(yVals2, "小花每月支出");
        barDataSet2.setColor(Color.GREEN);//设置第二组数据颜色

        BarDataSet barDataSet3 = new BarDataSet(yVals3, "小蔡每月支出");
        barDataSet3.setColor(Color.YELLOW);//设置第三组数据颜色
        BarDataSet barDataSet4 = new BarDataSet(yVals4, "小灵每月支出");
        barDataSet4.setColor(Color.BLUE);//设置第三组数据颜色
        BarData bardata = new BarData(barDataSet, barDataSet2, barDataSet3, barDataSet4);
```

### 如何自定义标签

这个功能在3.0+之前的代码是和现在不一样的，之前的是定义一个`list<string>` 传入BarData的构造器中，现在这个API去掉了，下载想要自定义X,Y轴标签只能通过`Axias` 的`setValueFormatter` 方法可以自定义，

### X轴标签和对应的刻度对不齐问题

这个在上文已经说了`(barSpace + barWidth)* groupCount + groupSpace =1.0f`满足这个条件即可

