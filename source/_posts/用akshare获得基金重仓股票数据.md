toc: true
title: 用akshare获得基金重仓股票数据
date: 2021-02-17 12:33 
tags: [pandas, akshare, excel]
description:

---

# 目的

给定基金组合，查询每种基金有哪些重仓股票，占比多少，最终得到每个股票的市值和在基金组合中的占比。

<!--more-->

在github上找到了[akshare](https://www.akshare.xyz/zh_CN/latest/)，根据官网介绍

>AKShare 是基于 Python 的财经数据接口库, 目的是实现对股票、期货、期权、基金、外汇、债券、指数、加密货币等金融产品的基本面数据、实时和历史行情数据、衍生数据从数据采集、数据清洗到数据落地的一套工具, 主要用于学术研究目的.

>AKShare 的特点是获取的是相对权威的财经数据网站公布的原始数据, 通过利用原始数据进行各数据源之间的交叉验证, 进而再加工, 从而得出科学的结论.

# 搭建开发环境

在很久以前做过一点python2开发，没用过python3和pandas，最近又正好在研究excel的powertpivot。为了加快速度，将基金组合输入excel，通过akshare得到股票数据，再写入excel，通过excel进行后续的分析。

搭建环境还是使用命令行工具scoop，用scoop安装miniconda和vscode。再通过conda安装jupter、numpy、pandas，通过pip安装akshare。vscode安装jupter和python插件。其中jupter主要是进行测试实验，实验通过了再写入代码，节约了很多调试时间。

安装过程参考了很多网上资料，当时也没有记下来，不过好在资料很多，随便搜搜就能搜出一大堆，这里就不展示了。

# 涉及功能点

在jupter中边学边试，最后归纳下来大概有几个功能点：

1. 读取excel，包括指定字段数据类型，指定索引等
2. 写入excel，包括删除sheet，在已有excel添加sheet，最后用了openpyxl
3. pandas基本的定位行、定位列、行过滤
4. pandas合并

# 代码

```python
import numpy as np
import pandas as pd
import akshare as ak
import openpyxl

# excel文件
f = r'excel.xlsx'
wb = openpyxl.load_workbook(f)
writer = pd.ExcelWriter(f, engine='openpyxl')
writer.book = wb

# 删除已有的目标sheet
stocks_sheet_name = '基金持有股票'
if ('基金持有股票' in wb.sheetnames):
    wb.remove(wb[stocks_sheet_name])

# 读取基金组合信息
f_types = {'基金代码':str, '市值':int, '基金名称':str, '组合占比':float}
funds = pd.read_excel(f, "基金组合", dtype=f_types)
funds = funds.set_index('基金代码')

# 初始化股票表
stocks = pd.DataFrame()

for i in funds.index:

    # 根据基金代码查询基金持有股票
    # 函数返回字段
    # [股票代码,股票名称,占净值比例,持股数,持仓市值,季度]
    s=ak.fund_em_portfolio_hold(code=i, year="2020")
    s=s[s['季度'].str.contains('4季度')] # 过滤4季度持有股票，避免重复
    s['占净值比例'] = s['占净值比例'].astype('float64')/100

    # 补齐基金代码
    s['基金代码']=i

    # 每个基金重复查询和处理，汇总成一张表
    stocks = stocks.append(s.iloc[:,1:],ignore_index=True)

# 结果写文件
stocks.to_excel(writer, sheet_name=stocks_sheet_name, index=False)
writer.save()
writer.close()
```

# 后记

后来用powershell也写了一个功能一样的脚本，使用蛋卷网提供的API，功能点包括

1. powershell读、写excel，使用excel com对象
2. powershell转换json数据，遍历

```powershell

$e = New-Object -ComObject excel.application
$f = $e.Workbooks.Open("excel.xlsx")
$s = $f.worksheets.item("funds")
$funds = $s.Range("funds").Columns(1).value2
$s = $f.worksheets.item("stocks")

$row = 2
foreach ($fund in $funds)
{
    $FundDetail = Invoke-Expression "(Invoke-WebRequest https://danjuanapp.com/djapi/fund/detail/$fund).content | ConvertFrom-Json"
    $Stocks = $FundDetail.data.fund_position.stock_list
    foreach( $Stock in $Stocks ) 
    {
        write-host $fund, $Stock.name, $Stock.code, $Stock.percent
        $s.cells.item($row,1)=$fund
        $s.cells.item($row,2)=$Stock.name
        $s.cells.item($row,3)=$Stock.code
        $s.cells.item($row,4)=$Stock.percent
        $row = $row + 1 
    }
}

$f.save()
$f.close()
$e.quit()
```

