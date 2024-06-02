# sentinel spi

为什么优秀的框架总是提供用户扩展的能力？

我想这是因为业务逻辑是千变万化的，框架中能抽象的只是部分的公共逻辑，在应用接入框架时，难免需要做一些本地化调整。框架一般通过接口（interface）的形式
来提供扩展（用户通过实现这些接口来获得某些能力或完成某些逻辑），那么框架如何直到这些接口实现呢？

一般就两种方式：
- 用户告诉框架接口实现：通过框架提供的 SDK 注册对应的接口实现
- 框架自己来找接口实现：框架按照约定（SPI、包扫描等）方式来发现对应的接口实现

什么意思呢，我们以大名鼎鼎的 Spring 来举例：

```java
// 通过注解 + 包扫描的方式来查找 WebMvcConfigurer 接口实现
@Configuration
public class InterceptorConfig implements WebMvcConfigurer {
    private void addSpringMvcInterceptor(InterceptorRegistry registry) {
        SentinelWebMvcConfig config = new SentinelWebMvcConfig();
        // 通过 registry.addInterceptor 注册 Interceptor 接口实现
        registry.addInterceptor(new SentinelWebInterceptor(config)).addPathPatterns("/**");
    }

}
```

那么 Sentinel 中的扩展中有哪些呢？其又是如何感知到这些扩展实现呢？

# sentinel spi

Java 中提供了 SPI（Service Provider Interface）机制来扩展、增强我们的应用。Sentinel 中定义了自己的 SPI 实现，用于加载对应接口实现。
对应的 sentinel spi 实现代码：

```java
class SpiLoader {
    public void load() {
        // 判断是否已经加载过
        if (!loaded.compareAndSet(false, true)) {
            return;
        }

        // 读取的文件全路径（包含类的全路径名称）
        // 如：/META-INF.services/com.alibaba.csp.sentinel.init.initFunc
        String fullFileName = SPI_FILE_PREFIX + service.getName();
        // 判断使用哪种 classLoader
        ClassLoader classLoader;
        // ...
        
        // 从当前 classPath 读取 jar 包中的 spi 文件
        Enumeration<URL> urls = null;
        try {
            urls = classLoader.getResources(fullFileName);
        } catch (IOException e) {
            fail("Error locating SPI configuration file,filename=" + fullFileName + ",classloader=" + classLoader, e);
        }
        // ...

        // 解析 spi 文件中的内容，并加载为 class
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();

            InputStream in = null;
            BufferedReader br = null;
            try {
                in = url.openStream();
                br = new BufferedReader(new InputStreamReader(in, StandardCharsets.UTF_8));
                String line;
                while ((line = br.readLine()) != null) {
                    // 忽略空行，忽略以 # 开始的注释 容
                    // ...

                    // 根据文本创建对应的 class
                    Class<S> clazz = null;
                    try {
                        clazz = (Class<S>) Class.forName(line, false, classLoader);
                    } catch (ClassNotFoundException e) {
                        fail("class " + line + " not found", e);
                    }

                    // 判断是否重复加载，class 是否为 service 类型子类
                    // ...
                    
                    // 读取 @Spi 注解属性
                    classList.add(clazz);
                    Spi spi = clazz.getAnnotation(Spi.class);
                    String aliasName = spi == null || "".equals(spi.value()) ? clazz.getName() : spi.value();
                    // ...
                }
            } catch (IOException e) {
                fail("error reading SPI configuration file[" + url + "]", e);
            } finally {
                closeResources(in, br);
            }
        }

        sortedClassList.addAll(classList);
        // 根据 class 中定义的 @Spi 注解 order 属性进行排序
        Collections.sort(sortedClassList, new Comparator<Class<? extends S>>() {
            @Override
            public int compare(Class<? extends S> o1, Class<? extends S> o2) {
                // ...
            }
        });
    }
}
```

核心的扩展接口信息如下：

| 接口                        | 接口描述                               | 创建时机                                                                   | 运行时机                    |
|---------------------------|------------------------------------|------------------------------------------------------------------------|-------------------------|
| InitFunc                  | 用于实现自定义的初始化逻辑                      | sentinel 初始化时触发（sentinel core 模块在进入资源时会通过 InitExecutor.doInit 触发初始化动作） | 仅初始化触发一次                |
| MetricExtension           | 用于实现sentinel内部统计的自定义逻辑扩展           | sentinel 初始化时触发                                                        | 进入资源时触发                 |
| SlotChainBuilder          | 用于创建自定义的slot chain                 | 为资源创建 slot chain 实例时触发                                                 | 仅初始化创建一次                |
| ProcessorSlot             | 用于实现自定义的资源进入、退出逻辑                  | 在为资源创建 slot chain 实例时触发                                                | 进入、退出资源时触发              |
| CommandCenter             | 用于commandCenter开始前、开始、结束阶段的自定义逻辑扩展 | 在transport模块初始化时触发                                                     | 仅初始化调用一次                |
| CommandHandler            | 用于实现自定义的请求（commend request）处理      | 在transport模块初始化时触发                                                     | 发生请求（CommandRequest）时触发 |
| CommandHandlerInterceptor | 用于实现自定义的请求（commend request）拦截      | 在transport模块初始化时触发                                                     | 发生请求（CommandRequest）时触发 |
| HeartbeatSender           | 用于实现自定义的心跳发送器                      | sentinel 初始化时触发                                                        | 定时触发                    |

# 接口注册

上述代码中， sentinel 实现了一套自己的 SPI 机制来加载依赖的接口，以保证框架正常的运行。

另外 sentinel 还提供了一些接口方便用户在特定实际去自定义自己的逻辑，我们可以通过接口注册的方式告知 sentinel 我们的接口实现。
核心的扩展接口信息如下：

| 接口                         | 接口描述             | 运行时机                                                       |
|----------------------------|------------------|------------------------------------------------------------|
| ReadableDataSource         | 用于读取外部数据源获取最新的配置 | 1. 在定义时初始化拉取一次<br/>2.后续更新时由外部数据源推送最新规则或定时从外部数据源拉取最新规则      |
| WritableDataSource         | 用于将最新配置写入外部数据源   | 在1.8.7版本中以下动作会触发<br/>- 网关api组更新<br/>- 网关规则更新<br/>- 热点规则更新时 |
| ProcessorSlotEntryCallback | 进入资源时回调方法        | 在 StatisticSlot 进入资源后触发                                    |
| ProcessorSlotExitCallback  | 退出资源时回调方法        | 在 StatisticSlot 退出资源后触发                                    |

使用方法很简单，一般通过对应 Registry 类的静态方法进行注册，我们以 ProcessorSlotEntryCallback 接口为例演示一下基础使用方法：

```java
class Demo {
    public static void main(String[] args) {
        StatisticSlotCallbackRegistry.addEntryCallback("entryCb1", new ProcessorSlotEntryCallback<DefaultNode>() {
            @Override
            public void onPass(Context context, ResourceWrapper resourceWrapper, DefaultNode param, int count, Object... args) throws Exception {
                System.out.println("entry resource success : " + resourceWrapper.getName());
            }
            @Override
            public void onBlocked(BlockException ex, Context context, ResourceWrapper resourceWrapper, DefaultNode param, int count, Object... args) {
                System.out.println("entry resource blocked : " + resourceWrapper.getName());
            }
        });
    }
}
```

# 总结
