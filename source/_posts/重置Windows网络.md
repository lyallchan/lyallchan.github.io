toc: true
title: 重置Windows网络
date: 2021-02-25 09:38
tags: [WSL2, DNS, 网络]
description:

---

# 重置Windows网络

WSL2 DNS有时不能正确解析，有时又是正常的，看到一篇[帖子](https://gist.github.com/sivinnguyen/8bc0125b274250683a97e149cf270040)，提到重置Windows可以解决，试了一下，没有动WSL的DNS配置，还是由WSL动态生成的，只是重置了Windows网络，就解决了，记录相关的命令

<!--more-->

```bat
wsl --shutdown  
netsh winsock reset  
netsh int ip reset all  
netsh winhttp reset proxy  
ipconfig /flushdns  
```