# sentinel-learning

笔者在日常工作中使用到 sentinel 实现项目限流、降级等，出于了解实现原理、学习优秀框架代码的目的，编写了该文档。   
该文档主要讲述 sentinel 的基本概念和部分核心实现，帮助刚接触 sentinel 的开发人员快速了解 sentinel。

本项目解析的 sentinel 代码版本为 1.8.7，可能与您阅读的代码实现有部分出入，但实现原理应该基本相同。

## 目录
- [什么是 sentinel？](./preamble.md)
- [自己来实现 sentinel](./sentinel-core.md)
- [sentinel 中的核心概念](./concept.md)
- [sentinel 进入、退出资源](./slot-chain.md) 
- [sentinel 如何统计资源访问情况](./sliding-window.md)
- [sentinel 中的扩展](./sentinel-spi.md)
- [sentinel 集成其他框架，如何快速定义资源](./sentinel-integrated.md)
- [sentinel 控制台](./sentinel-dashboard.md)
- [规则的持久化和动态更新](./dynamic-datasource.md)

## 参考
- [sentinel-tutorial](https://github.com/all4you/sentinel-tutorial)