# sentinel 核心流程 -- SphU#entry



## 核心流程

1. 核心就这一行代码 entry = SphU.entry("/hello/world");
2. 触发限流时候会抛出 BlockException

```java
try {
    // 核心就这一行代码
    entry = SphU.entry("/hello/world");
} catch (BlockException e) {
    // ......
}
```

### 核心方法的封装

1. 默认 count 为 1，默认 EntryType 为 OUT 出站，默认资源类型为 COMMON，默认无附加参数。

```java
	public static final Sph sph = new CtSph();

	public static Entry entry(String name) throws BlockException {
        return Env.sph.entry(name, EntryType.OUT, 1, OBJECTS0);
    }

	@Override
    public Entry entry(String name) throws BlockException {
        StringResourceWrapper resource = new StringResourceWrapper(name, EntryType.OUT);
        return entry(resource, 1, OBJECTS0);
    }

	public StringResourceWrapper(String name, EntryType e) {
        super(name, e, ResourceTypeConstants.COMMON);
    }
```



## CtSph#entryWithPriority

com.alibaba.csp.sentinel.CtSph#entryWithPriority

```java
    private Entry entryWithPriority(ResourceWrapper resourceWrapper, int count, boolean prioritized, Object... args)
        throws BlockException {
        // 从当前线程中获取 Context
        Context context = ContextUtil.getContext();
        if (context instanceof NullContext) {
            // The {@link NullContext} indicates that the amount of context has exceeded the threshold,
            // so here init the entry only. No rule checking will be done.
            return new CtEntry(resourceWrapper, null, context);
        }

        // 如果没获取到 Context，就创建一个名为 sentinel_default_context 的 Context，并与当前线程绑定
        if (context == null) {
            // Using default context.
            context = InternalContextUtil.internalEnter(Constants.CONTEXT_DEFAULT_NAME);
        }

        // Global switch is close, no rule checking will do.
        if (!Constants.ON) {
            return new CtEntry(resourceWrapper, null, context);
        }

        // 获取 ProcessorSlot 责任链
        ProcessorSlot<Object> chain = lookProcessChain(resourceWrapper);

        /*
         * Means amount of resources (slot chain) exceeds {@link Constants.MAX_SLOT_CHAIN_SIZE},
         * so no rule checking will be done.
         */
        if (chain == null) {
            return new CtEntry(resourceWrapper, null, context);
        }

        Entry e = new CtEntry(resourceWrapper, chain, context);
        try {
            // 使用 ProcessorSlot 责任链处理资源
            chain.entry(context, resourceWrapper, null, count, prioritized, args);
        } catch (BlockException e1) {
            // 感知到 BlockException 时的处理，entry 进行 exit 退出
            e.exit(count, args);
            throw e1;
        } catch (Throwable e1) {
            // This should not happen, unless there are errors existing in Sentinel internal.
            RecordLog.info("Sentinel unexpected exception", e1);
        }
        return e;
    }
```

