toc: true
title: VIM美化XML-JSON
date: 2018-10-18 14:32:33
tags: [VIM, XML, JSON]
description: 

---

# VIM美化XML-JSON

使用VIM美化XLM和JSON，XML使用`xmllint`，JSON使用`json_pp`或者`python -m json.tool`

<!--more-->

# VIM美化XML

```
%!xmllint --format -
```

VIM调用外部命令
    % - 文档所有内容
    ！- VIM调用外部命令
    xmllint - 实际美化xml命令
    --format - 格式化
    - - 输出到屏幕，被VIM捕获后，输出到VIM

# VIM美化JSON

```
%!python -m json.tool
```

