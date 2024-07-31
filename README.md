# sentinel-learning

笔者在日常工作中使用到 sentinel 来实现项目限流、降级等，出于了解限流原理、学习优秀框架代码的目的，编写了该文档。
该文档主要讲述了 sentinel 的基本概念和部分核心实现，文章中大量使用图例来阐释底层原理，希望能让读者更加直观、快速的了解
sentinel 的核心概念和底层原理。

<u>_本项目使用的 sentinel 代码版本为 1.8.7，可能与您阅读的代码实现有部分出入，但实现原理应基本相同。_</u>

## 目录
- [什么是 sentinel？](./preamble.md)
- [自己来实现 sentinel](./sentinel-implement.md)
- [sentinel 中的核心概念](./concept.md)
- [sentinel 进入、退出资源](./slot-chain.md) 
- [sentinel 如何统计资源访问情况](./sliding-window.md)
- [sentinel 中的扩展](./sentinel-spi.md)
- [sentinel 集成其他框架](./sentinel-integrated.md)
- [sentinel 控制台](./sentinel-dashboard.md)
- [sentinel 规则的动态更新](./dynamic-datasource.md)
- [sentinel 网关限流原理]()
- [sentinel实践-QPS检测失效]()
- [sentinel实践-网关的线程数检测]()

## 贡献

本人水平有限，文中难免有所纰漏。如发现有错误、遗漏或需补充之处，欢迎提交 PR 或者 issue。

## 参考
本文在编写过程中参考了 [sentinel-tutorial](https://github.com/all4you/sentinel-tutorial) 项目以及 [官网文档](https://sentinelguard.io/zh-cn/docs/introduction.html) 内容。

**最后，文章编写不易，如有帮助，欢迎 star ⭐**