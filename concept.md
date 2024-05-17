# concept

## Resource
Resource 是 sentinel 中最重要的概念之一，在 sentinel 中对 Resource 定义规则（API 定义或动态数据源写入），检查资源请求流量是否符合定义规则预期
，来保护业务代码或其他临界区资源。资源可以通过以下两种 API 来定义：
1. try-catch 方式（通过 SphU.entry(...)），当 catch 到BlockException时执行异常处理(或fallback)
2. if-else 方式（通过 SphO.entry(...)），当返回 false 时执行异常处理(或fallback)

除了 API 方式之外，从 0.1.1 版本开始，我们还可以通过 @SentinelResource 注解来定义资源。

在 sentinel 中，Resource 的实现有两种：
- StringResource，对字符描述资源的包装实现
- MethodResource，对调用方法的包装实现
  ![](./images/resource-uml.png)

## Context

## Slot