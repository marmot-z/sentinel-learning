# sentinel-learning

笔者在日常工作中使用到 sentinel 实现项目限流、降级等，出于了解实现原理、学习优秀框架代码的目的，编写了该文档。   
该文档主要讲述 sentinel 的基本概念和部分核心实现，帮助刚接触 sentinel 的开发人员快速了解 sentinel。

本项目解析的 sentinel 代码版本为 1.8.7，可能与您阅读的代码实现有部分出入，但实现原理应该基本相同。

## 目录
- [sentinel core](./sentinel-core.md)
  - [concept（核心概念）](./concept.md)
  - [slot chain（处理链）](./slot-chain.md) 
  - [sliding window（滑动窗口，统计时使用数据结构）](./sliding-window.md)
- sentinel SPI
- sentinel adapter
- sentinel dashboard
- sentinel extension（规则的实例化）

## 参考
- [sentinel-tutorial](https://github.com/all4you/sentinel-tutorial)