## Raft 算法

raft 共识算法的优点在于可以在高效的解决分布式系统中各个节点日志内容一致性问题的同时，也使得集群具备一定的容错能力。即使集群中出现部分节点故障、网络故障等问题，仍可保证其余大多数节点正确的步进。甚至当更多的节点（一般来说超过集群节点总数的一半）出现故障而导致集群不可用时，依然可以保证节点中的数据不会出现错误的结果。



工作原理：

a node can be in 1 of 3 states

- Follower
- Candidate
- Leader

all our nodes start in follower state

 每个处于follower状态的节点，都有一个随机超时时间（150ms and 300ms.），在超时时间之内没有收到leader节点的心跳消息，该follower节点就会变为 candidate 节点，并向其他节点发送投票消息（投字节为leader）

如果该candidate节点收到半数以上的同意票，就可以升级为leader。

系统的所有变更现在都通过领导者实现。

https://www.infoq.cn/article/coreos-analyse-etcd/

https://www.jianshu.com/p/8e4bbe7e276c

