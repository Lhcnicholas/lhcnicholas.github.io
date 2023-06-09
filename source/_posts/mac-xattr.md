---
title: Mac打不开应用，显示“文件已损坏”
date: 2023-05-09 09:53:35
tags: 
    - Mac
    - Xattr
---

平时在网上下载的应用，由于不是在苹果商店里购买的，经常会出现抱错，“文件已损坏"。这时候就很头疼。

## 解决办法

解决办法很简单，一行命令就搞定了。
``` bash
sudo xattr -rd com.apple.quarantine {{file_path}}
```

引用一段AI的回答。

> 这是一个在MacOS上的shell命令，用于从文件或目录中删除隔离属性（quarantine attribute）。隔离属性是操作系统为从互联网下载或从其他来源接收的文件添加的一种安全措施，以防止用户意外执行可能有害的代码。<br>
命令`sudo xattr -rd com.apple.quarantine {{file_path}}`将从指定的文件或目录中删除隔离属性，使其可以在没有任何警告或限制的情况下执行。`sudo`命令用于以管理员权限运行命令，这可能取决于文件的权限。`-r`选项用于递归地从指定目录中删除属性，`-d`选项用于从目录本身中删除属性。