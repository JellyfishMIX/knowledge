# zookeeper watcher 机制



## 说明

1. 本文基于 zookeeper 3.8 编写。
2. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## Watcher 接口

org.apache.zookeeper.Watcher

使用了观察者模式，Watcher 即观察者，用于数据发生变化(产生事件)时，感知变化。

```java
public interface Watcher {
    void process(WatchedEvent event);
}
```

变化即事件

org.apache.zookeeper.WatchedEvent

```java
public class WatchedEvent {
    /**
     * 事件枚举
     */
    private final EventType eventType;
    /**
     * 事件关联的节点路径
     */
    private String path;
}

enum EventType {
    /**
     * Watcher 监听的数据节点被创建时
     */
    NodeCreated(1),
    /**
     * Watcher 监听的数据节点被删除时
     */
    NodeDeleted(2),
    /**
     * Watcher 监听的数据节点内容发生变更时（无论内容数据是否变化）
     */
    NodeDataChanged(3),
    /**
     * Watcher 监听的数据节点的子节点列表发生变更时
     */
    NodeChildrenChanged(4);
}
```



## WatchManager 工厂类 -- 用于管理 watcher

org.apache.zookeeper.server.watch.WatchManager

1. 维护两个 map, 即两组映射关系。增删改时需要同时维护这两个 map，因此增删改的方法加了 synchronized 修饰。
   1. watchTable，key: 数据节点路径，value: path 对应的 watcher。有了这个结构，当节点有变动的时候可以直接从 map 里 get 出所有监听此节点的 Watcher 集合，然后 for 循环遍历逐个通知。
   2. watch2Paths，key: watch, value: watch 关联的 path。辅助结构，用于根据 watch 找到关联的 path，当一个 watcher 断开连接时，可以高效地从 watchTable 中移除 path-watcher 映射关系。
2. 
3. 触发 watcher
   1. 加锁收集需要通知的 watcher，解锁后再执行通知，减小锁范围提高并发性能。
   2. 结尾统计信息。

```java
public class WatchManager implements IWatchManager {
    /**
     * key: 数据节点路径，value: path 对应的 watcher
     * 有了这个结构，当节点有变动的时候可以直接从 map 里 get 出所有监听此节点的 Watcher 集合，然后 for 循环遍历逐个通知
     */
	private final HashMap<String, HashSet<Watcher>> watchTable = new HashMap<String, HashSet<Watcher>>();
    
    /**
     * key: watch, value: watch 关联的 path
     * 辅助结构，用于根据 watch 找到关联的 path，当一个 watcher 断开连接时，可以高效地从 watchTable 中移除 path-watcher 映射关系
     */
    private final HashMap<Watcher, HashSet<String>> watch2Paths = new HashMap<Watcher, HashSet<String>>();
    
    /**
     * 添加 watcher
     */
    public synchronized boolean addWatch(String path, Watcher watcher) {
        // ...
        Set<Watcher> list = watchTable.get(path);
        if (list == null) {
            // don't waste memory if there are few watches on a node
            // rehash when the 4th entry is added, doubling size thereafter
            // seems like a good compromise
            list = new HashSet<>(4);
            watchTable.put(path, list);
        }
        list.add(watcher);
        Set<String> paths = watch2Paths.get(watcher);
        if (paths == null) {
            // cnxns typically have many watches, so use default cap here
            paths = new HashSet<>();
            watch2Paths.put(watcher, paths);
        }
        watcherModeManager.setWatcherMode(watcher, path, watcherMode);
        return paths.add(path);
    }

    /**
     * 移除 watcher
     */
    @Override
    public synchronized void removeWatcher(Watcher watcher) {
        Set<String> paths = watch2Paths.remove(watcher);
        if (paths == null) {
            return;
        }
        for (String p : paths) {
            Set<Watcher> list = watchTable.get(p);
            if (list != null) {
                list.remove(watcher);
                if (list.isEmpty()) {
                    watchTable.remove(p);
                }
            }
            watcherModeManager.removeWatcher(watcher, p);
        }
    }

    /**
     * 触发 watcher
     */
    @Override
    public WatcherOrBitSet triggerWatch(String path, EventType type, WatcherOrBitSet supress) {
        WatchedEvent e = new WatchedEvent(type, KeeperState.SyncConnected, path);
        Set<Watcher> watchers = new HashSet<>();
        PathParentIterator pathParentIterator = getPathParentIterator(path);
        // 加锁收集需要通知的 watcher，解锁后再执行通知，减小锁范围提高并发性能
        synchronized (this) {
            for (String localPath : pathParentIterator.asIterable()) {
                Set<Watcher> thisWatchers = watchTable.get(localPath);
                if (thisWatchers == null || thisWatchers.isEmpty()) {
                    continue;
                }
                Iterator<Watcher> iterator = thisWatchers.iterator();
                while (iterator.hasNext()) {
                    Watcher watcher = iterator.next();
                    WatcherMode watcherMode = watcherModeManager.getWatcherMode(watcher, localPath);
                    if (watcherMode.isRecursive()) {
                        if (type != EventType.NodeChildrenChanged) {
                            watchers.add(watcher);
                        }
                    } else if (!pathParentIterator.atParentPath()) {
                        watchers.add(watcher);
                        if (!watcherMode.isPersistent()) {
                            iterator.remove();
                            Set<String> paths = watch2Paths.get(watcher);
                            if (paths != null) {
                                paths.remove(localPath);
                            }
                        }
                    }
                }
                if (thisWatchers.isEmpty()) {
                    watchTable.remove(localPath);
                }
            }
        }
        if (watchers.isEmpty()) {
            if (LOG.isTraceEnabled()) {
                ZooTrace.logTraceMessage(LOG, ZooTrace.EVENT_DELIVERY_TRACE_MASK, "No watchers for " + path);
            }
            return null;
        }

        // 执行通知 watcher
        for (Watcher w : watchers) {
            if (supress != null && supress.contains(w)) {
                continue;
            }
            w.process(e);
        }

        // 统计信息
        switch (type) {
            case NodeCreated:
                ServerMetrics.getMetrics().NODE_CREATED_WATCHER.add(watchers.size());
                break;

            case NodeDeleted:
                ServerMetrics.getMetrics().NODE_DELETED_WATCHER.add(watchers.size());
                break;

            case NodeDataChanged:
                ServerMetrics.getMetrics().NODE_CHANGED_WATCHER.add(watchers.size());
                break;

            case NodeChildrenChanged:
                ServerMetrics.getMetrics().NODE_CHILDREN_WATCHER.add(watchers.size());
                break;
            default:
                // Other types not logged.
                break;
        }

        return new WatcherOrBitSet(watchers);
    }
}
```

