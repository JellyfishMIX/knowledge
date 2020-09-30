# TCP 三次握手，四次挥手



## 四次挥手

### 流程

![5e9da8dd000178b708320864](https://image-hosting.jellyfishmix.com/20201024134927.jpg)

1. 主动关闭方（active close）先发送 FIN，进入 FIN_WAIT1 状态。

2. 被动关闭方（passive close）收到 FIN，发送 ACK，进入CLOSE_WAIT 状态；主动关闭方收到这个 ACK，进入 FIN_WAIT2 状态。

3. 被动关闭方发送 FIN，进入 LAST_ACK 状态。

4. 主动关闭方收到 FIN，发送 ACK，进入TIME_WAIT 状态，服务端收到 ACK，进入 CLOSE 状态。

5. 主动关闭方 TIME_WAIT 持续 2 倍 MSL 时长，在 linux 体系中大概是 60s，转换成 CLOSE 状态。

#### 能不能发送完 ACK 之后**主动关闭方**跳过 TIME_WAIT 就直接进入 CLOSE 状态呢？

不行的，因为网络原因，主动关闭方的 ACK 可能会发送失败，此时被动关闭方重新发送一次 FIN，在这个时候如果主动关闭方在 TIME_WAIT 状态，还会再发送一次 ACK，从而保证 TCP 协议的可靠性。这也是为什么 TIME_WAIT 会持续 2MSL 的时长的原因：重新发送时，一个FIN＋一个ACK+不定期的延迟时间，大致是在 2MSL 的范围。（1MSL ACK 发给被动关闭方，如果此ACK发送失败，被动关闭方会重新发一次FIN，因此再加1MSL 等待被动关闭方发送潜在的FIN）。



## 引用/参考

[高薪之路--Java面试题精选集 - jiehao - 慕课网](https://www.imooc.com/read/67#catalog)