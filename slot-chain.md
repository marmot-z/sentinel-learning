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
public class CtSph {
    @Override
    public Entry entry(String name, EntryType type, int count, Object... args) throws BlockException {
        // 根据名称创建 Resource
        StringResourceWrapper resource = new StringResourceWrapper(name, type);
        return entry(resource, count, args);
    }
}
```

在创建完资源之后，接下里就是对资源的统计以及规则的检查了：

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

获取资源对应的 processSlotChain

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

processSlotChain 的执行

```java
public abstract class AbstractLinkedProcessorSlot<T> implements ProcessorSlot<T> {
    
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
TODO 确认 entranceNode 的声明周期？为什么会有 invocation tree？invocation tree 不准的问题？
```java
public class NodeSelectorSlot {
    public void entry(Context context, ResourceWrapper resourceWrapper, Object obj, int count, boolean prioritized, Object... args)
            throws Throwable {
        // 每种 context 下的每个资源对应一个 DefaultNode 实例
        DefaultNode node = map.get(context.getName());
        // double check，线程安全的创建 DefaultNode
        if (node == null) {
            synchronized (this) {
                node = map.get(context.getName());
                if (node == null) {
                    node = new DefaultNode(resourceWrapper, null);
                    /*
                     * 创建调用树，该树在资源被调用时创建，直至程序退出时被销毁
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
}

public class Context {
    public Node getLastNode() {
        // 第一次申请 entry，则资源对应的 defaultNode 挂载在 entranceNode 下
        // 继续申请 entry，则资源对应的 defaultNode 挂载在上一个资源对应的 defaultNode 下
        if (curEntry != null && curEntry.getLastNode() != null) {
            return curEntry.getLastNode();
        } else {
            return entranceNode;
        }
    }
}
``` 

## ClusterSlotBuilder
```java
public class ClusterBuilderSlot {

    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                      boolean prioritized, Object... args)
            throws Throwable {
        // 每种资源对应一个 clusterNode 实例
        if (clusterNode == null) {
            synchronized (lock) {
                if (clusterNode == null) {
                    // Create the cluster node.
                    clusterNode = new ClusterNode(resourceWrapper.getName(), resourceWrapper.getResourceType());
                    // ...
                }
            }
        }
        node.setClusterNode(clusterNode);
        
        // ...

        fireEntry(context, resourceWrapper, node, count, prioritized, args);
    }
}
```