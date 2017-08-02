# 健康检查

我们现在看到Consul运行时如此简单.添加节点和服务,查询节点和服务.在这一节.我们将继续添加健康检查到节点和服务.健康检查是服务发现的关键组件.预防使用到不健康的服务.

这一步建立在前一节的Consul集群创建之上.目前你应该有一个包含两个节点的Consul集群.

## 定义检查

和服务类似,一个检查可以通过检查定义或HTTP API请求来注册.

我们将使用和检查定义来注册检查.和服务类似,因为这是建立检查最常用的方式.

在第二个节点的配置目录建立两个定义文件:

```
vagrant@n2:~$ echo '{"check": {"name": "ping",
  "script": "ping -c1 163.com >/dev/null", "interval": "30s"}}' \
  >/etc/consul.d/ping.json

vagrant@n2:~$ echo '{"service": {"name": "web", "tags": ["rails"], "port": 80,
  "check": {"script": "curl localhost >/dev/null 2>&1", "interval": "10s"}}}' \
  >/etc/consul.d/web.json
```

第一个定义增加了一个主机级别的检查,名字为 "ping" . 这个检查每30秒执行一次,执行 `ping -c1 163.com`. 在基于脚本的健康检查中,脚本运行在与Consul进程一样的用户下.如果这个命令以非0值退出的话这个节点就会被标记为不健康.这是所有基于脚本的健康检查的约定.

第二个命令定义了名为`web`的服务,添加了一个检查.每十分钟通过curl发送一个请求,确定web服务器可以访问.和主机级别的检查一样.如果脚本以非0值退出则标记为不健康.

现在重启第二个agent或者发送`SIGHUP`信号,你应该可以看到如下的日志内容输出:

```
==> Reloading configuration...
    2016/08/18 15:29:57 [INFO] agent: Synced service 'web'
    2016/08/18 15:29:57 [INFO] agent: Synced check 'ping'
    2016/08/18 15:29:58 [WARN] agent: Check 'service:web' is now critical
```

前几行检查到agent同步了新的定义.最后一行检查到web服务出于危险状态.这是因为我们实际上没有运行一个web服务器.所以```curl``的测试会一直失败!

## 检查健康状态

现在我们加入了一些简单的检查.我们能适应HTTP API来检查他们.首先我们检查有哪些失败的检查.使用这个命令\(注意:这个命令可以运行在任何节点\)

```
[root@hdp3 consul.d]# curl http://localhost:8500/v1/health/state/critical
[{"Node":"hdp3","CheckID":"service:web","Name":"Service 'web' check","Status":"critical","Notes":"","Output":"","ServiceID":"web","ServiceName":"web","CreateIndex":878,"ModifyIndex":878}]
```

我们可以看到,只有一个检查我们的`web`服务在`critical`状态

另外,我们可以尝试用DNS查询web服务,Consul将不会返回结果.因为服务不健康.

```
[root@hdp3 consul.d]# dig @127.0.0.1 -p 8600 web.service.consul

; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.47.rc1.el6 <<>> @127.0.0.1 -p 8600 web.service.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 33096
;; flags: qr aa rd; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;web.service.consul.           IN         A

;; AUTHORITY SECTION:
consul.            0          IN         SOA        ns.consul. postmaster.consul. 1471507354 3600 600 86400 0

;; Query time: 8 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Thu Aug 18 16:02:34 2016
;; MSG SIZE  rcvd: 104
```

## 下一步

在本章,你学到了如何鉴定的添加健康检查.检查定义可以通过配合文件并发送`SIGHUP`到agent进行更新.另外,HTTP API可以用来动态添加,移除和修改检查,以及进行TTL检查.TTL可以让应用程序更紧密的与Consul集成.将检查的状态加入到业务逻辑的计算.

