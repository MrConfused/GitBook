---
description: 分析项目编译耗时，有目的的提高编译效率
---

# 分析iOS项目编译耗时

1. 下载xclogparser：`brew install xclogparser`
2. build需要分析的项目
3. 找到编译生成的log文件，一般在`DerivedData/<Project>/Logs/Build/XXX.xcactivitylog`
4. 生成分析文件：`xclogparser parse --file DerivedData/<Project>/Logs/Build/XXX.xcactivitylog --reporter chromeTracer > xx.txt`
5. 用[https://ui.perfetto.dev/](https://ui.perfetto.dev) 打开上面生成的txt文件
