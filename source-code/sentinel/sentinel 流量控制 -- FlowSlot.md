# sentinel 流量控制 -- FlowSlot



## FlowSlot -- 负责流量控制的责任链 processor

com.alibaba.csp.sentinel.slots.block.flow.FlowSlot

FlowSlot 是 ProcessorSlot 责任链上的实现之一，负责流量控制功能。

1. entry 方法: 责任链 slot 入口方法，对于 FlowSlot，入口处做了规则验证，然后调用下一个 slot 的 entry 方法。
2. exit 方法: 责任链 slot 出口方法，对于 FlowSlot，出口处没什么要做的，直接调用下一个 slot 的 exit 方法即可。
3. ruleProvider 方法: 函数映射，资源名 -> 资源对应的流控规则。
4. checkFlow 方法: 使用 FlowRuleChecker 进行规则验证，核心方法。

```java
@Spi(order = Constants.ORDER_FLOW_SLOT)
public class FlowSlot extends AbstractLinkedProcessorSlot<DefaultNode> {

    /**
     * 工具
     */
    private final FlowRuleChecker checker;

    public FlowSlot() {
        this(new FlowRuleChecker());
    }

    FlowSlot(FlowRuleChecker checker) {
        AssertUtil.notNull(checker, "flow checker should not be null");
        this.checker = checker;
    }

    /**
     * 责任链 slot 入口方法，对于 FlowSlot，入口处做了规则验证，然后调用下一个 slot 的 entry 方法
     */
    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                      boolean prioritized, Object... args) throws Throwable {
        // 规则验证
        checkFlow(resourceWrapper, context, node, count, prioritized);

        // 触发下一个 slot 的 entry 方法
        fireEntry(context, resourceWrapper, node, count, prioritized, args);
    }

    /**
     * 规则验证
     */
    void checkFlow(ResourceWrapper resource, Context context, DefaultNode node, int count, boolean prioritized)
        throws BlockException {
        checker.checkFlow(ruleProvider, resource, context, node, count, prioritized);
    }

    /**
     * 责任链 slot 出口方法，对于 FlowSlot，出口处没什么要做的，直接调用下一个 slot 的 exit 方法即可
     */
    @Override
    public void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
        // 触发下一个 slot 的 exit 方法
        fireExit(context, resourceWrapper, count, args);
    }

    /**
     * 函数映射，资源名 -> 资源对应的流控规则
     */
    private final Function<String, Collection<FlowRule>> ruleProvider = new Function<String, Collection<FlowRule>>() {
        @Override
        public Collection<FlowRule> apply(String resource) {
            // Flow rule map should not be null.
            Map<String, List<FlowRule>> flowRules = FlowRuleManager.getFlowRuleMap();
            return flowRules.get(resource);
        }
    };
```



## FlowRuleChecker -- 流量控制规则验证

com.alibaba.csp.sentinel.slots.block.flow.FlowRuleChecker#checkFlow

1. 根据资源名获取流控规则。
2. 遍历流控规则检查是否放行，任何一个流控规则不放行则抛异常。

```java
    public void checkFlow(Function<String, Collection<FlowRule>> ruleProvider, ResourceWrapper resource,
                          Context context, DefaultNode node, int count, boolean prioritized) throws BlockException {
        if (ruleProvider == null || resource == null) {
            return;
        }
        // 根据资源名获取流控规则
        Collection<FlowRule> rules = ruleProvider.apply(resource.getName());
        if (rules != null) {
            // 遍历流控规则检查是否放行，任何一个流控规则不放行则抛异常
            for (FlowRule rule : rules) {
                if (!canPassCheck(rule, context, node, count, prioritized)) {
                    throw new FlowException(rule.getLimitApp(), rule);
                }
            }
        }
    }
```

### canPassCheck -- 流控规则验证

com.alibaba.csp.sentinel.slots.block.flow.FlowRuleChecker#canPassCheck

1. 流控规则分两种，一种是集群模式调用 passClusterCheck，一种是单机模式调用 passLocalCheck。

```java
    public boolean canPassCheck(/*@NonNull*/ FlowRule rule, Context context, DefaultNode node, int acquireCount,
                                                    boolean prioritized) {
        String limitApp = rule.getLimitApp();
        if (limitApp == null) {
            return true;
        }

        // 流控规则是集群模式
        if (rule.isClusterMode()) {
            return passClusterCheck(rule, context, node, acquireCount, prioritized);
        }

        // 流控规则是单机模式
        return passLocalCheck(rule, context, node, acquireCount, prioritized);
    }
```

### passLocalCheck - 单机模式流控规则验证

com.alibaba.csp.sentinel.slots.block.flow.FlowRuleChecker#passLocalCheck

1. 根据流控模式选择 node。
2. 根据流控效果验证规则。快速失败 / Warm Up / 排队等待。

```java
    private static boolean passLocalCheck(FlowRule rule, Context context, DefaultNode node, int acquireCount,
                                          boolean prioritized) {
        // 根据流控模式选择 node。
        Node selectedNode = selectNodeByRequesterAndStrategy(rule, context, node);
        if (selectedNode == null) {
            return true;
        }

        // 根据流控效果验证规则。快速失败 / Warm Up / 排队等待
        return rule.getRater().canPass(selectedNode, acquireCount, prioritized);
    }
```



## 流控效果 -- 快速失败 DefaultController

com.alibaba.csp.sentinel.slots.block.flow.controller.DefaultController#canPass

实现是 DefaultController，效果: 当 token 数超出设置的阈值后，直接不通过。token 数: 阈值支持 QPS 和并发线程数，token 根据阈值类型确定是 qps 数还是线程数。

1. 获取当前 node 的 token 数。
2. 如果当前 token 数 + 所需 token 数超出阈值，那么流控不通过。未超出阈值流控通过。

```java
    @Override
    public boolean canPass(Node node, int acquireCount, boolean prioritized) {
        // 获取当前 node 的 token 数
        int curCount = avgUsedTokens(node);
        // 如果当前 token 数 + 所需 token 数超出阈值，那么流控不通过
        if (curCount + acquireCount > count) {
            // 省略优先级逻辑...
            return false;
        }
        // 未超出阈值，返回成功
        return true;
    }
```

