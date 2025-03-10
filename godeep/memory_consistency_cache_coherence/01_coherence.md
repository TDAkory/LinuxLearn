# Coherence

## 基本模型

![Baseline system model](https://raw.githubusercontent.com/TDAkory/ImageResources/master/img/baselinesystemmodel.png)

书中考虑如上所示的基本模型：包括单个多核处理器芯片和片外主存。

* 多核处理器芯片由多个单线程核心组成，每个核心都有自己的私有数据缓存，以及一个由所有核心共享的最后一级缓存 (last-level cache, LLC)
* 每个核心的数据缓存都使用物理地址进行访问，并且采用写回 (write-back) 方式。核心和 LLC 通过互连网络相互通信。

需要注意的是，书中“缓存”一词特指核心的私有数据缓存，LLC被认为是“内存侧缓存，memory side cache”。

## 一致性协议和接口

一致性协议必须确保 writes 对所有处理器都可见

处理器通过一致性接口（coherence interface）与 一致性协议（coherence protocol）交互。该接口提供了两种方法：

1. read-request，接受内存位置作为参数并返回一个值
2. write-request，接受内存位置和值作为参数，并返回一个确认（acknowledgment）

![coherence protocal interface](https://raw.githubusercontent.com/TDAkory/ImageResources/master/img/coherence_protocal_interface.png)

书中将一致性协议分为了两类，主要根据是其是否能够与连贯性模型（consistency model）清晰的隔离开来，独立考虑：

1. "**Consistency-agnostic coherence**": a write is made visible to all other cores before returning, because writes are propagated synchronously. 这种协议提供了与一个不包含缓存的原子内存系统相同的接口，这有助于分离关注点，任何与这个协议交互的子系统，都可以假设其交互对象不存在缓存，是个原子的内存系统。
2. "**Consistency-directed coherence**": more-recent category, writes are propagated asynchronously—a write can thus return before it has been made visible to all processors, thus allowing for stale values (in real time) to be observed. 这种协议需要提供保证：即最终writes的可见顺序符合 consistency model 规定的排序规则。

## 一致性不变量（Consistency-Agnostic Coherence Invariants）

1. **"Single-Writer, Multiple-Read (SWMR)"** Invariant. For any memory location A, at any given time, there exists only a single core that may write to A (and can also read it) or some number of cores that may only read A.
![deviding memory location lifetime into epochs](https://raw.githubusercontent.com/TDAkory/ImageResources/master/img/memory_loc_lifetime_into_epoch.png)
2. **"Data-Value Invariant"**. The value of the memory location at the start of an epoch is the same as the value of the memory location at the end of the its last read-write epoch.

绝大多数 coherence 协议，称为“无效化协议 (invalidate protocols)”，都是为了维护这些不变量而明确设计的。如果一个核心想要读取一个内存位置，它会向其他核心发送消息以获取该内存位置的当前值，并确保其他核心没有将该内存位置的缓存副本保持在 read-write 状态。这些消息结束了任何活跃的 read-write 时期，并开始一个 read-only 时期。如果一个核心想要写入一个内存位置，它会向其他核心发送消息以获取内存位置的当前值，如果它还没有有效的 read-only 缓存副本，并确保没有其他核心有以 read-only 或 read-write 状态缓存该内存位置的副本。这些消息结束任何活跃的 read-write 或 read-only 时期，并开始一个新的 read-write 时期。