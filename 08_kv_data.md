# 键值数据存储

除了提供服务发现和健康检查的集成.Consul提供了一个易用的键/值存储.这可以用来保持动态配置,协助服务协调,领袖选举,做开发者可以想到的任何事情.

这一章假设你已经有至少一个Consul的agent在运行.

## 简单使用

为了演示如果简单的使用键值存储.我们将操作一些键.查询本地agent我们首先确认现在还没有存储任何key.

```
[root@hdp3 consul.d]# curl -v http://localhost:8500/v1/kv/?recurse
* About to connect() to localhost port 8500 (#0)
*   Trying ::1... 拒绝连接
*   Trying 127.0.0.1... connected
* Connected to localhost (127.0.0.1) port 8500 (#0)
> GET /v1/kv/?recurse HTTP/1.1
> User-Agent: curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.21 Basic ECC zlib/1.2.3 libidn/1.18 libssh2/1.4.2
> Host: localhost:8500
> Accept: */*
>
< HTTP/1.1 404 Not Found
< X-Consul-Index: 1
< X-Consul-Knownleader: true
< X-Consul-Lastcontact: 0
< Date: Thu, 18 Aug 2016 08:21:39 GMT
< Content-Length: 0
< Content-Type: text/plain; charset=utf-8
<
* Connection #0 to host localhost left intact
* Closing connection #0
```

因为没有key所以我们得到了一个404响应.现在我们```PUT``一些示例的Key:

```
[root@hdp3 consul.d]# curl -X PUT -d 'test' http://localhost:8500/v1/kv/web/key1
[root@hdp3 consul.d]# curl -X PUT -d 'test' http://localhost:8500/v1/kv/web/key2?flags=42
[root@hdp3 consul.d]# curl -X PUT -d 'test'  http://localhost:8500/v1/kv/web/sub/key3
[root@hdp3 consul.d]# curl http://localhost:8500/v1/kv/?recurse
[{"LockIndex":0,"Key":"web/key1","Flags":0,"Value":"dGVzdA==","CreateIndex":1201,"ModifyIndex":1201},{"LockIndex":0,"Key":"web/key2","Flags":42,"Value":"dGVzdA==","CreateIndex":1205,"ModifyIndex":1206},{"LockIndex":0,"Key":"web/sub/key3","Flags":0,"Value":"dGVzdA==","CreateIndex":1217,"ModifyIndex":1217}]
```

我们创建了值为"test"的3个Key,注意返回的值是经过了base64编码的.用来支持非UTF8编码字符.对Key ```web/key2```我们设置了一个标志值为 ```42```.所有的key支持设置一个64位的整形数字标志.Consul内部不适用这个值.但是他可以被客户端适用来做一些元数据.

完成设置后,我们发起了一个```GET```请求来接收多个key的值,使用```?recurse```参数.

你可以获取单个的key

```
[root@hdp3 consul.d]# curl http://localhost:8500/v1/kv/web/key1
[{"LockIndex":0,"Key":"web/key1","Flags":0,"Value":"dGVzdA==","CreateIndex":1201,"ModifyIndex":1201}]
```

删除key也很简单.通过```DELETE```动作来完成.我们可以通过指定完整路径来删除一个单独的key.或者我们可以使用```?recurse```递归的删除主路径下所有key.

```
[root@hdp3 consul.d]# curl -X DELETE http://localhost:8500/v1/kv/web/sub?recurse
true
[root@hdp3 consul.d]# curl http://localhost:8500/v1/kv/web?recurse
[{"LockIndex":0,"Key":"web/key1","Flags":0,"Value":"dGVzdA==","CreateIndex":1201,"ModifyIndex":1201},{"LockIndex":0,"Key":"web/key2","Flags":42,"Value":"dGVzdA==","CreateIndex":1205,"ModifyIndex":1206}]
```

可以通过发送相同的URL并提供不同的消息体的```PUT```请求去修改一个Key.另外,Consul提供一个检查并设置的操作,实现原子的Key修改.通过```?cas=```参数加上```GET```中最近的```ModifyIndex```来达到. 例如我们想修改 "web/key1":

```
[root@hdp3 consul.d]# curl -X PUT -d 'newval' http://localhost:8500/v1/kv/web/key1?cas=1201
true
[root@hdp3 consul.d]# curl -X PUT -d 'newval' http://localhost:8500/v1/kv/web/key1?cas=1201
false
```

在这种情况下,第一次```CAS``` 更新成功因为```ModifyIndex```是```1201```.而第二次失败是因为```ModifyIndex```在第一次更新后已经不是```1201```了 .

我们也可以使用```ModifyIndex```来等待key值的改变.例如我们想等待```key2```被修改:

```
[root@hdp3 consul.d]# curl http://localhost:8500/v1/kv/web/key2
[{"LockIndex":0,"Key":"web/key2","Flags":42,"Value":"dGVzdA==","CreateIndex":1205,"ModifyIndex":1206}]

[root@hdp3 consul.d]# curl "http://localhost:8500/v1/kv/web/key2?index=1206&wait=5s"
[{"LockIndex":0,"Key":"web/key2","Flags":42,"Value":"dGVzdA==","CreateIndex":1205,"ModifyIndex":1206}]
```

通过提供 ```?index=```,我们请求等待key值有一个比```1206```更大的```ModifyIndex```.虽然```?wait=5s```参数限制了这个请求最多5秒,否则返回当前的未改变的值. 这样可以有效的等待key的改变.另外,这个功能可以用于等待一组key.直到其中的某个key有修改.

## 下一步

这里有一些说API可以支持的操作的例子,要查看完整文档.请查看[这里](https://www.consul.io/docs/agent/http/kv.html).

下面我们将看一看Consul支持的WebUI选项.