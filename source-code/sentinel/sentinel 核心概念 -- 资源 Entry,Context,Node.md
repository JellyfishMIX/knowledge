# sentinel 核心概念 -- 资源 Entry,Context,Node



## 资源和规则

1. 资源：可以是一个方法、一个接口或一段代码。以一个 HTTP 接口为例，假设我们为 `/a/b/c`这个接口配置了限流 QPS 为 100 的限制，那么 `/a/b/c` 就被称为资源。资源是被 sentinel 保护和管理的对象，sentinel 通过对资源的调用进行限流、熔断降级等控制。
2. 规则：用来定义资源应该遵循的约束条件。在上述示例中，为 `/a/b/c` 接口配置的 QPS 为 100 的限制就是一个规则。sentinel 支持多种规则类型，如流控规则、熔断降级规则以及其他自定义规则。规则可以通过多种方式配置，例如 API、文件、配置中心等。
3. 在 sentinel 中，资源和规则是相互关联的。sentinel 会根据配置的规则，对资源的调用进行相应的控制，从而保证系统的稳定性和高可用。



## Entry -- 对应资源概念

com.alibaba.csp.sentinel.Entry

1. Entry 对应资源的概念，核心属性有两个，一个是 Node 一个是 ResourceWrapper。
   1. ResourceWrapper 承载了具体资源信息。
   2. node 是指标的载体。
2. ResourceWrapper 承载了具体资源信息，包括:
   1. 资源名。
   2. 调用方向。
   3. 资源类型。

```java
public abstract class Entry implements AutoCloseable {

    private static final Object[] OBJECTS0 = new Object[0];

    private final long createTimestamp;
    private long completeTimestamp;

    /**
     * node 是指标的载体
     */
    private Node curNode;
    /**
     * {@link Node} of the specific origin, Usually the origin is the Service Consumer.
     */
    private Node originNode;

    private Throwable error;
    private BlockException blockError;

    /**
     * 资源具体信息
     */
    protected final ResourceWrapper resourceWrapper;
}

public abstract class ResourceWrapper {
    /**
     * 资源名
     */
    protected final String name;

    /**
     * 调用方向
     */
    protected final EntryType entryType;

    /**
     * 资源类型
     * @see com.alibaba.csp.sentinel.ResourceTypeConstants
     */
    protected final int resourceType;
}

public enum EntryType {
    /**
     * Inbound traffic
     * 入站
     */
    IN,
    /**
     * Outbound traffic
     * 出站
     */
    OUT;
}

public final class ResourceTypeConstants {
    public static final int COMMON = 0;
    public static final int COMMON_WEB = 1;
    public static final int COMMON_RPC = 2;
    public static final int COMMON_API_GATEWAY = 3;
    public static final int COMMON_DB_SQL = 4;
}
```



## Context -- 承载资源的上下文

com.alibaba.csp.sentinel.context.Context

每个资源都要附属于 Context 下。

```java
public class Context {

    /**
     * Context name.
     */
    private final String name;

    /**
     * The entrance node of current invocation tree.
     */
    private DefaultNode entranceNode;

    /**
     * Current processing entry.
     */
    private Entry curEntry;
}
```



## ProcessorSlot -- 用于对资源做处理

com.alibaba.csp.sentinel.slotchain.ProcessorSlot

1. processor 的容器，processor 将使用责任链模式用于对资源做处理。

```java
/**
 * A container of some process and ways of notification when the process is finished.
 * processor 的容器，processor 将使用责任链模式用于对资源做处理。
 *
 */
public interface ProcessorSlot<T> {

    /**
     * Entrance of this slot.
     * 进入这个 slot
     */
    void entry(Context context, ResourceWrapper resourceWrapper, T param, int count, boolean prioritized,
               Object... args) throws Throwable;

    /**
     * Means finish of {@link #entry(Context, ResourceWrapper, Object, int, boolean, Object...)}.
     * 当前 slot 处理完毕，触发下一个 slot 的 entry 方法
     */
    void fireEntry(Context context, ResourceWrapper resourceWrapper, Object obj, int count, boolean prioritized,
                   Object... args) throws Throwable;

    /**
     * Exit of this slot.
     * 离开当前 slot
     */
    void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args);

    /**
     * Means finish of {@link #exit(Context, ResourceWrapper, int, Object...)}.
     *
     * 离开当前 slot 结束，触发下一个 slot 的 exit 方法
     */
    void fireExit(Context context, ResourceWrapper resourceWrapper, int count, Object... args);
}
```

### ProcessorSlot 主要实现

1. NodeSelectorSlot: 负责收集资源的路径，并将这些资源的调用路径以树状结构存储起来，主要用于根据调用路径来限流降级。
2. ClusterBuilderSlot: 负责实现集群限流功能。集群限流可以实现跨应用的资源访问控制，ClusterBuilderSlot 将处理请求的流量信息汇报到集群的统计节点 ClusterNode，然后根据集群限流规则决定是否限制资源的访问。
3. StatisticSlot: 负责记录资源的访问统计信息，如通过的请求数，阻塞的请求数，响应时间等。StatisticSlot 将每次资源访问的信息记录在资源的统计节点 StatisticNode 中。这些统计信息是 Sentinel 执行流量控制(如限流, 熔断降级)的重要指标。
4. SystemSlot: 负责系统保护功能，提供基于系统负载，系统平均响应时间和系统入口 qps 的自适应降级保护策略。
5. AuthoritySlot: 负责实现授权规则功能，主要控制不同来源应用的黑白名单访问权限。
6. FlowSlot: 负责流量控制功能，包括针对不同来源的流量限制，基于调用关系的流量控制等。
7. DegradeSlot: 负责熔断降级功能，支持基于异常比例，异常数和响应时间的降级策略。

![责任链模式-02.png](https://image-hosting.jellyfishmix.com/20231208012432.png)



## Node -- 指标的载体

com.alibaba.csp.sentinel.node.Node

1. 获取总请求量。

```java
public interface Node extends OccupySupport, DebugSupport {

    long totalRequest();

    long totalPass();

    long totalSuccess();

    long blockRequest();

    long totalException();

    double passQps();

    double blockQps();

    double totalQps();
    
    double successQps();
}
```

### Node 主要实现

StatisticNode: 基类实现，拥有基础统计功能。

DefaultNode: 默认节点，用于统计一个资源在当前 Context 中的数据，意味着 DefaultNode 是以 Context 和 Entry 为维度进行统计。

EntranceNode：继承自 DefaultNode，也是每一个 Context 的入口节点，用于统计当前 Context 的总体流量数据，统计维度为 Context。

ClusterNode：ClusterNode 保存的是同一个资源的相关统计信息，以资源为维度的，不区分 Context。

### Node 子类关系示意图

![image.png](https://image-hosting.jellyfishmix.com/20231208014653.png)

### Node 子类实现类图

![image-20231208015015056](https://image-hosting.jellyfishmix.com/20231208015015.png)
