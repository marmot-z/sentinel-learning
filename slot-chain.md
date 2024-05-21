# slot chain

在前面我们通过自己动手实现，了解了 sentinel 中的一些难点。接下来我们通过阅读源码 + 调试的方式来
了解 sentinel 实现。 我们以官方示例为入口，开始 sentinel 的源码阅读之旅吧。

```java
class Demo {
    public static void main(String[] args) {
        initFlowRules();

        while (true) {
            try (Entry entry = SphU.entry("HelloWorld")) {
                System.out.println("hello world");
            } catch (BlockException ex) {
                System.out.println("blocked!");
            }
        }
    }
}
```

在这段代码中，我们可以看到除了定义规则之外，最核心的代码就是 `SphU.entry` 方法了。顺着调用链路往下走，
我们可以看到逻辑大致如下：

```java
class CtSph {
    @Override
    public Entry entry(String name, EntryType type, int count, Object... args) throws BlockException {
        // 根据名称创建 Resource
        StringResourceWrapper resource = new StringResourceWrapper(name, type);
        return entry(resource, count, args);
    }
}
```
这里引入了一个概念：Resource

Resource 是 sentinel 中最重要的概念之一，在 sentinel 中对 Resource 定义规则（API 定义或动态数据源写入），检查资源请求流量是否符合定义规则预期
，来保护业务代码或其他临界区资源。资源可以通过以下两种 API 来定义：
1. try-catch 方式（通过 SphU.entry(...)），当 catch 到BlockException时执行异常处理(或fallback)
2. if-else 方式（通过 SphO.entry(...)），当返回 false 时执行异常处理(或fallback)

除了 API 方式之外，从 0.1.1 版本开始，我们还可以通过 @SentinelResource 注解来定义资源。

在 sentinel 中，Resource 的实现有两种：
- StringResource，对字符描述资源的包装实现
- MethodResource，对调用方法的包装实现
  ![](./images/resource-uml.png)

在创建完资源之后，接下里就是对资源的统计以及规则的检查了：

```java
class CtSph {
    private Entry entryWithPriority(ResourceWrapper resourceWrapper, int count, boolean prioritized, Object... args)
            throws BlockException {
        // 创建 context
        Context context = ContextUtil.getContext();
        // context 的判断处理和一些条件检查
        // ...

        // 获取资源的处理链
        ProcessorSlot<Object> chain = lookProcessChain(resourceWrapper);

        Entry e = new CtEntry(resourceWrapper, chain, context, count, args);
        try {
            // 开始对资源进行处理，包括请求统计和规则检查
            chain.entry(context, resourceWrapper, null, count, prioritized, args);
        } catch (BlockException e1) {
            e.exit(count, args);
            throw e1;
        } catch (Throwable e1) {
            RecordLog.info("Sentinel unexpected exception", e1);
        }
        return e;
    }
}
```

这里又引入两个新的概念：Context 和 Slot：

### Context
在程序中较长的处理链路上，随着调用链路越深，相关附加信息就会越多，为了保存这些信息，我们会创建额外的容器来存储，这种类我们一般称之为上下文（Context），
比如 servlet 中的 ServletContext 等。在 sentinel 中每一次调用也有对应的上下文 Context，官方对其描述为：
`This class holds metadata of current invocation`。

其内部维护了以下几个字段：
- entranceNode：当前调用链的根节点
- Entry：当前调用链的 Entry
- node：与当前entry所对应的curNode
- origin：当前调用链的调用源

另外说下，上下文一般可以通过函数参数或者线程上下文（这种方式更常见）进行获取。


### Slot
根据上述代码我们可以看到，slot 与资源的处理有关，并且它们是以 chain（链）的模式进行工作的。

在 sentinel 中，处理资源的最小单元为 slot（槽），它们通过特定顺序组成一个 SlotChain 进行工作，来
实现资源的限流、降级、熔断等功能。

Slot 通过 [Sentinel 中的 SPI 机制加载]()，其顺序也可以通过提供 SlotChainBuilder 实现来自定义编排。
相关的 slot 默认实现有以下这些（安装调用顺序从前至后排列）：
1. NodeSelectorSlot
2. ClusterBuilderSlot
3. LogSlot
4. StatisticsSlot
5. AuthoritySlot
6. SystemSlot
7. FlowSlot
8. DefaultCircuitBreakerSlot
9. DegradeSlot

