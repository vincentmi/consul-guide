# Consul Template 简介

Consul Template 提供一个方便的方式从Consul获取数据通过consul-template的后台程序保存到文件系统.

这个后台进程监控Consul示例的变化并更新任意数量的模板到文件系统.作为一个附件功能,模板更新完成后consul-template可以运行任何命令.可以查看示例部分看这个功能将会对哪些应用场景产生帮助.

## 安装

你可以在[发布页](https://releases.hashicorp.com/consul-template/)下载发布包.如果你希望自己编译请查看[说明文档](https://github.com/hashicorp/consul-template#contributing).

## 使用

#### 选项

查看全部选项,使用以下命令

```
consul-template -h 
```

#### 命令行

```CLI```接口支持上面出现的所有选项.

查询 ```demo.consul.io```这个```Consul```实例(agent).渲染模板文件 ```/tmp/template.ctmpl``` 保存到 ``` /tmp/result```, 运行```Consul-template```服务直到直到手动结束:

```
consul-template \
  -consul demo.consul.io \
  -template "/tmp/template.ctmpl:/tmp/result"
```

查询本地的```Consul```实例(agent),一旦模板发生变化渲染模板并重启```Nginx```,如果```Consul```不可用30秒重试一次:

```
consul-template \
  -consul 127.0.0.1:8500 \
  -template "/tmp/template.ctmpl:/var/www/nginx.conf:service nginx restart" \
  -retry 30s \
  -once
```

查询一个```Consul```实例,渲染多个模板并作为服务直到停止:

```
consul-template \
  -consul my.consul.internal:6124 \
  -template "/tmp/nginx.ctmpl:/var/nginx/nginx.conf:service nginx restart" \
  -template "/tmp/redis.ctmpl:/var/redis/redis.conf:service redis restart" \
  -template "/tmp/haproxy.ctmpl:/var/haproxy/haproxy.conf"
```

查询一个需要权限验证的```Consul```实例,将渲染后的模板输出到控制台而不写入磁盘.在这个例子中```-template```的第二个和第三个参数是必须的但是被忽略了.这个文件将不会被写入磁盘,命令也不会被执行.

```
$ consul-template \
  -consul my.consul.internal:6124 \
  -template "/tmp/template.ctmpl:/tmp/result:service nginx restart"
  -dry
```

使用SSL证书进行```Consul```的查询:

```
$ consul-template \
  -consul 127.0.0.1:8543 \
  -ssl \
  -ssl-cert /path/to/client/cert.pem \
  -ssl-ca-cert /path/to/ca/cert.pem \
  -template "/tmp/template.ctmpl:/tmp/result" \
  -dry \
  -once
```

查询```Consul```并启动一个子进程.模板的变化会发送指令给子进程.详细的说明请查看[这里](https://github.com/hashicorp/consul-template#exec-mode).

```
$ consul-template \
  -template "/tmp/in.ctmpl:/tmp/result" \
  -exec "/sbin/my-server"
```


#### 配置文件

```Consul-Template```配置文件是使用[HashiCorp Configuration Language (HCL)](https://github.com/hashicorp/hcl)编写的.这意味着```Consul Template```是和JSON兼容的,查看更多信息请查看 [HCL 规范](https://github.com/hashicorp/hcl)

配置文件语法支持上面的所有的选项,除非在表格中进行标明.

```json

// 这是要连接的Consul Agent的地址.默认为127.0.0.1:8500.这是Consul的默认绑定地址和端口.
// 不建议你直接与 Consul的 Server直接进行交互,请与本地的Consul Agent进行交互.这样做是有一些原因
// 最重要的是本地agent可以复用与server的连接.减少HTTP的连接数.另外这个地址更好记.
consul = "127.0.0.1:8500"

// 这是用于连接Consul的ACL token. 如果你的集群未启用就不需要设置.
//
// 这个选项也可以通过环境变量 CONSUL_TOKEN 来进行设置
token = "abcd1234"

// 这是监听出发reload事件的信号,默认值如下所示.将这个值设置为空将引起 CT ,从而不监听reload事件
reload_signal = "SIGHUP"

// 这是监听出发core dump事件的信号,默认值如下所示.将这个值设置为空将引起 CT ,从而不监听core dump信号
dump_signal = "SIGQUIT"

// 这是监听出发graceful stop事件的信号,默认值如下所示.将这个值设置为空将引起 CT ,从而不监听graceful stop信号
kill_signal = "SIGINT"

// 这是连接Consul的重试时间.Consul Template是高容错的设计.这意味着,出现失败他不会退出.而按照
// 分布式系统的惯例进行指数补偿和重试来等待集群恢复.
retry = "10s"

// This is the maximum interval to allow "stale" data. By default, only the
// Consul leader will respond to queries; any requests to a follower will
// forward to the leader. In large clusters with many requests, this is not as
// scalable, so this option allows any follower to respond to a query, so long
// as the last-replicated data is within these bounds. Higher values result in
// less cluster load, but are more likely to have outdated data.
// 这是允许陈旧数据的最大时间.Consul默认只有领袖对请求进行相应.所有对追随者的请求将被转发给领袖.
// 在有大量请求的大型集群中,这显得不够有扩展性.所以这个选项允许任何追随者响应查询,只要最后复制的数据
// 在这个范围内.数值越高,越减少集群负载,但是更容易接受到过期数据.
max_stale = "10m"

// 这是log的等级,如果你找到了bug,请打开debug 日志,这样我们可以更好的定位问题.这个选项也可用在命令行.
log_level = "warn"

// 这是存放Consul Template 进程的PID文件的路径,如果你计划发送定制的信号到这个进程这会比较有用.
pid_file = "/path/to/pid"

// 这是一个静止定时器,他定义了在模板渲染之前等待集群达到一致状态的最小和最大时间.
// 这对于一些变化较大的系统中比较有用,可以减少模板渲染的次数
wait = "5s:10s"


// 这是 Vault配置的开始
// Vault是HashiCorp的另外一个产品
vault {
  // This is the address of the Vault leader. The protocol (http(s)) portion
  // of the address is required.
  address = "https://vault.service.consul:8200"

  // This is the token to use when communicating with the Vault server.
  // Like other tools that integrate with Vault, Consul Template makes the
  // assumption that you provide it with a Vault token; it does not have the
  // incorporated logic to generate tokens via Vault's auth methods.
  //
  // This value can also be specified via the environment variable VAULT_TOKEN.
  token = "abcd1234"

  // This option tells Consul Template to automatically renew the Vault token
  // given. If you are unfamiliar with Vault's architecture, Vault requires
  // tokens be renewed at some regular interval or they will be revoked. Consul
  // Template will automatically renew the token at half the lease duration of
  // the token. The default value is true, but this option can be disabled if
  // you want to renew the Vault token using an out-of-band process.
  //
  // Note that secrets specified in a template (using {{secret}} for example)
  // are always renewed, even if this option is set to false. This option only
  // applies to the top-level Vault token itself.
  renew = true

  // This section details the SSL options for connecting to the Vault server.
  // Please see the SSL options below for more information (they are the same).
  ssl {
    // ...
  }
}

// 这部分配置请求的基本的权限验证信息
auth {
  enabled  = true
  username = "test"
  password = "test"
}

// 这部分配置连接到Consul服务器的SSL信息.
ssl {
  // 使用SSL需要先打开这个开关
  enabled = true

  // This enables SSL peer verification. The default value is "true", which
  // will check the global CA chain to make sure the given certificates are
  // valid. If you are using a self-signed certificate that you have not added
  // to the CA chain, you may want to disable SSL verification. However, please
  // understand this is a potential security vulnerability.
  verify = false

  // This is the path to the certificate to use to authenticate. If just a
  // certificate is provided, it is assumed to contain both the certificate and
  // the key to convert to an X509 certificate. If both the certificate and
  // key are specified, Consul Template will automatically combine them into an
  // X509 certificate for you.
  cert = "/path/to/client/cert"
  key = "/path/to/client/key"

  // This is the path to the certificate authority to use as a CA. This is
  // useful for self-signed certificates or for organizations using their own
  // internal certificate authority.
  ca_cert = "/path/to/ca"
}

// 设置连接到syslog服务器的配置
// 用于进行日志记录syslog {
  // 打开开关
  enabled = true

  // 设备名称
  facility = "LOCAL5"
}

// This block defines the configuration for de-duplication mode. Please see the
// de-duplication mode documentation later in the README for more information
// on how de-duplication mode operates.
deduplicate {
  // This enables de-duplication mode. Specifying any other options also enables
  // de-duplication mode.
  enabled = true

  // This is the prefix to the path in Consul's KV store where de-duplication
  // templates will be pre-rendered and stored.
  prefix = "consul-template/dedup/"
}

// This block defines the configuration for exec mode. Please see the exec mode
// documentation at the bottom of this README for more information on how exec
// mode operates and the caveats of this mode.
exec {
  // This is the command to exec as a child process. There can be only one
  // command per Consul Template process.
  command = "/usr/bin/app"

  // This is a random splay to wait before killing the command. The default
  // value is 0 (no wait), but large clusters should consider setting a splay
  // value to prevent all child processes from reloading at the same time when
  // data changes occur. When this value is set to non-zero, Consul Template
  // will wait a random period of time up to the splay value before reloading
  // or killing the child process. This can be used to prevent the thundering
  // herd problem on applications that do not gracefully reload.
  splay = "5s"

  // This defines the signal that will be sent to the child process when a
  // change occurs in a watched template. The signal will only be sent after
  // the process is started, and the process will only be started after all
  // dependent templates have been rendered at least once. The default value
  // is "" (empty or nil), which tells Consul Template to restart the child
  // process instead of sending it a signal. This is useful for legacy
  // applications or applications that cannot properly reload their
  // configuration without a full reload.
  reload_signal = "SIGUSR1"

  // This defines the signal sent to the child process when Consul Template is
  // gracefully shutting down. The application should begin a graceful cleanup.
  // If the application does not terminate before the `kill_timeout`, it will
  // be terminated (effectively "kill -9"). The default value is "SIGTERM".
  kill_signal = "SIGINT"

  // This defines the amount of time to wait for the child process to gracefully
  // terminate when Consul Template exits. After this specified time, the child
  // process will be force-killed (effectively "kill -9"). The default value is
  // "30s".
  kill_timeout = "2s"
}

// 这部分定义了对模板的配置,和其他配置块不同.这部分可以针对不同模板配置多次.也可以在CLI命令
// 直接进行配置
template {
  // 这是输入模板的配置文件路径,必选项
  source = "/path/on/disk/to/template.ctmpl"

  // 这是源模板渲染之后存放的路径,如果父目录不存在Consul Template会尝试进行创建
  destination = "/path/on/disk/where/template/will/render.txt"

  // This is the optional command to run when the template is rendered. The
  // command will only run if the resulting template changes. The command must
  // return within 30s (configurable), and it must have a successful exit code.
  // Consul Template is not a replacement for a process monitor or init system.
  // 这是当模板渲染完成后可选的要执行的命令.这个命令只会在模板发生改变后才会运行.这个命令必须要在30秒
  // 内进行返回(可配置),必须返回一个成功的退出码.Consul Template不能替代进程监视或者init 系统
  // 的功能
  command = "restart service foo"

  // 这是最大的等待命令返回的时间,默认是30秒
  command_timeout = "60s"

  // 这是渲染后的文件的权限,如果不设置,Consul Template将去匹配之前已经存在的文件的权限.
  // 如果文件不存在,权限会被设置为 0644
  perms = 0600

  // 这个选项对渲染之前的文件进行备份.他保持一个备份.
  // 这个选项在发生意外更高时,有一个回滚策略.
  backup = true

  // 模板的分隔符,默认是 "{{"和"}}".但是对于一些模板用其他的分隔符可能更好
  // 可以避免与本身的冲突
  left_delimiter  = "{{"
  right_delimiter = "}}"

  // 这是最小和最大等待渲染一个新模板和执行命令的时间.使用 分号 个号.如果忽略最大值,最大
  // 值会被设置为最小值的4倍.这个选项没有默认值.这个值相对全局所以的等待时间有最高优先级
  wait = "2s:6s"
}
```

> 
> 注意: 不是所有的选项都是必选的.例如: 如果你没有使用Vault你不用设置这一块.
> 类似的你没有使用syslog系统你也不需要指定syslog配置.
> 

为了更加安全,```token```也可以从环境变量里读取,使用 ```CONSUL_TOKEN``` 和 ```VAULT_TOKEN```.强烈建议你不要把token放到未加密的文本配置文件中.

查询 nyc3 demo 的Consul示例, 渲染模板 ```/tmp/template.ctmpl``` 到```/tmp/result```.运行Consul Template直到服务停止:

```json
consul = "nyc3.demo.consul.io"

template {
  source      = "/tmp/template.ctmpl"
  destination = "/tmp/result"
}
```

如果一个用一个目录替换文件,所以这个目录中的文件会递归的安装Go walk函数的顺序进行合并.所以如果多个文件定义了```consul``` key 则最后一个将会被使用,注意,符号链接不会被加入.

**在命令行指定的选项,优先于配置文件**

## 模板语法

Consul Template 使用了Go的模板语法.如果你对他的语法不熟悉建议你读下文档.他的语法看起来与 Mustache, Handlebars, 或者 Liquid 类似.

在Go 提供的模板函数之外,Consul Template暴露了以下的函数:

###API 函数###

##### datacenters####

查询目录中的所有数据中心.使用以下语法:

```
{{datacenters}}
```

##### file #####

读取并输出磁盘上的本地文件,如果无法读取产生一个错误.使用如下语法

```
{{file "/path/to/local/file"}}
```

这个例子将输出 ```/path/to/local/file``` 文件内容到模板. **注意:这不会在嵌套模板中被处理**

##### key #####

查询Consul指定key的值,如果key的值不能转换为字符串,将产生错误.使用如下语法:

```
{{key "service/redis/maxconns@east-aws"}}
```

上面的例子查询了在```east-aws```数据中心的 ```service/redis/maxconns```的值.如果忽略数据中心参数,将会查询本地数据中心的值:

```
{{key "service/redis/maxconns"}}
```
Consul键值结构的美妙在于,这完全取决于你!

##### key_or_default #####

查询Consul中指定的key的值,如果key不存在,则返回默认值.使用方式如下

```
{{key_or_default "service/redis/maxconns@east-aws" "5"}}
```
注意Consul Template使用了多个阶段的运算.在第一阶段的运算如果Consul没有返回值,则会一直使用默认值.后续模板解析中如果值存在了则会读取真实的值.这很重要,运维Consul Templae不会因为```key_or_default```没找到key而阻塞模板的的渲染.即使key存在如果Consul没有按时返回这个数据,也会使用默认值来进行替代.

##### ls #####

查看Consul的所有以指定前缀开头的key-value对.如果有值无法转换成字符串则会产生一个错误:

```
{{range ls "service/redis@east-aws"}}
{{.Key}} {{.Value}}{{end}}
```

如果Consul实例在```east-aws```数据中心存在这个结构```service/redis```,渲染后的模板应该类似这样:

```
minconns 2
maxconns 12
```
如果你忽略数据中心属性,则会返回本地数据中心的查询结果.

##### node #####

查询目录中的一个节点信息

```
{{node "node1"}}
```

如果未指定任何参数,则当前agent所在节点将会被返回:

```
{{node}}
```

你可以指定一个可选的参数来指定数据中心:

```
{{node "node1" "@east-aws"}}
```

如果指定的节点没有找到则会返回```nil```.如果节点存在就会列出节点的信息和节点提供的服务.

```
{{with node}}{{.Node.Node}} ({{.Node.Address}}){{range .Services}}
  {{.Service}} {{.Port}} ({{.Tags | join ","}}){{end}}
{{end}}
```

##### nodes #####

查询目录中的全部节点,使用如下语法

```
{{nodes}}
```

这个例子会查询Consul的默认数据中心.你可以使用可选参数指定一个可选参数来指定数据中心:

```
{{nodes "@east-aws"}}
```

这个例子会查询```east-aws```数据中心的所有几点.


##### secret #####

查询```Vault```中指定路径的密匙.如果指定的路径不存在或者```Vault```的Token没有足够权限去读取指定的路径,将会产生一个错误.如果路径存在但是key不存在则返回```<no value>```.

```
{{with secret "secret/passwords"}}{{.Data.password}}{{end}}
```

可以使用如下字段:


```
LeaseID - the unique lease identifier
LeaseDuration - the number of seconds the lease is valid
Renewable - if the secret is renewable
Data - the raw data - this is a map[string]interface{}, so it can be queried using Go's templating "dot notation"
If the map key has dots "." in it, you need to access the value using the index function:

{{index .Data "my.key.with.dots"}}
If additional arguments are passed to the function, then the operation is assumed to be a write operation instead of a read operation. The write operation must return data in order to be valid. This is especially useful for the PKI secret backend, for example.

{{ with secret "pki/issue/my-domain-dot-com" "common_name=foo.example.com" }}
{{ .Data.certificate }}
{{ end }}
The parameters must be key=value pairs, and each pair must be its own argument to the function:

{{ secret "path/" "a=b" "c=d" "e=f" }}
Please always consider the security implications of having the contents of a secret in plain-text on disk. If an attacker is able to get access to the file, they will have access to plain-text secrets.
```
Please note that Vault does not support blocking queries. As a result, Consul Template will not immediately reload in the event a secret is changed as it does with Consul's key-value store. Consul Template will fetch a new secret at half the lease duration of the original secret. For example, most items in Vault's generic secret backend have a default 30 day lease. This means Consul Template will renew the secret every 15 days. As such, it is recommended that a smaller lease duration be used when generating the initial secret to force Consul Template to renew more often.

##### secrets #####

Query Vault to list the secrets at the given path. Please note this requires Vault 0.5+ and the endpoint you want to list secrets must support listing. Not all endpoints support listing. The result is the list of secret names as strings.
```
{{range secrets "secret/"}}{{.}}{{end}}
```
The trailing slash is optional in the template, but the generated secret dependency will always have a trailing slash in log output.

To iterate and list over every secret in the generic secret backend in Vault, for example, you would need to do something like this:
```
{{range secrets "secret/"}}
{{with secret (printf "secret/%s" .)}}
{{range $k, $v := .Data}}
{{$k}}: {{$v}}
{{end}}
{{end}}
{{end}}
```
You should probably never do this. Please also note that Vault does not support blocking queries. To understand the implications, please read the note at the end of the secret function.

##### service #####

查询Consul中匹配表达式的服务.语法如下:

```
{{service "release.web@east-aws"}}
```
上面的例子查询Consul中,在```east-aws```数据中心存在的健康的 ```web```服务.tag和数据中心参数是可选的.从当前数据中心查询所有节点的```web```服务而不管tag,查询语法如下:
```
{{service "web"}}
```

这个函数返回```[]*HealthService```结构.可按照如下方式应用到模板:

```
{{range service "web@data center"}}
server {{.Name}} {{.Address}}:{{.Port}}{{end}}
```

产生如下输出:

```
server nyc_web_01 123.456.789.10:8080
server nyc_web_02 456.789.101.213:8080
```

默认值会返回健康的服务,如果你想获取所有服务,可以增加```any```选项,如下:

```
{{service "web" "any"}}
```

这样就会返回注册过的所有服务,而不论他的状态如何.

如果你想过滤指定的一个或者多个健康状态,你可以通过逗号隔开多个健康状态:

```
{{service "web" "passing, warning"}}
```

这样将会返回被他们的节点和服务级别的检查定义标记为 "passing" 或者 "warning"的服务.
请注意逗号是 ```OR```而不是```AND```的意思.

指定了超过一个状态过滤,并包含```any```将会返回一个错误.因为```any```是比所有状态更高级的过滤.

后面这2种方式有些架构上的不同:

```
{{service "web"}}
{{service "web" "passing"}}
```

前者会返回Consul认为```healthy```和```passing```的所有服务.后者将返回所有已经在Consul注册的服务.然后会执行一个客户端的过滤.通常如果你想获取健康的服务,你应该不要使用```passing```参数,直接忽略第三个参数即可.然而第三个参数在你想查询 ```passing```或者```warning```的服务会比较有用,如下:

```
{{service "web" "passing, warning"}}
```

服务的状态也是可见的,如果你想自己做一些额外的过滤,语法如下:

```
{{range service "web" "any"}}
{{if eq .Status "critical"}}
// Critical state!{{end}}
{{if eq .Status "passing"}}
// Ok{{end}}
```

执行命令时,在Consul将服务设置为维护模式,只需要在你的命令上包上Consul的 ```maint``` 调用:

```
#!/bin/sh
set -e
consul maint -enable -service web -reason "Consul Template updated"
service nginx reload
consul maint -disable -service web
```

另外如果你没有安装Consul agent,你可以直接调用API请求:

```
#!/bin/sh
set -e
curl -X PUT "http://$CONSUL_HTTP_ADDR/v1/agent/service/maintenance/web?enable=true&reason=Consul+Template+Updated"
service nginx reload
curl -X PUT "http://$CONSUL_HTTP_ADDR/v1/agent/service/maintenance/web?enable=false"
```


##### services #####

查询Consul目录中的所有服务,使用如下语法:

```
{{services}}
```

这个例子将查询Consul的默认数据中心,你可以指定一个可选参数来指定数据中心:

```
{{services "@east-aws"}}
```
请注意: ```services```函数与```service```是不同的,```service```接受更多参数并且查询监控的服务列表.这个查询Consul目录并返回一个服务的tag的Map,如下:

```
{{range services}}
{{.Name}}
{{range .Tags}}
  {{.}}{{end}}
{{end}}
```

##### tree #####

查询所有指定前缀的key-value值对,如果其中的值有无法转换为字符串的则引发错误:

```
{{range tree "service/redis@east-aws"}}
{{.Key}} {{.Value}}{{end}}
```
如果Consul实例在```east-aws```数据中心有一个```service/redis```结构,模板的渲染结果类似下面:

```
minconns 2
maxconns 12
nested/config/value "value"
```

和```ls```不同,```tree```返回前缀下的所有key.和Unix的tree命令比较像.如果忽略数据中心参数,则会使用本地数据中心

### 帮助函数 ###

##### byKey #####

将```tree```返回的key-value值对结果创建一个map,这个map根据他们的顶级目录进行组合.例如如果Consul的kv存储如下结构:

```
groups/elasticsearch/es1
groups/elasticsearch/es2
groups/elasticsearch/es3
services/elasticsearch/check_elasticsearch
services/elasticsearch/check_indexes
```

使用下面的模板:

```
{{range $key, $pairs := tree "groups" | byKey}}{{$key}}:
{{range $pair := $pairs}}  {{.Key}}={{.Value}}
{{end}}{{end}}
```

结果如下:

```
elasticsearch:
  es1=1
  es2=1
  es3=1
```

注意顶部的key会被从key的值中剥离出来.如果在剥离前缀后没有前缀,值会被从列表移除.

结果的对会被映射为map,使用可以使用key来访问一个单独的值:

```
{{$weights := tree "weights"}}
{{range service "release.web"}}
  {{$weight := or (index $weights .Node) 100}}
  server {{.Node}} {{.Address}}:{{.Port}} weight {{$weight}}{{end}}
```

##### byTag #####

将被```service```或者```services```函数返回的列表,按照tag对服务创建Map.

```
{{range $tag, $services := service "web" | byTag}}{{$tag}}
{{range $services}} server {{.Name}} {{.Address}}:{{.Port}}
{{end}}{{end}}
```

##### contains #####

检查目标是否包含在枚举的元素中

```
{{ if .Tags | contains "production" }}
# ...
{{ end }}
env
```

读取当前进程可以访问的环境变量.

```
{{env "CLUSTER_ID"}}
```

这个函数可以加入管道:

```
{{env "CLUSTER_ID" | toLower}}
```

##### explode #####

将```tree```或者```ls```的结果转化为深度嵌套的map,用来进行解析和递归.

```
{{ tree "config" | explode }}
```

注意: 解开后,你将丢失所有的关于键值对的元数据.

你可以访问深度嵌套的值:

```
{{ with tree "config" | explode }}
{{.a.b.c}}{{ end }}
```
注意: 你需要在Consul中保存有一个合理的格式的数据.可以查看Go的 text/template包获取更多信息.

##### in #####

检查目标十分在一个可枚举的元素中.

```
{ if in .Tags "production" }}
# ...
{{ end }}
```

##### loop #####

接受多个参数,行为受这些参数的影响.

如果给```loop```一个数字,他讲返回一个```goroutine```,开始于0循环直到等于参数的值:

```
{{range loop 5}}
# Comment{{end}}
```
如果给2个数字,则这个函数返回一个 ```goroutine``` 从第一个数字开始循环直到等于第二个参数的值.

```
{{range $i := loop 5 8}}
stanza-{{$i}}{{end}}
```
渲染结果为:

```
stanza-5
stanza-6
stanza-7
```

Note: It is not possible to get the index and the element since the function returns a goroutine, not a slice. In other words, the following is not valid:

```
# Will NOT work!
{{range $i, $e := loop 5 8}}
# ...{{end}}
```


##### join #####

将提供的列表作为管道与提供的字符串连接:

```
{{$items | join ","}}
```

##### trimSpace #####

对输入的内容移除掉空白,tab和换行符

```
{ file "/etc/ec2_version"| trimSpace }}
```


##### parseBool #####

将给定的字符串解析为布尔值:

```
{{"true" | parseBool}}
```

这个可以与一个key检查和条件检查相结合.如下:

```
{{if key "feature/enabled" | parseBool}}{{end}}
```

##### parseFloat #####

将给定的字符串解析为 10进制 float64类型数字:

```
{{"1.2" | parseFloat}}
```

##### parseInt #####

将给定字符串解析为10禁止 int64类型数字:

```
{{"1" | parseInt}}
```

这个可以与其他的帮助函数结合使用,例如:

```
{{range $i := loop key "config/pool_size" | parseInt}}
# ...{{end}}
```

##### parseJSON #####
将输入,通常是通过key获取的值,解析成JSON

Takes the given input (usually the value from a key) and parses the result as JSON:

```
{{with $d := key "user/info" | parseJSON}}{{$d.name}}{{end}}
```

注意 : Consul Template计算模板很多次.第一次计算时会是空,因为数据还未载入,这意味着我们需要检查空的响应.例如:

```
{{with $d := key "user/info" | parseJSON}}
{{if $d}}
...
{{end}}
{{end}}
```

它只适用简单的key,但是如果你想遍历key或者使用index函数会失败.将要访问的代码包含在 ```{{ if $d }}...{{end}}``` 之中就够了.

Alternatively you can read data from a local JSON file:

```
{{with $d := file "/path/to/local/data.json" | parseJSON}}{{$d.some_key}}{{end}}
```

##### parseUint

Takes the given string and parses it as a base-10 int64:

```
{{"1" | parseUint}}
```

See parseInt for examples.

########## plugin

Takes the name of a plugin and optional payload and executes a Consul Template plugin.

```
{{plugin "my-plugin"}}
```

This is most commonly combined with a JSON filter for customization:

```
{{tree "foo" | explode | toJSON | plugin "my-plugin"}}
```

Please see the plugins section for more information about plugins.

##### regexMatch

Takes the argument as a regular expression and will return true if it matches on the given string, or false otherwise.

`````
{{"foo.bar" | regexMatch "foo([.a-z]+)"}}
````
##### regexReplaceAll

Takes the argument as a regular expression and replaces all occurrences of the regex with the given string. As in go, you can use variables like $1 to refer to subexpressions in the replacement string.

```
{{"foo.bar" | regexReplaceAll "foo([.a-z]+)" "$1"}}
```

##### replaceAll

Takes the argument as a string and replaces all occurrences of the given string with the given string.

```
{{"foo.bar" | replaceAll "." "_"}}
```

This function can be chained with other functions as well:

```
{{service "web"}}{{.Name | replaceAll ":" "_"}}{{end}}
```

##### split

Splits the given string on the provided separator:

```
{{"foo\nbar\n" | split "\n"}}
```

This can be combined with chained and piped with other functions:

```
{{key "foo" | toUpper | split "\n" | join ","}}
```

##### timestamp

Returns the current timestamp as a string (UTC). If no arguments are given, the result is the current RFC3339 timestamp:

```
{{timestamp}} // e.g. 1970-01-01T00:00:00Z
```
If the optional parameter is given, it is used to format the timestamp. The magic reference date Mon Jan 2 15:04:05 -0700 MST 2006 can be used to format the date as required:

```
{{timestamp "2006-01-02"}} // e.g. 1970-01-01
```

See Go's time.Format() for more information.

As a special case, if the optional parameter is "unix", the unix timestamp in seconds is returned as a string.

```
{{timestamp "unix"}} // e.g. 0
```

toJSON

Takes the result from a tree or ls call and converts it into a JSON object.

```
{{ tree "config" | explode | toJSON }} // e.g. {"admin":{"port":1234},"maxconns":5,"minconns":2}
```

Note: This functionality should be considered final. If you need to manipulate keys, combine values, or perform mutations, that should be done outside of Consul. In order to keep the API scope limited, we likely will not accept Pull Requests that focus on customizing the toJSON functionality.

toJSONPretty

Takes the result from a tree or ls call and converts it into a pretty-printed JSON object, indented by two spaces.

```
{{ tree "config" | explode | toJSONPretty }}
```
```
/*
{
  "admin": {
    "port": 1234
  },
  "maxconns": 5,
  "minconns": 2,
}
*/
```
Note: This functionality should be considered final. If you need to manipulate keys, combine values, or perform mutations, that should be done outside of Consul. In order to keep the API scope limited, we likely will not accept Pull Requests that focus on customizing the toJSONPretty functionality.

toLower

Takes the argument as a string and converts it to lowercase.

```
{{key "user/name" | toLower}}
```

See Go's strings.ToLower() for more information.

toTitle

Takes the argument as a string and converts it to titlecase.

```
{{key "user/name" | toTitle}}
```

See Go's strings.Title() for more information.

toUpper

Takes the argument as a string and converts it to uppercase.

```
{{key "user/name" | toUpper}}
```

See Go's strings.ToUpper() for more information.

toYAML

Takes the result from a tree or ls call and converts it into a pretty-printed YAML object, indented by two spaces.

```
{{ tree "config" | explode | toYAML }}
/*
admin:
  port: 1234
maxconns: 5
minconns: 2
*/
```
Note: This functionality should be considered final. If you need to manipulate keys, combine values, or perform mutations, that should be done outside of Consul. In order to keep the API scope limited, we likely will not accept Pull Requests that focus on customizing the toYAML functionality.

Math Functions

The following functions are available on floats and integer values.

add

Returns the sum of the two values.

```
{{ add 1 2 }} // 3
```

This can also be used with a pipe function.

```
{{ 1 | add 2 }} // 3
```

subtract

Returns the difference of the second value from the first.

```
{{ subtract 2 5 }} // 3
```

This can also be used with a pipe function.

```
{{ 5 | subtract 2 }}
```

Please take careful note of the order of arguments.

multiply

Returns the product of the two values.

```
{{ multiply 2 2 }} // 4
```

This can also be used with a pipe function.

```
{{ 2 | multiply 2 }} // 4
```

divide

Returns the division of the second value from the first.

```
{{ divide 2 10 }} // 5
```

This can also be used with a pipe function.

```
{{ 10 | divide 2 }} // 5
```

Please take careful note of the order or arguments.

Plugins

### Authoring Plugins

For some use cases, it may be necessary to write a plugin that offloads work to another system. This is especially useful for things that may not fit in the "standard library" of Consul Template, but still need to be shared across multiple instances.

Consul Template plugins must have the following API:

```
$ NAME [INPUT...]
```
NAME - the name of the plugin - this is also the name of the binary, either a full path or just the program name. It will be executed in a shell with the inherited PATH so e.g. the plugin cat will run the first executable cat that is found on the PATH.
INPUT - input from the template - this will always be JSON if provided
Important Notes

Plugins execute user-provided scripts and pass in potentially sensitive data from Consul or Vault. Nothing is validated or protected by Consul Template, so all necessary precautions and considerations should be made by template authors
Plugin output must be returned as a string on stdout. Only stdout will be parsed for output. Be sure to log all errors, debugging messages onto stderr to avoid errors when Consul Template returns the value.
Always exit 0 or Consul Template will assume the plugin failed to execute
Ensure the empty input case is handled correctly (see Multi-phase execution)
Data piped into the plugin is appended after any parameters given explicitly (eg ```{{ "sample-data" | plugin "my-plugin" "some-parameter"}}``` will call my-plugin some-parameter sample-data)
Here is a sample plugin in a few different languages that removes any JSON keys that start with an underscore and returns the JSON string:

```
#! /usr/bin/env ruby
require "json"

if ARGV.empty?
  puts JSON.fast_generate({})
  Kernel.exit(0)
end

hash = JSON.parse(ARGV.first)
hash.reject! { |k, _| k.start_with?("_")  }
puts JSON.fast_generate(hash)
Kernel.exit(0)
func main() {
  arg := []byte(os.Args[1])

  var parsed map[string]interface{}
  if err := json.Unmarshal(arg, &parsed); err != nil {
    fmt.Fprintln(os.Stderr, fmt.Sprintf("err: %s", err))
    os.Exit(1)
  }

  for k, _ := range parsed {
    if string(k[0]) == "_" {
      delete(parsed, k)
    }
  }

  result, err := json.Marshal(parsed)
  if err != nil {
    fmt.Fprintln(os.Stderr, fmt.Sprintf("err: %s", err))
    os.Exit(1)
  }

  fmt.Fprintln(os.Stdout, fmt.Sprintf("%s", result))
  os.Exit(0)
}
Caveats
```

## Exec Mode

As of version 0.16.0, Consul Template has the ability to maintain an arbitrary child process (similar to envconsul). This mode is most beneficial when running Consul Template in a container or on a scheduler like Nomad or Kubernetes. When activated, Consul Template will spawn and manage the lifecycle of the child process.

This mode is best-explained through example. Consider a simple application that reads a configuration file from disk and spawns a server from that configuration.

```
$ consul-template \
    -template="/tmp/config.ctmpl:/tmp/server.conf" \
    -exec="/bin/my-server -config /tmp/server.conf"
```

When Consul Template starts, it will pull the required dependencies and populate the /tmp/server.conf, which the my-server binary consumes. After that template is rendered completely the first time, Consul Template spawns and manages a child process. When any of the list templates change, Consul Template will send the configurable reload signal to that child process. If no reload signal is provided, Consul Template will kill and restart the process. Additionally, in this mode, Consul Template will proxy any signals it receives to the child process. This enables a scheduler to control the lifecycle of the process and also eases the friction of running inside a container.

A common point of confusion is that the command string behaves the same as the shell; it does not. In the shell, when you run foo | bar or foo > bar, that is actually running as a subprocess of your shell (bash, zsh, csh, etc.). When Consul Template spawns the exec process, it runs outside of your shell. This behavior is different from when Consul Template executes the template-specific reload command. If you want the ability to pipe or redirect in the exec command, you will need to spawn the process in subshell, for example:

exec {
  command = "$SHELL -c 'my-server > /var/log/my-server.log'"
}
Note that when spawning like this, most shells do not proxy signals to their child by default, so your child process will not receive the signals that Consul Template sends to the shell. You can avoid this by writing a tiny shell wrapper and executing that instead:

#!/usr/bin/env bash
trap "kill -TERM $child" SIGTERM

/bin/my-server -config /tmp/server.conf
child=$!
wait "$child"
Alternatively, you can use your shell's exec function directly, if it exists:

```
#!/usr/bin/env bash
exec /bin/my-server -config /tmp/server.conf > /var/log/my-server.log
```

There are some additional caveats with Exec Mode, which should be considered carefully before use:

If the child process dies, the Consul Template process will also die. Consul Template does not supervise the process! This is generally the responsibility of the scheduler or init system.
The child process must remain in the foreground. This is a requirement for Consul Template to manage the process and send signals.
The exec command will only start after all templates have been rendered at least once. One may have multiple templates for a single Consul Template process, all of which must be rendered before the process starts. Consider something like an nginx or apache configuration where both the process configuration file and individual site configuration must be written in order for the service to successfully start.
After the child process is started, any change to any dependent template will cause the reload signal to be sent to the child process. This reload signal defaults to nil, in which Consul Template will not kill and respawn the child. The reload signal can be specified and customized via the CLI or configuration file.
When Consul Template is stopped gracefully, it will send the configurable kill signal to the child process. The default value is SIGTERM, but it can be customized via the CLI or configuration file.
Consul Template will forward all signals it receives to the child process except its defined reload_signal, dump_signal, and kill_signal. If you disable these signals, Consul Template will forward them to the child process.
It is not possible to have more than one exec command (although each template can still have its own reload command).
Individual template reload commands still fire independently of the exec command.
De-Duplication Mode

Consul Template works by parsing templates to determine what data is needed and then watching Consul for any changes to that data. This allows Consul Template to efficiently re-render templates when a change occurs. However, if there are many instances of Consul Template rendering a common template there is a linear duplication of work as each instance is querying the same data.

To make this pattern more efficient Consul Template supports work de-duplication across instances. This can be enabled with the -dedup flag or via the deduplicate configuration block. Once enabled, Consul Template uses leader election on a per-template basis to have only a single node perform the queries. Results are shared among other instances rendering the same template by passing compressed data through the Consul K/V store.

Please note that no Vault data will be stored in the compressed template. Because ACLs around Vault are typically more closely controlled than those ACLs around Consul's KV, Consul Template will still request the secret from Vault on each iteration.

### Termination on Error

By default Consul Template is highly fault-tolerant. If Consul is unreachable or a template changes, Consul Template will happily continue running. The only exception to this rule is if the optional command exits non-zero. In this case, Consul Template will also exit non-zero. The reason for this decision is so the user can easily configure something like Upstart or God to manage Consul Template as a service.

If you want Consul Template to continue watching for changes, even if the optional command argument fails, you can append || true to your command. For example:

```
$ consul-template \
  -template "in.ctmpl:out.file:service nginx restart || true"
```

In this example, even if the Nginx restart command returns non-zero, the overall function will still return an OK exit code; Consul Template will continue to run as a service. Additionally, if you have complex logic for restarting your service, you can intelligently choose when you want Consul Template to exit and when you want it to continue to watch for changes. For these types of complex scripts, we recommend using a custom sh or bash script instead of putting the logic directly in the consul-template command or configuration file.

#### Command Environment

The current processes environment is used when executing commands with the following additional environment variables:

```
CONSUL_HTTP_ADDR
CONSUL_HTTP_TOKEN
CONSUL_HTTP_AUTH
CONSUL_HTTP_SSL
CONSUL_HTTP_SSL_VERIFY
```
These environment variables are exported with their current values when the command executes. Other Consul tooling reads these environment variables, providing smooth integration with other Consul tools (like consul maint or consul lock). Additionally, exposing these environment variables gives power users the ability to further customize their command script.

### Multi-phase Execution

Consul Template does an n-pass evaluation of templates, accumulating dependencies on each pass. This is required due to nested dependencies, such as:

```
{{range services}}
{{range service .Name}}
  {{.Address}}
{{end}}{{end}}
```
During the first pass, Consul Template does not know any of the services in Consul, so it has to perform a query. When those results are returned, the inner-loop is then evaluated with that result, potentially creating more queries and watches.

Because of this implementation, template functions need a default value that is an acceptable parameter to a range function (or similar), but does not actually execute the inner loop (which would cause a panic). This is important to mention because complex templates must account for the "empty" case. For example, the following will not work:

```
{{with index (service "foo") 0}}
# ...
{{end}}
```
This will raise an error like:

<index $services 0>: error calling index: index out of range: 0
That is because, during the first evaluation of the template, the service key is returning an empty slice. You can account for this in your template like so:

```
{{if service "foo"}}
{{with index (service "foo") 0}}
{{.Node}}
{{ end }}
{{ end }}
```

This will still add the dependency to the list of watches, but Go will not evaluate the inner-if, avoiding the out-of-index error.

## Examples

#### HAProxy

HAProxy is a very common load balancer. You can read more about the HAProxy configuration file syntax in the HAProxy documentation, but here is an example template for rendering an HAProxy configuration file with Consul Template:

```
global
    daemon
    maxconn {{key "service/haproxy/maxconn"}}

defaults
    mode {{key "service/haproxy/mode"}}{{range ls "service/haproxy/timeouts"}}
    timeout {{.Key}} {{.Value}}{{end}}

listen http-in
    bind *:8000{{range service "release.web"}}
    server {{.Node}} {{.Address}}:{{.Port}}{{end}}
```

Save this file to disk as haproxy.ctmpl and run the consul-template daemon:

```
$ consul-template \
  -consul demo.consul.io \
  -template haproxy.ctmpl:/etc/haproxy/haproxy.conf
  -dry
```

Depending on the state of the demo Consul instance, you could see the following output:

```
global
    daemon
    maxconn 4

defaults
    mode default
    timeout 5

listen http-in
    bind *:8000
    server nyc3-worker-2 104.131.109.224:80
    server nyc3-worker-3 104.131.59.59:80
    server nyc3-worker-1 104.131.86.92:80
```

For more information on how to save this result to disk or for the full list of functionality available inside a Consul template file, please consult the API documentation.

#### Varnish

Varnish is an common caching engine that can also act as a proxy. You can read more about the Varnish configuration file syntax in the Varnish documentation, but here is an example template for rendering a Varnish configuration file with Consul Template:

```
import directors;
{{range service "consul"}}
backend {{.Name}}_{{.ID}} {
    .host = "{{.Address}}";
    .port = "{{.Port}}";
}{{end}}

sub vcl_init {
  new bar = directors.round_robin();
{{range service "consul"}}
  bar.add_backend({{.Name}}_{{.ID}});{{end}}
}

sub vcl_recv {
  set req.backend_hint = bar.backend();
}
```

Save this file to disk as varnish.ctmpl and run the consul-template daemon:

```
$ consul-template \
  -consul demo.consul.io \
  -template varnish.ctmpl:/etc/varnish/varnish.conf \
  -dry
  
```
You should see the following output:

```
import directors;

backend consul_consul {
    .host = "104.131.109.106";
    .port = "8300";"
}

sub vcl_init {
  new bar = directors.round_robin();

  bar.add_backend(consul_consul);
}

sub vcl_recv {
  set req.backend_hint = bar.backend();
}
```

#### Apache httpd

Apache httpd is a popular web server. You can read more about the Apache httpd configuration file syntax in the Apache httpd documentation, but here is an example template for rendering part of an Apache httpd configuration file that is responsible for configuring a reverse proxy with dynamic end points based on service tags with Consul Template:

```
{{range $tag, $service := service "web" | byTag}}
# "{{$tag}}" api providers.
<Proxy balancer://{{$tag}}>
{{range $service}}  BalancerMember http://{{.Address}}:{{.Port}}
{{end}} ProxySet lbmethod=bybusyness
</Proxy>
Redirect permanent /api/{{$tag}} /api/{{$tag}}/
ProxyPass /api/{{$tag}}/ balancer://{{$tag}}/
ProxyPassReverse /api/{{$tag}}/ balancer://{{$tag}}/
{{end}}
```

Just like the previous examples, save this file to disk and run the consul-template daemon:

```
$ consul-template \
  -consul demo.consul.io \
  -template httpd.ctmpl:/etc/httpd/sites-available/balancer.conf
```

You should see output similar to the following:

```
# "frontend" api providers.
<Proxy balancer://frontend>
  BalancerMember http://104.131.109.106:8080
  BalancerMember http://104.131.109.113:8081
  ProxySet lbmethod=bybusyness
</Proxy>
Redirect permanent /api/frontend /api/frontend/
ProxyPass /api/frontend/ balancer://frontend/
ProxyPassReverse /api/frontend/ balancer://frontend/

# "api" api providers.
<Proxy balancer://api>
  BalancerMember http://104.131.108.11:8500
  ProxySet lbmethod=bybusyness
</Proxy>
Redirect permanent /api/api /api/api/
ProxyPass /api/api/ balancer://api/
ProxyPassReverse /api/api/ balancer://api/
```

### Querying all services


As of Consul Template 0.6.0, it is possible to have a complex dependency graph with dependent services. As such, it is possible to query and watch all services in Consul:

```
{{range services}}# {{.Name}}{{range service .Name}}
{{.Address}}{{end}}

{{end}}
```

Just like the previous examples, save this file to disk and run the consul-template daemon:

```
$ consul-template \
  -consul demo.consul.io \
  -template everything.ctmpl:/tmp/inventory
```

You should see output similar to the following:

```
# consul
104.131.121.232

# redis
104.131.86.92
104.131.109.224
104.131.59.59

# web
104.131.86.92
104.131.109.224
104.131.59.59
```

Running and Process Lifecycle

While there are multiple ways to run Consul Template, the most common pattern is to run Consul Template as a system service. When Consul Template first starts, it reads any configuration files and templates from disk and loads them into memory. From that point forward, changes to the files on disk do not propagate to running process without a reload.

The reason for this behavior is simple and aligns with other tools like haproxy. A user may want to perform pre-flight validation checks on the configuration or templates before loading them into the process. Additionally, a user may want to update configuration and templates simultaneously. Having Consul Template automatically watch and reload those files on changes is both operationally dangerous and against some of the paradigms of modern infrastructure. Instead, Consul Template listens for the SIGHUP syscall to trigger a configuration reload. If you update configuration or templates, simply send HUP to the running Consul Template process and Consul Template will reload all the configurations and templates from disk.

### Debugging ###

Consul Template can print verbose debugging output. To set the log level for Consul Template, use the -log-level flag:
```
$ consul-template -log-level info ...
<timestamp> [INFO] (cli) received redis from Watcher
<timestamp> [INFO] (cli) invoking Runner
# ...
```

You can also specify the level as debug:


```
$ consul-template -log-level debug ...
<timestamp> [DEBUG] (cli) creating Runner
<timestamp> [DEBUG] (cli) creating Consul API client
<timestamp> [DEBUG] (cli) creating Watcher
<timestamp> [DEBUG] (cli) looping for data
<timestamp> [DEBUG] (watcher) starting watch
<timestamp> [DEBUG] (watcher) all pollers have started, waiting for finish
<timestamp> [DEBUG] (redis) starting poll
<timestamp> [DEBUG] (service redis) querying Consul with &{...}
<timestamp> [DEBUG] (service redis) Consul returned 2 services
<timestamp> [DEBUG] (redis) writing data to channel
<timestamp> [DEBUG] (redis) starting poll
<timestamp> [INFO] (cli) received redis from Watcher
<timestamp> [INFO] (cli) invoking Runner
<timestamp> [DEBUG] (service redis) querying Consul with &{...}
# ...
```

## FAQ

### Q: How is this different than confd?
A: The answer is simple: Service Discovery as a first class citizen. You are also encouraged to read this Pull Request on the project for more background information. We think confd is a great project, but Consul Template fills a missing gap. Additionally, Consul Template has first class integration with Vault, making it easy to incorporate secret material like database credentials or API tokens into configuration files.

### Q: How is this different than Puppet/Chef/Ansible/Salt?
A: Configuration management tools are designed to be used in unison with Consul Template. Instead of rendering a stale configuration file, use your configuration management software to render a dynamic template that will be populated by Consul.

## Contributing

To build and install Consul Template locally, you will need a modern Go (Go 1.5+) environment.

First, clone the repo:

```
$ git clone https://github.com/hashicorp/consul-template.git
```

Next, download/update all the dependencies:

```
$ make updatedeps
```

To compile the consul-template binary and run the test suite:

```
$ make dev
```

This will compile the consul-template binary into bin/consul-template as well as your $GOPATH and run the test suite.

If you just want to run the tests:

```
$ make test
```

Or to run a specific test in the suite:

```
go test ./... -run SomeTestFunction_name
```

Submit Pull Requests and Issues to the Consul Template project on GitHub.







