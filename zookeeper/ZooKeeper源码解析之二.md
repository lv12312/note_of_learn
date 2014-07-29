##ZooKeeper源码浅析之二

###1、快速领导者选举算法

虽然默认的有三种选举算法，但是在最新的版本中，前两种都已经过时，现在只剩下这一种可用。

`org.apache.zookeeper.server.quorum.FastLeaderElection` 类，使用TCP实现的Leader选举算法。使用QuorumCnxManager来管理连接。另外，该算法是基于推的方式实现的。

首先看几个参数：

* finalizeWait 参数决定一个进程需要等待选举结束的超时时间。默认200毫秒。
* maxNotificationInterval 两次同时进行的通知的时间差。影响了网络分裂之后多长时间系统可以继续恢复正常。默认是60秒。