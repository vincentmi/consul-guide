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

> OS X用户注意: Consul 使用你的主机hostname作为默认的节点名字.如果你的主机名包含时间,到这个节点的DNS查询将不会工作.为了避免这个情况,使用`-node`参数来明确的设置node名.

## 集群成员

新开一个终端窗口运行`consul members`, 你可以看到Consul集群的成员.下一节我们将讲到加入集群.现在你应该只能看到一个成员,就是你自己:

```
[root@hdp2 ~]# consul members
Node  Address         Status  Type    Build  Protocol  DC
hdp2  10.0.0.52:8301  alive   server  0.6.4  2         dc1
```

这个输出显示我们自己的节点.运行的地址,健康状态,自己在集群中的角色,版本信息.添加`-detailed`选项可以查看到额外的信息.

`members`命令的输出是基于[gossip](https://www.consul.io/docs/internals/gossip.html)协议是最终一致的.意味着,在任何时候,通过你本地agent看到的结果可能不是准确匹配server的状态.为了查看到一致的信息,使用HTTP API\(将自动转发\)到Consul Server上去进行查询:

```
[root@hdp2 ~]#  curl localhost:8500/v1/catalog/nodes
[{"Node":"hdp2","Address":"10.0.0.52","TaggedAddresses":{"wan":"10.0.0.52"},"CreateIndex":3,"ModifyIndex":4}]
```

除了HTTP API ,DNS 接口也可以用来查询节点.注意,你必须确定将你的DNS查询指向Consul agent的DNS服务器,这个默认运行在 `8600`端口.DNS条目的格式\(例如:"Armons-MacBook-Air.node.consul"\)将在后面讲到.

```
$ dig @127.0.0.1 -p 8600 Armons-MacBook-Air.node.consul
...

;; QUESTION SECTION:
;Armons-MacBook-Air.node.consul.    IN  A

;; ANSWER SECTION:
Armons-MacBook-Air.node.consul. 0 IN    A   172.20.20.11
```

## 停止Agent

你可以使用Ctrl-C 优雅的关闭Agent. 中断Agent之后你可以看到他离开了集群并关闭.

在退出中,Consul提醒其他集群成员,这个节点离开了.如果你强行杀掉进程.集群的其他成员应该能检测到这个节点失效了.当一个成员离开,他的服务和检测也会从目录中移除.当一个成员失效了,他的健康状况被简单的标记为危险,但是不会从目录中移除.Consul会自动尝试对失效的节点进行重连.允许他从某些网络条件下恢复过来.离开的节点则不会再继续联系.

此外,如果一个agent作为一个服务器,一个优雅的离开是很重要的,可以避免引起潜在的可用性故障影响达成[一致性协议](https://www.consul.io/docs/internals/consensus.html).

查看[这里](https://www.consul.io/docs/internals/consensus.html)了解添加和移除server.

