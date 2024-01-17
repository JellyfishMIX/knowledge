# zookeeper session 机制



## 说明

1. 本文基于 zookeeper 3.8 编写。
2. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 简介

zookeeper 网络通信，维护的连接使用 session 机制。



## 初次接收到请求时建立连接

1. zookeeper server 与 client 间使用 NIO 机制进行网络通信，server node 与 node 间使用 BIO 机制进行网络通信。
2. NIO 机制接收到请求时，如果还未初始化，则建立连接。如果已初始化，则读取数据。

```java
    private void readPayload() throws IOException, InterruptedException, ClientCnxnLimitException {
        if (incomingBuffer.remaining() != 0) { // have we read length bytes?
            int rc = sock.read(incomingBuffer); // sock is non-blocking, so ok
            if (rc < 0) {
                handleFailedRead();
            }
        }

        if (incomingBuffer.remaining() == 0) { // have we read length bytes?
            incomingBuffer.flip();
            packetReceived(4 + incomingBuffer.remaining());
            if (!initialized) {
                // 还未初始化，建立连接
                readConnectRequest();
            } else {
                // 已初始化，读取数据
                readRequest();
            }
            lenBuffer.clear();
            incomingBuffer = lenBuffer;
        }
    }

	private void readConnectRequest() throws IOException, InterruptedException, ClientCnxnLimitException {
        if (!isZKServerRunning()) {
            throw new IOException("ZooKeeperServer not running");
        }
        // 处理连接请求
        zkServer.processConnectRequest(this, incomingBuffer);
        // 设置初始化标记
        initialized = true;
    }


	public void processConnectRequest(ServerCnxn cnxn, ByteBuffer incomingBuffer)
        throws IOException, ClientCnxnLimitException {
        // ...
        // 构建 Session
        long id = createSession(cnxn, passwd, sessionTimeout);
        // ...
    }
```

触发 Session 创建的流程用图来表示是这样的:

![session-12.png](https://image-hosting.jellyfishmix.com/20231117120410.png)



## 建立连接时创建 Session

org.apache.zookeeper.server.ZooKeeperServer#createSession

```java
    long createSession(ServerCnxn cnxn, byte[] passwd, int timeout) {
        if (passwd == null) {
            // Possible since it's just deserialized from a packet on the wire.
            passwd = new byte[0];
        }
        // 生成 sessionId
        long sessionId = sessionTracker.createSession(timeout);
        Random r = new Random(sessionId ^ superSecret);
        r.nextBytes(passwd);
        ByteBuffer to = ByteBuffer.allocate(4);
        to.putInt(timeout);
        cnxn.setSessionId(sessionId);
        Request si = new Request(cnxn, sessionId, 0, OpCode.createSession, to, null);
        submitRequest(si);
        return sessionId;
    }
```



## sessionId-session 映射关系

org.apache.zookeeper.server.SessionTrackerImpl

```java
public class SessionTrackerImpl extends ZooKeeperCriticalThread implements SessionTracker {
	protected final ConcurrentHashMap<Long, SessionImpl> sessionsById = new ConcurrentHashMap<Long, SessionImpl>();
}
```



## session 过期机制

zookeeper session 过期时间可配置，使用独立的清理线程执行过期，为了防止大量不一样的过期时间使清理线程执行频繁，zookeeper 采用区间的方式分配过期时间，即 [0, 2000)ms 过期时间为 2000ms, [2000, 4000)ms 过期时间为 4000ms...以此类推

### 获取过期时间

org.apache.zookeeper.server.ExpiryQueue#roundToNextInterval

```java
    /**
     * 获取过期时间
     * @param time 设置的过期时间
     * @return 实际过期时间
     */
    private long roundToNextInterval(long time) {
        // expirationInterval 就是 tickTime，默认是 2000 毫秒，也就是每隔 2s 进行一次 Session 检查
        return (time / expirationInterval + 1) * expirationInterval;
    }
```

举例:

假设 tickTime(即 expirationInterval) 为 2000ms:

1. 假设 Session1 设置的过期时间是 1900ms，也就是 1900ms 后过期，那么代入到公式里得到的 Map 的 key 就是`(1900/2000 + 1) * 2000 = 2000ms`

2. 假设 Session2 设置的过期时间是 3000ms，也就是 3000ms 后过期，那么代入到公式里得到的 Map 的 key 就是`(3000/2000 + 1) * 2000 = 4000ms`

3. 假设 Session3 设置的过期时间是 5000ms，也就是 5000ms 后过期，那么代入到公式里得到的 Map 的 key 就是`(5000/2000 + 1) * 2000 = 6000ms`

### 执行 Session 过期

org.apache.zookeeper.server.SessionTrackerImpl#run

1. 获取当前时间举例下一个过期时间点的剩余时间，如果还有剩余时间则 SessionTrackerImpl 线程阻塞休眠。
2. 已达到下一个过期时间点，移除过期的 session。

```java
    @Override
    public void run() {
        try {
            while (running) {
                // 获取当前时间举例下一个过期时间点的剩余时间，如果还有剩余时间则 SessionTrackerImpl 线程阻塞休眠
                long waitTime = sessionExpiryQueue.getWaitTime();
                if (waitTime > 0) {
                    Thread.sleep(waitTime);
                    continue;
                }

                // 已达到下一个过期时间点，移除过期的 session
                for (SessionImpl s : sessionExpiryQueue.poll()) {
                    ServerMetrics.getMetrics().STALE_SESSIONS_EXPIRED.add(1);
                    setSessionClosing(s.sessionId);
                    expirer.expire(s);
                }
            }
        } catch (InterruptedException e) {
            handleException(this.getName(), e);
        }
        LOG.info("SessionTrackerImpl exited loop!");
    }
```

