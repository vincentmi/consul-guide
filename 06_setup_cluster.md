# 建立集群

我们开始了第一个agent并且在agent上注册并查询了服务.这些展示了Consul是如何的易用.但是我们还不知道Consul如何进行扩容成一个可扩展,面向生成环境的服务发现架构.这一章我们将创建我们第一个拥有多个成员的真正的集群.

当一个agent启动时,他开始不知道其他节点的信息,他是一个成员的孤立集群.为了了解其他集群成员这个agent必须加入一个已经存在的集群.要加入一个已经存在的集群,只需要知道一个已经存在的集群成员.通过与这个成员的沟通来发现其他成员,Consul agent可以加入任何agent而不只是出于server模式的agent.

## 启动Agent 

>
> 官方版本教程里使用了Vagrant来启动虚拟机.我已经创建了多个虚拟机
> 因此跳过这部分
>

我们启动了另外的2台主机,10.0.0.53 ,10.0.0.54 和之前安装的方式一样,将consul拷贝到```PATH```目录完成安装.

在之前的示例中,我们使用了```-dev```参数来快速的创建一个开发模式的server.然而这并不能充分的在集群环境下使用.现在我们将忽略掉```-dev```标签,用我们的集群选项来替换他.

每个集群中的节点都必须要一个唯一的名字.Consul默认会使用机器的hostname.我们可以使用```-node```手动覆盖他.

我们也可以使用[-bind](https://www.consul.io/docs/agent/options.html#_bind)指定一个绑定的地址让Consul在这个地址上进行监听,这个地址必须可以被其他集群成员访问到.绑定地址不是必须提供,Consul选择第一个私有IP进行监听,不过最好还是指定一个.生产环境的服务器通常有多个网络接口.所以指定一个不会让Consul绑错网络接口.

第一个节点将扮演集群的唯一server,我们使用```-server```指定他.

```-bootstrap-expect``` 选项提示Consul我们期待加入的server节点的数量.这个选项的作用是启动时推迟日志复制直到我们期望的server都成功加入时.你可以阅读[启动指南](https://www.consul.io/docs/guides/bootstrapping.html)了解更多.

最后,我们加入 ```config-dir```选项,指定服务和健康检查定义文件存放的路径.

加到一起,命令如下:

```
consul agent -server -bootstrap-expect 1  -data-dir /tmp/consul -node=hdp2 -bind=10.0.0.52  -config-dir /etc/consul.d
```

现在在另外一个终端,我们将连接第二个节点:

```
ssh hdp3
# 登录第二台机器
```

这一次我们设置绑定的IP地址为第二个节点的IP的地址,并指定节点名称.因为这个节点将不是Consule的server.我们没有打开```server```开关.命令如下:

```
consul agent -data-dir /tmp/consul -node=hdp3 -bind=10.0.0.53 -config-dir /etc/consul.d
```

现在,你有运行了两个Consul的agent,一个作为server另一个作为client.这两个agent还互相不知道对方,只是作为独立的单节点集群.为了验证这个你可以在每个agent运行```consul member```,只能看到各自自己这一个集群成员. 

## 加入一个集群

现在我们告诉第一个agent来加入第二个agent,在新的终端运行如下命令

```
ssh hdp2
consul join 10.0.0.53
Successfully joined cluster by contacting 1 nodes.
```

>
> 如果出现 ```Error joining the cluster: dial tcp 10.0.0.53:8301: getsockopt: no route to host```
> 
> 可能是业务防火墙的原因,检查端口```8301```是否被允许
>

你应该可以看到在每个agent的日志输出窗口的一些输出.如果你仔细阅读会发现.他们收到了加入信息,如果你在每个agent运行```consul members```你会看到类似下面的内容:

```
[root@hdp2 ~]# consul members
Node  Address         Status  Type    Build  Protocol  DC
hdp2  10.0.0.52:8301  alive   server  0.6.4  2         dc1
hdp3  10.0.0.53:8301  alive   client  0.6.4  2         dc1
```

>
>记住:为了加入集群,一个Consul的agent只需要了解一个已经存在的集群成员.加入集群后agent会自动交流传递完整的成员信息.
>

## 启动时自动加入集群

理想的情况,当一个新的节点在数据中心启动时,他应该自动加入到Consul的集群中,而不需要人为干预.为了达到这个效果你可以使用[HashiCorp的Atlas](https://atlas.hashicorp.com/)和```-atlas-join```选项.示例如下:

```
consul agent -atlas-join \
  -atlas=ATLAS_USERNAME/infrastructure \
  -atlas-token="YOUR_ATLAS_TOKEN"
```
Atlas的用户名和token可以通过创建Atlas账号获取.这样当心的节点启动后他会自动加入到你的Consul集群,不需要硬编码配置.

另一种选择,你可以通过```-join```选项和```start_join```配置将其他已知的agent的地址进行硬编码来在启动时加入集群.

## 查询节点

就像查询服务一样.Consul有一个API用来查询节点自己.你可以通过DNS和HTTP的API来进行.

DNS API中节点名称结构为 ```NAME.node.consul```或者```NAME.node.DATACENTER.consul```.如果数据中心名字省略,Consul只会查询本地数据中心.

例如 从节点```hdp2```我们可以查询节点```hdp3```的地址:

```
[root@hdp2 ~]# dig @127.0.0.1 -p 8600 hdp3.node.consul

; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.47.rc1.el6 <<>> @127.0.0.1 -p 8600 hdp3.node.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 5351
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;hdp3.node.consul.     		IN     	A

;; ANSWER SECTION:
hdp3.node.consul.      	0      	IN     	A      	10.0.0.53

;; Query time: 1 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Thu Aug 18 14:32:02 2016
;; MSG SIZE  rcvd: 66
```

除服务之外查询节点的能力对于系统管理任务非常重要.例如知道节点的SSH登录地址,可以简单的将节点加入到Consul集群并查询他.

## 离开集群 

离开集群,你可以```Ctrl-C```优雅的退出,也可以直接Kill掉agent进程.优雅的退出可以让节点转变为离开状态.否则节点将被标记为失败.详细的细节可以查看[这里](https://www.consul.io/intro/getting-started/agent.html#stopping).

