---
title:  jsontool使用
date: 2018年06月21日 22时15分52秒
tags:  [命令]
categories: tool
toc: true
---

json-tool使用：`java -jar json-tool.jar "json文件目录" "jsonPath路径"`
示例：

```
java -jar /Users/yaosong/Documents/json-tool.jar "/Users/yaosong/tmp/access_report_data_by_token.json"  "$.report_data.behavior_check[?(@.check_point_cn == '朋友圈在哪里')].evidence"
```
![示例截图](https://www.github.com/yaosong5/tuchuang/raw/master/mdtc/2018/5/30/1527644586687.jpg)