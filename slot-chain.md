# slot chain

在前面的章节，我们通过自己动手实现和阅读[概念章节](./concept.md)，了解了 sentinel 中的一些概念以及实现难点。接下来我们通过阅读源码 + 调试的方式来
了解 sentinel 实现和原理。 我们以官方示例为切入口，开启 sentinel 的源码阅读之旅。

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

# 进入资源

在这段代码中，我们可以看到除了定义规则之外，最核心的代码就是 `SphU.entry` 方法了。在 entry 方法中，第一步就是为资源创建对应的 wrapper 实例

```java
public class CtSph {
    @Override
    public Entry entry(String name, EntryType type, int count, Object... args) throws BlockException {
        // 根据名称创建 Resource
        StringResourceWrapper resource = new StringResourceWrapper(name, type);
        return entry(resource, count, args);
    }
}
```

在创建完资源 wrapper 实例后，接下里就是 sentinel 的核心处理逻辑：即对资源的统计以及规则的检查。具体请看以下代码：

```java
public class CtSph {
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

通过以上代码以及在前面的[概念章节](./concept.md)中的介绍，我们可以了解到，资源的统计和规则检查主要是通过 processSlot 来实现。那么是如何创建对应的 processSlot 实例呢？请看以下代码：

```java
public class CtSph {
    ProcessorSlot<Object> lookProcessChain(ResourceWrapper resourceWrapper) {
        // 根据 resource 获取 processSlotChain
        ProcessorSlotChain chain = chainMap.get(resourceWrapper);
        // double check，线程安全的创建 processSlotChain 
        if (chain == null) {
            synchronized (LOCK) {
                chain = chainMap.get(resourceWrapper);
                if (chain == null) {
                    // processSlotChain 是否超过上限（即资源上限：6000）
                    if (chainMap.size() >= Constants.MAX_SLOT_CHAIN_SIZE) {
                        return null;
                    }

                    // 创建新的 processSlotChain
                    chain = SlotChainProvider.newSlotChain();
                    // ...
                }
            }
        }
        return chain;
    }
}

public final class SlotChainProvider {
    
    public static ProcessorSlotChain newSlotChain() {
        // 通过 spi 机制获取 SlotChainBuilder 实现，用于加载 slot 实现并编排顺序
        // 如果没有，则使用 DefaultSlotChainBuilder 创建 processSlotChain
        slotChainBuilder = SpiLoader.of(SlotChainBuilder.class).loadFirstInstanceOrDefault();

        if (slotChainBuilder == null) {
            // ...
            slotChainBuilder = new DefaultSlotChainBuilder();
        } 
        // ...
        
        return slotChainBuilder.build();
    }
} 
```

在前面的[概念章节](./concept.md)中我们了解到 processSlotChain 内部是以责任链的模式运行，那么其内部到底是怎样运行的呢？都有哪些逻辑呢？

我们可以通过一张官方的示意图来一探究竟：
![](https://user-images.githubusercontent.com/9434884/69955207-1e5d3c00-1538-11ea-9ab2-297efff32809.png)
在 processSlotChain 中有一个特定顺序排列的 slot 单链表，请求依次通过不同的 slot 来完成不同的功能。大致完成的功能如下：
1. 先创建数据结构用于数据统计，然后检查规则
2. 先检查优先级高的规则，再检查优先级低的规则

接下来我们按照调用顺序依次解读对应的 slot 代码，了解其实现的功能。 

processSlotChain 中的责任链实现。每个 slot 都持有下个 slot 的引用，通过 next.transformEntry 和 next.exit 来触发资源进入和退出的对应操作。
```java
public abstract class AbstractLinkedProcessorSlot<T> {
    
    @Override
    public void fireEntry(Context context, ResourceWrapper resourceWrapper, Object obj, int count, boolean prioritized, Object... args)
        throws Throwable {
        if (next != null) {
            next.transformEntry(context, resourceWrapper, obj, count, prioritized, args);
        }
    }

    @SuppressWarnings("unchecked")
    void transformEntry(Context context, ResourceWrapper resourceWrapper, Object o, int count, boolean prioritized, Object... args)
        throws Throwable {
        // 调用下一个 slot 的 entry 方法
        T t = (T)o;
        entry(context, resourceWrapper, t, count, prioritized, args);
    }

    @Override
    public void fireExit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
        // 调用下一个 slot 的 exit 方法
        if (next != null) {
            next.exit(context, resourceWrapper, count, args);
        }
    }
}
```

## NodeSelectorSlot
在进入资源前，NodeSelectorSlot 为每种 context 下的每个资源创建一个 defaultNode 实例，并维护一颗全局的调用树（每个 context 对应一颗调用树）
```java
public class NodeSelectorSlot {
    public void entry(Context context, ResourceWrapper resourceWrapper, Object obj, int count, boolean prioritized, Object... args)
            throws Throwable {
        // 在进入资源前，为每种 context 下的每个资源创建一个 DefaultNode 实例
        DefaultNode node = map.get(context.getName());
        // double check，线程安全的创建 DefaultNode
        if (node == null) {
            synchronized (this) {
                node = map.get(context.getName());
                if (node == null) {
                    node = new DefaultNode(resourceWrapper, null);
                    /*
                     * 根据资源进入的顺序，创建对应的调用树
                     * 每个 context 的 resource 对应的 node 仅创建一次，其在调用树中的位置确定后不再变化
                     * 
                     *   Constants.ROOT
                     *          |
                     *          | child
                     *          ↓
                     *  context1.entranceNode
                     *          |
                     *          | child
                     *          ↓
                     *  resource1 defaultNode
                     *          |
                     *          | child
                     *          ↓
                     *  resource2 defaultNode
                     */
                    ((DefaultNode) context.getLastNode()).addChild(node);
                }

            }
        }

        context.setCurNode(node);
        fireEntry(context, resourceWrapper, node, count, prioritized, args);
    }

    @Override
    public void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
        // 退出资源时，什么也不做
        fireExit(context, resourceWrapper, count, args);
    }
}

public class Context {
    public Node getLastNode() {
        // 第一次申请 entry，则资源对应的 defaultNode 挂载在 entranceNode 下
        // 继续申请 entry，则资源对应的 defaultNode 挂载在上一个资源的 defaultNode 下
        if (curEntry != null && curEntry.getLastNode() != null) {
            return curEntry.getLastNode();
        } else {
            return entranceNode;
        }
    }
}
``` 

## ClusterSlotBuilder
在进入资源前，ClusterSlotBuilder 为每种资源创建一个 clusterNode，用于记录资源全局的统计数据

```java
public class ClusterBuilderSlot {

    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                      boolean prioritized, Object... args)
            throws Throwable {
        // 在进入资源前，为每种资源创建一个 clusterNode 实例
        if (clusterNode == null) {
            synchronized (lock) {
                if (clusterNode == null) {
                    clusterNode = new ClusterNode(resourceWrapper.getName(), resourceWrapper.getResourceType());
                    // ...
                }
            }
        }
        node.setClusterNode(clusterNode);
        
        // ...
        fireEntry(context, resourceWrapper, node, count, prioritized, args);
    }

    @Override
    public void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
        // 退出资源时，什么也不做
        fireExit(context, resourceWrapper, count, args);
    }
}
```

## LogSlot
在进入和退出资源时，LogSlot 会记录相关异常信息
```java
public class LogSlot {

    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode obj, int count, boolean prioritized, Object... args)
        throws Throwable {
        // 在进入资源出现异常时，打印相关异常，并重新抛出 blockException
        try {
            fireEntry(context, resourceWrapper, obj, count, prioritized, args);
        } catch (BlockException e) {
            EagleEyeLogUtil.log(resourceWrapper.getName(), e.getClass().getSimpleName(), e.getRuleLimitApp(),
                context.getOrigin(), e.getRule() != null ? e.getRule().getId() : null, count);
            throw e;
        } catch (Throwable e) {
            RecordLog.warn("Unexpected entry exception", e);
        }
    }

    @Override
    public void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
        // 在退出资源出现异常时，打印相关异常
        try {
            fireExit(context, resourceWrapper, count, args);
        } catch (Throwable e) {
            RecordLog.warn("Unexpected entry exit exception", e);
        }
    }
}
```

## StatisticsSlot
在进入资源会后，StatisticsSlot 会统计各个维度（资源、特定来源、系统）的 passQps、threadNum 等值；

在退出资源时，StatisticsSlot 会统计各个维度（资源、特定来源、系统）的 rt、successCount、exceptionQps 等值
```java
public class StatisticsSlot {
    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                      boolean prioritized, Object... args) throws Throwable {
        try {
            // 先进行规则（如：flowRule、authorityRule等）的检查
            fireEntry(context, resourceWrapper, node, count, prioritized, args);

            // 资源通过 qps + count，调用线程数 +1
            node.increaseThreadNum();
            node.addPassRequest(count);

            // 特定来源的通过 qps + count，调用线程数 +1
            if (context.getCurEntry().getOriginNode() != null) {
                // ...
            }

            // 系统的通过 qps + count，调用线程数 +1
            if (resourceWrapper.getEntryType() == EntryType.IN) {
                // ...
            }

            // 执行注册的 slot entry 的 onPass 回调方法
        } 
        // 在流控规则效果设置为排队等待时，进入资源的 qps 超过规则限制后，
        // 会休眠一段时间再抛出该异常，此时应该允许访问资源
        catch (PriorityWaitException ex) {
            // 资源的调用线程数 +1
            // 特定来源的资源调用线程数 +1
            // 系统的线程调用数 +1
            // 执行注册的 slot entry 的 onPass 回调方法
        } catch (BlockException e) {
            // 将 blockException 记录到当前 entry
            context.getCurEntry().setBlockError(e);
            // 资源被阻塞的请求数 +count
            // 特定来源的资源被阻塞的请求数 +count
            // 系统的被阻塞的请求数 +count
            // 执行注册的 slot entry 的 onBlock 回调方法
            throw e;
        } catch (Throwable e) {
            // 将 blockException 记录到当前 entry
            context.getCurEntry().setError(e);
            throw e;
        }
    }

    @Override
    public void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
        Node node = context.getCurNode();

        // 进入资源时未出现 blockException，记录相关值（rt、successCount、exceptionQps）
        if (context.getCurEntry().getBlockError() == null) {
            // 计算 rt 值
            long completeStatTime = TimeUtil.currentTimeMillis();
            context.getCurEntry().setCompleteTimestamp(completeStatTime);
            long rt = completeStatTime - context.getCurEntry().getCreateTimestamp();

            Throwable error = context.getCurEntry().getError();

            // 记录 rt、进入成功数量以及异常 qps
            recordCompleteFor(node, count, rt, error);
            recordCompleteFor(context.getCurEntry().getOriginNode(), count, rt, error);
            if (resourceWrapper.getEntryType() == EntryType.IN) {
                recordCompleteFor(Constants.ENTRY_NODE, count, rt, error);
            }
        }
        
        // 执行注册的 slot exit 的 onExit 回调方法
        Collection<ProcessorSlotExitCallback> exitCallbacks = StatisticSlotCallbackRegistry.getExitCallbacks();
        for (ProcessorSlotExitCallback handler : exitCallbacks) {
            handler.onExit(context, resourceWrapper, count, args);
        }

        fireExit(context, resourceWrapper, count, args);
    }
}
```

## AuthoritySlot
进入资源前，AuthoritySlot 会对请求来源进行白/黑名单检查（AuthorityRule），是则通过，否则拒绝

```java
public class AuthoritySlot {

    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count, boolean prioritized, Object... args)
            throws Throwable {
        // 进入资源前进行来源白/黑名单检查
        checkBlackWhiteAuthority(resourceWrapper, context);
        fireEntry(context, resourceWrapper, node, count, prioritized, args);
    }

    @Override
    public void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
        // 退出资源时，什么也不做
        fireExit(context, resourceWrapper, count, args);
    }

    void checkBlackWhiteAuthority(ResourceWrapper resource, Context context) throws AuthorityException {
        List<AuthorityRule> rules = AuthorityRuleManager.getRules(resource.getName());
        if (rules == null) {
            return;
        }

        // 根据配置的 AuthorityRule 检查来源是否在配置的合法来源列表中
        for (AuthorityRule rule : rules) {
            // 如果不在，则抛出异常（代表拒绝）
            if (!AuthorityRuleChecker.passCheck(rule, context)) {
                throw new AuthorityException(context.getOrigin(), rule);
            }
        }
    }
}
```

## SystemSlot
进入资源前，SystemSlot 会检查当前系统情况是否低于配置的系统规则（SystemRule），是则通过，否则拒绝

```java
public class SystemSlot {

    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                      boolean prioritized, Object... args) throws Throwable {
        // 进入资源前，检查当前系统情况是否低于配置的符合规则
        // 1，pass qps 是否小于配置的系统规则
        // 2，调用线程数是否小于配置的系统规则
        // 3，rt 平均值是否小于配置的系统规则
        // 4，系统负载是否小于配置的系统规则
        // 5，系统 cpu 使用率是否小于配置的系统规则
        SystemRuleManager.checkSystem(resourceWrapper, count);
        fireEntry(context, resourceWrapper, node, count, prioritized, args);
    }

    @Override
    public void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
        // 退出资源时，什么也不做
        fireExit(context, resourceWrapper, count, args);
    }

}
```

## FlowSlot
进入资源前，FlowSlot 检查当前资源情况是否低于配置的流控规则（FlowRule），是则通过，否则拒绝
```java
public class FlowSlot {
    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                      boolean prioritized, Object... args) throws Throwable {
        // 进入资源前，判断当前资源情况是否低于配置的流控规则
        checkFlow(resourceWrapper, context, node, count, prioritized);

        fireEntry(context, resourceWrapper, node, count, prioritized, args);
    }

    @Override
    public void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
        // 退出资源时，什么也不做
        fireExit(context, resourceWrapper, count, args);
    }
}
```

## DefaultCircuitBreakerSlot
进入资源前，DefaultCircuitBreakerSlot 会检查当前资源情况是否低于配置的熔断规则（DefaultCircuitBreakerRule），是则通过，否则拒绝
```java
public class DefaultCircuitBreakerSlot {
    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                      boolean prioritized, Object... args) throws Throwable {
        // 进入资源前，判断当前资源情况是否低于配置的熔断规则
        performChecking(context, resourceWrapper);

        fireEntry(context, resourceWrapper, node, count, prioritized, args);
    }

    @Override
    public void exit(Context context, ResourceWrapper r, int count, Object... args) {
        Entry curEntry = context.getCurEntry();
        // 进入资源异常时，则直接退出
        if (curEntry.getBlockError() != null) {
            fireExit(context, r, count, args);
            return;
        }

        if (DegradeRuleManager.hasConfig(r.getName())) {
            fireExit(context, r, count, args);
            return;
        }

        List<CircuitBreaker> circuitBreakers = DefaultCircuitBreakerRuleManager.getDefaultCircuitBreakers(r.getName());

        if (circuitBreakers == null || circuitBreakers.isEmpty()) {
            fireExit(context, r, count, args);
            return;
        }

        // 进入资源正常，则更新熔断器状态
        if (curEntry.getBlockError() == null) {
            // passed request
            for (CircuitBreaker circuitBreaker : circuitBreakers) {
                circuitBreaker.onRequestComplete(context);
            }
        }

        fireExit(context, r, count, args);
    }
}
```

## DegradeSlot
进入资源前，DegradeSlot 会检查当前资源情况是否低于配置的降级规则（DegradeRule），是则通过，否则拒绝
```java
public class DegradeSlot {
    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                      boolean prioritized, Object... args) throws Throwable {
        // 进入资源前，判断当前资源情况是否低于配置的降级规则
        performChecking(context, resourceWrapper);

        fireEntry(context, resourceWrapper, node, count, prioritized, args);
    }

    @Override
    public void exit(Context context, ResourceWrapper r, int count, Object... args) {
        Entry curEntry = context.getCurEntry();
        // 进入资源异常时，则直接退出
        if (curEntry.getBlockError() != null) {
            fireExit(context, r, count, args);
            return;
        }
        List<CircuitBreaker> circuitBreakers = DegradeRuleManager.getCircuitBreakers(r.getName());
        if (circuitBreakers == null || circuitBreakers.isEmpty()) {
            fireExit(context, r, count, args);
            return;
        }

        // 进入资源正常，则更新熔断器状态
        if (curEntry.getBlockError() == null) {
            // passed request
            for (CircuitBreaker circuitBreaker : circuitBreakers) {
                circuitBreaker.onRequestComplete(context);
            }
        }

        fireExit(context, r, count, args);
    }
}
```

# 退出资源

上述就是 SphU.entry （即进入资源）的大概逻辑了，那 entry.exit（即退出资源）是又是什么样的流程呢？

在 sentinel 中申请多个 entry 时，entry 之间会形成一个双向链表。在 entry.exit 时，此链表用于保证退出顺序与申请顺序相反，并且能确保所有 entry 退出完毕后正常释放相关对象。

```java
class CtEntry {

    /**
     * 创建 entry 时调用此方法
     */
    private void setUpEntryFor(Context context) {
        /*
         * 申请多个 entry 时，entry.parent 会指向上一个申请的 entry
         * 同时 context.curEntry 指向当前 entry，形成下面的 entry 双向链表
         * 
         *                            context.curEntry
         *                                  |
         *                                  ↓
         *            +------+ child   +------+
         *            |      | ---->   |      |
         * null <---- |entry1|         |entry2| ------> null
         *     parent |      | <----   |      | child
         *            +------+ parent  +------+
         */
        this.parent = context.getCurEntry();
        if (parent != null) {
            ((CtEntry) parent).child = this;
        }
        context.setCurEntry(this);
    }

    /**
     * 退出 entry 时调用此方法
     */
    protected void exitForContext(Context context, int count, Object... args) throws ErrorEntryFreeException {
        if (context != null) {
            // Null context should exit without clean-up.
            if (context instanceof NullContext) {
                return;
            }

            // 当进入多个资源时，context.curEntry 总是指向最后申请资源的 entry
            // 如果此时 context.curEntry 不等于当前 entry，代表当前退出的不是最后申请的 entry
            // 违背了 entry 释放顺序原则（与进入顺序相反），此时会抛出进入和退出的顺序不一致的异常
            if (context.getCurEntry() != this) {
                String curEntryNameInContext = context.getCurEntry() == null ? null
                        : context.getCurEntry().getResourceWrapper().getName();
                // 释放之前申请的 entry 
                CtEntry e = (CtEntry) context.getCurEntry();
                while (e != null) {
                    e.exit(count, args);
                    e = (CtEntry) e.parent;
                }
                String errorMessage = String.format("The order of entry exit can't be paired with the order of entry"
                                + ", current entry in context: <%s>, but expected: <%s>", curEntryNameInContext,
                        resourceWrapper.getName());
                throw new ErrorEntryFreeException(errorMessage);
            } else {
                // 调用 processSlotChain 执行各 slot 的 exit 逻辑
                if (chain != null) {
                    chain.exit(context, resourceWrapper, count, args);
                }
                // Go through the existing terminate handlers (associated to this invocation).
                callExitHandlersAndCleanUp(context);

                // 本 entry 退出，context.curEntry 指向 entry.parent
                // 用于退出上一个申请的 entry
                context.setCurEntry(parent);
                if (parent != null) {
                    ((CtEntry) parent).child = null;
                }
                
                // 如果 parent 为空，则代表本次申请的 entry 全部退出完毕
                // 释放 context 对象
                if (parent == null) {
                    if (ContextUtil.isDefaultContext(context)) {
                        ContextUtil.exit();
                    }
                }
                // Clean the reference of context in current entry to avoid duplicate exit.
                clearEntryContext();
            }
        }
    }
}
```

以上就是 sentinel 进入、退出资源的大概逻辑了，下章我们将介绍数据如何统计以及对应的数据结构。