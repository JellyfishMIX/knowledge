# Hashtable，Properties



## Hashtable

Hashtable实现了Map接口，它和HashMap类很相似，但是它支持同步（线程安全）。

像HashMap一样，Hashtable在哈希表中存储键值对 `key - value` 。

像HashMap一样，Hashtable在哈希表中存储键/值对。当使用一个哈希表，要指定用作键的对象，以及要链接到该键的值。

然后，该键经过哈希处理，所得到的散列码被用作存储在该表中值的索引。



## Properties

Properties 继承于 Hashtable，因此支持同步（线程安全）。

表示一个持久的属性集.属性列表中每个键及其对应值都是一个字符串。

被设计用于在一些文件中存储配置和资源。