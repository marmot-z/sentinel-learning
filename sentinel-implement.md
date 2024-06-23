# 自行实现 sentinel

## 简介
sentinel-core 模块提供了资源定义、规则定义以及流量拦截等功能。其可以与应用集成后独立使用，是 sentinel 的核心基础。

## 自己如何实现一个 sentinel-core 
在开始阅读 sentinel-core 模块代码前，我们先尝试自行实现下 sentinel 核心功能，了解实现过程中可能遇见的困难可以更好帮助理解 sentinel 中的部分代码实现。

### 目标
阅读 sentinel 官网文档，我们可以得知 sentinel 工作模式如下：
1. 定义规则（资源单位时间内可以被访问的次数等）
2. 在进入资源前根据规则判断是否可以访问
3. 退出资源后，清理相关统计

这里有两个难点：
1. 如何统计资源的访问量
2. 如何根据统计数据，检查规则是否通过

### 实现
想要对一个资源进行保护，一般是在访问资源前判断是否满足访问要求（比如：1s 内访问量不能超过 10 次）。
如果满足则可以获取凭证访问资源，并且在访问后释放相关的凭证；如果不满足则获取凭证为 null 或抛出异常。 如同 java 中的锁（Lock）工作模式一般。
伪代码如下：
```java
class Main  {
    public static void main(String[] args) {
        // 配置访问规则
        initResourceAccessRules();

        // 通过 resourceName（资源名） 和 origin（调用源）获取资源访问凭证
        ResourceAccessToken token;
        try {
            token = ResourceAccessTokenManager.acquireToken(resourceName, origin);
            // 执行业务逻辑
            // ....
        } catch (BlockException e) {
            // 获取资源的时候被限流
            e.printStackTrace();
            System.err.println("访问资源被限流，资源名:" + resourceName);
        } finally {
            // 回收凭证
            if (token != null) {
                ResourceAccessTokenManager.release(token);
            }
        }
    }
}
```

#### 获取访问凭证
在获取凭证前，我们需要判断当前资源的访问情况是否小于规则定义的上限，才能起到保护资源的目的。假设规则类型有多种：
流控规则、降级规则、授权规则、系统规则等，需要所有规则验证通过才可以获取凭证。伪代码如下：

```java
class ResourceAccessTokenManager {
    public ResourceAccessToken acquireToken(String resourceName) {
        // 获取（或创建）资源对应的统计信息，用于后续规则的检查
        StatisticsInfo statisticsInfo = ResourceStatistician.getStatisticsInfo(resourceName);
        
        try {
            // 检查资源的不同规则
            // 实际运行中，规则也有优先级之分。此处为减少复杂度，所有规则无优先级之分
            // 规则检查的实现一般是判断已经通过的访问量 + 要访问的量，是否大于规则设置的最大访问量，小则放行，大则拦截
            boolean pass = FlowAccessRuleChecker.check(resourceName, statisticsInfo);
            if (!pass) {
                throw new BlockedException(resourceName + " 流控规则不通过");
            }

            pass = AuorityAccessRuleChecker.check(resourceName, statisticsInfo);
            if (!pass) {
                throw new BlockedException(resourceName + " 验证规则不通过");
            }

            pass = SystemAccessRuleChecker.check(resourceName, statisticsInfo);
            if (!pass) {
                throw new BlockedException(resourceName + " 系统规则不通过");
            }

            pass = DegradeAccessRuleChecker.check(resourceName, statisticsInfo);
            if (!pass) {
                throw new BlockedException(resourceName + " 降级规则不通过");
            }

            // 规则检查通过，可以访问资源，通过的访问量 +1
            statisticsInfo.increasePassCount();
            
            return new DefaultResourceAsscessToken(resourceName, statisticsInfo);
        } catch (BlockedException e) {
            // 资源检查失败，拦截的访问量 +1
            statisticsInfo.increaseBlockedCount();
            throw e;
        }
    }
    
    public void release(ResourceAccessToken token) {
        // 资源访问完毕，通过的访问量 -1
        token.getStatisticsInfo().decreasePassCount();
    }
}
```

在日常工作中，上述的代码可以使用责任链的设计模式来优化（这在很多框架里面都有运用，如：servlet 中的 Filter、Spring MVC 中的 Interceptor）。
优化后的伪代码如下：

```java
class ResourceAccessTokenManager {
    public ResourceAccessToken acquireToken(String resourceName) {
        StatisticsInfo statisticsInfo = ResourceStatistician.getStatisticsInfo(resourceName);
        // 获取或创建资源对应的规则检查链
        RuleCheckerChain chain = RuleCheckerChainManager.get(resourceName);
        
        // chain 内部维护一个单链表，链表内保存着各种 RuleChecker 实现
        // 迭代链表，依次调用 RuleChecker 的 check 方法
        // 当任一 check 方法返回 false 时，chain 执行终止，返回 false
        if (!chain.check(resourceName)) {
            statisticsInfo.increaseBlockedCount();
            throw new BlockedException(resourceName + " 规则不通过");
        }

        statisticsInfo.increasePassCount();
        return new DefaultResourceAsscessToken(resourceName, statisticsInfo);
    }
}
```

#### 如何统计资源的访问量
在上述我们可以看到在规则的检查过程中，需要依赖资源当前的访问统计数据。如何保存、查询资源的统计数据，一个合适的数据结构至关重要。
我们假设资源的访问数据就两种：
1. 当前一秒内正常的访问量；
2. 当前一秒内被拦截的访问量（用于降级、熔断规则的检查）。   

那我们需要创建一个实体，其有两个属性：passCount（正常的访问量），blockedCount（拦截的访问量）。伪代码如下：

```java
class StatisticsStruct {
    /**
     * 当前统计结构的生命时长（ms）
     */
    public static final int LIFETIME_MILLIS = 1000;
    /**
     * 正常的访问量 <br>
     * 在规则检查通过后访问资源前，访问量 +1 <br>
     * 在资源访问完毕，手动释放，访问量 -1
     */
    int passCount;
    /**
     * 被拦截的访问量 <br>
     * 在规则检查不通过时，访问量 +1
     */
    int blockedCount;
    /**
     * 开始统计时的时间戳
     */
    long startTimestamp;
}
```

以上实体就代表近 1s 内的资源访问情况，那如果过了 1s，那我们应该重置里面的数值，并重新开始统计访问量。这需要一个额外的容器来管理
StatisticsStruct ，伪代码如下：

```java
class StatisticsInfo {
    /**
     * 当前的统计信息
     */
    private StatisticsStruct currentStatisticsStruct;

    public void incrementPassCount() { getCurrentStatisticsStruct().passCount++; }
    public void incrementBlockedCount() { getCurrentStatisticsStruct().blockedCount++; }
    public void decrementPassCount() { getCurrentStatisticsStruct().passCount--; }
    public int getPassCount() { return getCurrentStatisticsStruct().passCount; }
    public int getBlockedCount() { return getCurrentStatisticsStruct().blockedCount; }
    
    private StatisticsStruct getCurrentStatisticsStruct() {
        if (currentStatisticsStruct == null) {
            return (this.currentStatisticsStruct = new StatisticsStruct());
        }
        
        // 判断统计结构的开始统计时间是否在 1s 内
        // 是则继续统计，否则创建新的结构体重新开始统计
        long currentMillis = System.currentTimeMillis();
        boolean expired = currentMillis - StatisticsStruct.LIFETIME_MILLIS > currentStatisticsStruct.startTimestamp;
        
        return expired ?
                (this.currentStatisticsStruct = new StatisticsStruct()) :
                currentStatisticsStruct;
    }
}
```
上面这种方式有一个严重的缺陷：就是在新的 1s 时，过去 1s 的统计信息会全部被丢弃，导致统计时的不连贯。如：上半秒内访问了 7 次，下半秒访问了 6 次，近 1s 内应该
是访问了 13 次。但由于上半秒的统计情况被清空，导致只有下半秒的 6 次被统计。统计次数（6）远远小于实际次数（13），造成资源超限访问。

对于这种情况，使用时间窗口可以比较好的解决该问题。假如我们有 4 个统计结构，每 1/4s 创建一个新的统计结构并丢弃一个旧的统计结构，那统计值则会更精确（因为前 3/4s 的统计值仍被保留）。
理论上，同时保存的统计结构越多，更新时间频率越高，则统计值越精准。但是权衡成本和收益，我们此处假设为同时维护 4 个统计结构。那上述伪代码改动如下：

```java
class StatisticsStruct {
    public static final int LIFETIME_MILLIS = 250;
    // ...
}

class StatisticsInfo {
    /**
     * 结构数组 <br>
     * 当一个结构过期的同时，又会创建新的结构。则可以使用原先过期结构的位置上存放新的结构。
     * 所以此处可以使用有限的数组表示时间窗口
     */
    private StatisticsStruct[] array = new StatisticsStruct[4];

    public void incrementPassCount() { getCurrentStatisticsStruct().passCount++; }
    public void incrementBlockedCount() { getCurrentStatisticsStruct().blockedCount++; }
    public void decrementPassCount() { getCurrentStatisticsStruct().passCount--; }
    public int getPassCount() { return getSum(s -> s.passCount); }
    public int getBlockedCount() { return getSum(s -> s.blockedCount); }
    
    private StatisticsStruct getCurrentStatisticsStruct() {
        int currentMillis = System.currentTimeMillis();
        int index = (currentMillis / StatisticsStruct.LIFETIME_MILLIS) % array.length;
        StatisticsStruct struct = array[index];
        
        // 还不存在，创建对应结构
        if (struct == null) {
            return (array[index] = new StatisticsStruct());
        }
        // 结构的声明周期小于当前时间戳，应该废弃，创建新的结构
        else if (struct.startTimestamp + StatisticsStruct.LIFETIME_MILLIS < currentMillis) {
            return (array[index] = new StatisticsStruct());
        }
        // 当前时间大于等于结构开始时间，小于等于结构结束时间，代表正在使用
        else if (struct.startTimestamp <= currentMillis &&
                currentMillis <= struct.startTimestamp + StatisticsStruct.LIFETIME_MILLIS) {
            return array[index];
        }
        // 结构的开始时间大于当前时间戳，这种情况不应该存在
        else if (currentMillis < struct.startTimestamp) {
            throw new IllegalStateException();
        }
    }
    
    private int getSum(Function<StatisticsStruct, Integer> keyExtractor) {
        int currentMillis = System.currentTimeMillis();
        // 近 3/4s 开始的结构体都是有效的
        int validStartMillis = currentMillis - (array.length - 1) * StatisticsStruct.LIFETIME_MILLIS;
        
        // 找到有效结构体的统计量进行汇总
        return Array.stream(array)
                .filter(s -> s.startTimestamp > validStartMillis)
                .mapToInt(keyExtractor)
                .sum()
                .get();
    }
}
```


好了，以上就是我们对 sentinel 核心库的两个难点的自行实现了。接下来我们就来看下 sentinel 中是如何抽象解决这些问题的吧。