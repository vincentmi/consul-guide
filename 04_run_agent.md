# 运行Agent

完成Consul的安装后,必须运行agent. agent可以运行为server或client模式.每个数据中心至少必须拥有一台server . 建议在一个集群中有3或者5个server.部署单一的server,在出现失败时会不可避免的造成数据丢失.

其他的agent运行为client模式.一个client是一个非常轻量级的进程.用于注册服务,运行健康检查和转发对server的查询.agent必须在集群中的每个主机上运行.

查看启动数据中心的细节请查看[这里](https://www.consul.io/docs/guides/bootstrapping.html).

## 启动 Agent

为了更简单,现在我们将启动Consul agent的开发模式.这个模式快速和简单的启动一个单节点的Consul.这个模式不能用于生产环境,因为他不持久化任何状态.

```
[root@hdp2 ~]# consul agent -dev
==> Starting Consul agent...
==> Starting Consul agent RPC...
==> Consul agent running!
         Node name: 'hdp2'
        Datacenter: 'dc1'
            Server: true (bootstrap: false)
       Client Addr: 127.0.0.1 (HTTP: 8500, HTTPS: -1, DNS: 8600, RPC: 8400)
      Cluster Addr: 10.0.0.52 (LAN: 8301, WAN: 8302)
    Gossip encrypt: false, RPC-TLS: false, TLS-Incoming: false
             Atlas: <disabled>

==> Log data will now stream in as it occurs:

    2016/08/17 15:20:41 [INFO] serf: EventMemberJoin: hdp2 10.0.0.52
    2016/08/17 15:20:41 [INFO] serf: EventMemberJoin: hdp2.dc1 10.0.0.52
    2016/08/17 15:20:41 [INFO] raft: Node at 10.0.0.52:8300 [Follower] entering Follower state
    2016/08/17 15:20:41 [INFO] consul: adding LAN server hdp2 (Addr: 10.0.0.52:8300) (DC: dc1)
    2016/08/17 15:20:41 [INFO] consul: adding WAN server hdp2.dc1 (Addr: 10.0.0.52:8300) (DC: dc1)
    2016/08/17 15:20:41 [ERR] agent: failed to sync remote state: No cluster leader
    2016/08/17 15:20:42 [WARN] raft: Heartbeat timeout reached, starting election
    2016/08/17 15:20:42 [INFO] raft: Node at 10.0.0.52:8300 [Candidate] entering Candidate state
    2016/08/17 15:20:42 [DEBUG] raft: Votes needed: 1
    2016/08/17 15:20:42 [DEBUG] raft: Vote granted from 10.0.0.52:8300. Tally: 1
    2016/08/17 15:20:42 [INFO] raft: Election won. Tally: 1
    2016/08/17 15:20:42 [INFO] raft: Node at 10.0.0.52:8300 [Leader] entering Leader state
    2016/08/17 15:20:42 [INFO] raft: Disabling EnableSingleNode (bootstrap)
    2016/08/17 15:20:42 [DEBUG] raft: Node 10.0.0.52:8300 updated peer set (2): [10.0.0.52:8300]
    2016/08/17 15:20:42 [INFO] consul: cluster leadership acquired
    2016/08/17 15:20:42 [DEBUG] consul: reset tombstone GC to index 2
    2016/08/17 15:20:42 [INFO] consul: member 'hdp2' joined, marking health alive
    2016/08/17 15:20:42 [INFO] consul: New leader elected: hdp2
    2016/08/17 15:20:43 [INFO] agent: Synced service 'consul'
```

如你所见,Consul Agent 启动并输出了一些日志数据.从这些日志中你可以看到,我们的agent运行在server模式并且声明作为一个集群的领袖.额外的本地镀锌被标记为一个健康的成员.

