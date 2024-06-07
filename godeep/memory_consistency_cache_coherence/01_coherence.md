# Coherence

## 基本模型

![Baseline system model](https://raw.githubusercontent.com/TDAkory/ImageResources/main/img/baselinesystemmodel.png)

书中考虑如上所示的基本模型：包括单个多核处理器芯片和片外主存。

* 多核处理器芯片由多个单线程核心组成，每个核心都有自己的私有数据缓存，以及一个由所有核心共享的最后一级缓存 (last-level cache, LLC)
* 每个核心的数据缓存都使用物理地址进行访问，并且采用写回 (write-back) 方式。核心和 LLC 通过互连网络相互通信。

需要注意的是，书中“缓存”一词特指核心的私有数据缓存，LLC被认为是“内存侧缓存，memory side cache”。

## 一致性协议和接口

一致性协议必须确保 writes 对所有处理器都可见

