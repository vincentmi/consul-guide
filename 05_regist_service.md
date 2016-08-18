# 注册服务

在之前的步骤我们运行了第一个agent.看到了集群的成员,查询节点,在这个指南我们将注册我们的第一个服务并查询这些服务.

## 定义一个服务

可以通过提供服务定义或者调用HTTP API来注册一个服务.服务定义文件是注册服务的最通用的方式.所以我们将在这一步使用这种方式.我们将会建立在前一步我们覆盖的代理配置。

首先,为Consul配置创建一个目录.Consul会载入配置文件夹里的所有配置文件.在Unix系统中通常类似 ```/etc/consul.d``` (.d 后缀意思是这个路径包含了一组配置文件).

```
$ sudo mkdir /etc/consul.d
```
然后,我们将编写服务定义配置文件.假设我们有一个名叫```web```的服务运行在 80端口.另外,我们将给他设置一个标签.这样我们可以使用他作为额外的查询方式:

```
echo '{"service": {"name": "web", "tags": ["rails"], "port": 80}}' \
    >/etc/consul.d/web.json
```
现在重启agent , 设置配置目录:

```
$ consul agent -dev -config-dir /etc/consul.d
==> Starting Consul agent...
...
    [INFO] agent: Synced service 'web'
...
```

你可能注意到了输出了 "synced" 了 web这个服务.意思是这个agent从配置文件中载入了服务定义,并且成功注册到服务目录.

如果你想注册多个服务,你应该在Consul配置目录创建多个服务定义文件.


## 查询服务

一旦agent启动并且服务同步了.我们可以通过DNS或者HTTP的API来查询服务.


### DNS API

让我们首先使用DNS API来查询.在DNS API中,服务的DNS名字是 ```NAME.service.consul```. 虽然是可配置的,但默认的所有DNS名字会都在```consul```命名空间下.这个子域告诉Consul,我们在查询服务,```NAME```则是服务的名称.

对于我们上面注册的Web服务.它的域名是 ```web.service.consul``` : 

```
[root@hdp2 consul.d]# dig @127.0.0.1 -p 8600 web.service.consul

; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.47.rc1.el6 <<>> @127.0.0.1 -p 8600 web.service.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 46501
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;web.service.consul.   		IN     	A

;; ANSWER SECTION:
web.service.consul.    	0      	IN     	A      	10.0.0.52

;; Query time: 0 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Wed Aug 17 19:07:05 2016
;; MSG SIZE  rcvd: 70
```

如你所见,一个```A```记录返回了一个可用的服务所在的节点的IP地址.```A``记录只能设置为IP地址. 有也可用使用 DNS API 来接收包含 地址和端口的 SRV记录:

```
[root@hdp2 ~]# dig @127.0.0.1 -p 8600 web.service.consul SRV

; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.47.rc1.el6 <<>> @127.0.0.1 -p 8600 web.service.consul SRV
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 33415
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;web.service.consul.   		IN     	SRV

;; ANSWER SECTION:
web.service.consul.    	0      	IN     	SRV    	1 1 80 hdp2.node.dc1.consul.

;; ADDITIONAL SECTION:
hdp2.node.dc1.consul.  	0      	IN     	A      	10.0.0.52

;; Query time: 1 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Thu Aug 18 10:40:48 2016
;; MSG SIZE  rcvd: 130
```
```SRV```记录告诉我们 ```web``` 这个服务运行于节点```hdp2.node.dc1.consul``` 的```80```端口. DNS额外返回了节点的A记录.


最后,我们也可以用 DNS API 通过标签来过滤服务.基于标签的服务查询格式为```TAG.NAME.service.consul```. 在下面的例子中,我们请求Consul返回有 ```rails```标签的 ```web```服务.我们成功获取了我们注册为这个标签的服务:

```
[root@hdp2 ~]# dig @127.0.0.1 -p 8600 rails.web.service.consul SRV

; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.47.rc1.el6 <<>> @127.0.0.1 -p 8600 rails.web.service.consul SRV
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 3517
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;rails.web.service.consul.     	IN     	SRV

;; ANSWER SECTION:
rails.web.service.consul. 0    	IN     	SRV    	1 1 80 hdp2.node.dc1.consul.

;; ADDITIONAL SECTION:
hdp2.node.dc1.consul.  	0      	IN     	A      	10.0.0.52

;; Query time: 1 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Thu Aug 18 11:26:17 2016
;; MSG SIZE  rcvd: 142
```

### HTTP API

除了DNS API之外,HTTP API也可以用来进行服务查询:

```
[root@hdp2 ~]# curl http://localhost:8500/v1/catalog/service/web
[{"Node":"hdp2","Address":"10.0.0.52","ServiceID":"web","ServiceName":"web","ServiceTags":["rails"],"ServiceAddress":"","ServicePort":80,"ServiceEnableTagOverride":false,"CreateIndex":4,"ModifyIndex":254}]
```

目录API给出所有节点提供的服务.稍后我们会像通常的那样带上健康检查进行查询.就像DNS内部处理的那样.这是只查看健康的实例的查询方法:

```
[root@hdp2 ~]# curl http://localhost:8500/v1/catalog/service/web?passing
[{"Node":"hdp2","Address":"10.0.0.52","ServiceID":"web","ServiceName":"web","ServiceTags":["rails"],"ServiceAddress":"","ServicePort":80,"ServiceEnableTagOverride":false,"CreateIndex":4,"ModifyIndex":254}]
```

### 更新服务

服务定义可以通过配置文件并发送```SIGHUP```给agent来进行更新.这样你可以让你在不关闭服务或者保持服务请求可用的情况下进行更新.

另外 HTTP API可以用来动态的添加,移除和修改服务.



