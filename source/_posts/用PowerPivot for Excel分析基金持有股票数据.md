toc: true
title: 用PowerPivot for Excel分析基金持有股票数据
date: 2021-02-17 14:03  
tags: [PowerPivot, Excel]
description:

---

# 目的

给定基金组合，查询每种基金有哪些重仓股票，占比多少，最终得到每个股票的市值和在基金组合中的占比。

在[前面一篇文章](http://lyallchan.github.io/2021/02/17/%E7%94%A8akshare%E8%8E%B7%E5%BE%97%E5%9F%BA%E9%87%91%E9%87%8D%E4%BB%93%E8%82%A1%E7%A5%A8%E6%95%B0%E6%8D%AE/)中，已经通过akshare获取了基金持有股票的数据，这些数据包括

基金数据：基金代码、基金名称、市值、组合占比
股票数据：股票代码、股票名称、占净值比例、基金代码

<!--more-->

# 使用PowerQuery导入数据

新建一个excel，将上面的数据通过PowerQuery导入，设置必要的数据类型。这样做的好处是所有分析都在新文件中进行，可以避免原始数据被修改损坏。相应的m代码为

基金
```
let
    源 = Excel.Workbook(File.Contents(#"文件路径"[文件路径]{0}), null, true),
    基金组合_Sheet = 源{[Item="基金组合",Kind="Sheet"]}[Data],
    提升的标题 = Table.PromoteHeaders(基金组合_Sheet, [PromoteAllScalars=true]),
    更改的类型 = Table.TransformColumnTypes(提升的标题,{{"基金代码", type text}, {"基金名称", type text}, {"市值", type number}, {"组合占比", type number}})
in
    更改的类型
```

股票
```
let
    源 = Excel.Workbook(File.Contents(文件路径[文件路径]{0}), null, true),
    基金持有股票_Sheet = 源{[Item="基金持有股票",Kind="Sheet"]}[Data],
    提升的标题 = Table.PromoteHeaders(基金持有股票_Sheet, [PromoteAllScalars=true]),
    更改的类型 = Table.TransformColumnTypes(提升的标题,{{"股票代码", type text}, {"股票名称", type text}, {"占净值比例", type number}, {"基金代码", type text}})
in
    更改的类型
```
其中的基金代码、股票代码都设置成text类型，避免丢失前导的0。

将上面两个表格加载到excel，选择“仅创建连接”和“将此数据添加到数据模型”。

![](./用PowerPivot%20for%20Excel分析基金持有股票数据/2021-02-17-14-33-04.png)

# 用PowerPivot分析

选择PowerPivot-管理，进入PowerPivot界面。
![](./用PowerPivot%20for%20Excel分析基金持有股票数据/2021-02-17-14-36-06.png)

## 设立关系

选择主页-关系图视图

![](./用PowerPivot%20for%20Excel分析基金持有股票数据/2021-02-17-14-38-02.png)

通过`基金代码`字段关联，很明显是一对多关系，`基金组合`为一端，`基金持有股票`为多端。

![](./用PowerPivot%20for%20Excel分析基金持有股票数据/2021-02-17-14-39-32.png)

## 增加计算字段

在`基金持有股票表`中，增加一个计算字段`股票市值`，表示一个基金中的一个股票的当前市值，用基金市值*股票占比

```
='基金持有股票'[占净值比例]*RELATED('基金组合'[市值])
```

由于`基金持有股票`表是多端，筛选器流只会从`一端`流到`多端`，要访问一端的表`基金组合`的相应记录，需要加上related函数，可以参考[DAX圣经](https://www.powerbigeek.com/understanding-relationship-in-powerbi/)。

## 增加度量值

为了计算各个股票的市值和以及占总资金的比例，增加一个度量值`市值占比`

```
市值占比:=
var s=CALCULATE(SUM('基金组合'[市值]),ALL('基金组合'))
return DIVIDE(sum('基金持有股票'[股票市值]),s)
```

分子是股票市值总和，分母是所有基金市值总和。

增加其它度量值，显示“基金持有股票”列表和一个“股票的持有基金”列表

在基金组合表中增加TOP股票

```
TOP股票:=CONCATENATEX('基金持有股票','基金持有股票'[股票名称],"-")
```

在基金持有股票表中增加基金持有
```
基金持有:=CONCATENATEX('基金持有股票',RELATED('基金组合'[基金名称]),"-")
```
注意这里是多端，还是要加上related函数才可以正确找到对应的记录

# 最后结果

基于`基金持有股票`做数据透视表，最后得到效果为

![](./用PowerPivot%20for%20Excel分析基金持有股票数据/2021-02-17-15-04-08.png)

不出所料，茅台、五粮液占比最高的，喝酒吃药是硬道理。



