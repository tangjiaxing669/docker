# etcd 使用入门

###1、etcd 简介

coreos开发的分布式服务系统，内部采用**raft**协议作为一致性算法。作为服务发现系统，有以下的特点：

- 简单：安装配置简单，而且提供了HTTP API进行交互，使用也很简单
- 安全：支持SSL证书验证
- 快速：根据官方提供的**[benchmark数据](https://coreos.com/etcd/docs/latest/benchmarks/etcd-2-2-0-rc-benchmarks.html)**单实例支持每秒2K+读操作
- 可靠：采用**raft**算法，实现分布式系统数据的可用性和一致性

> 下面所有的例子将会在`etcd v3.0.9`版本中实现，系统版本为`CentOS 7.2`。

etcd目前默认使用`2379`端口提供HTTP API服务，`2380`端口和peer通信（这两个端口已经被IANA官方预留给etcd）；在之前的版本中，可能会分别使用`4001`和`7001`，在使用的过程中需要注意这个区别。

虽然etcd也支持单点部署，但是在生产环境中推荐集群方式部署，一般etcd节点数会选择`3`、`5`、`7`。etcd会保证所有的节点都会保存数据，并保证数据的一致性和正确性。

###2、安装

因为etcd是`go`语言编写的，安装只需要下载对应的二进制文件，并放到合适的路径就行。例如：

```shell
[root@slave713 ~]# cd
[root@slave713 ~]# curl -L https://github.com/coreos/etcd/releases/download/v3.0.9/etcd-v3.0.9-linux-amd64.tar.gz -o etcd-v3.0.9-linux-amd64.tar.gz
[root@slave713 ~]# tar xzvf etcd-v3.0.9-linux-amd64.tar.gz && cd etcd-v3.0.9-linux-amd64
etcd-v3.0.9-linux-amd64/
etcd-v3.0.9-linux-amd64/etcd
etcd-v3.0.9-linux-amd64/README.md
etcd-v3.0.9-linux-amd64/READMEv2-etcdctl.md
etcd-v3.0.9-linux-amd64/etcdctl
etcd-v3.0.9-linux-amd64/Documentation/
etcd-v3.0.9-linux-amd64/Documentation/v2/
etcd-v3.0.9-linux-amd64/Documentation/v2/dev/
......

[root@slave713 etcd-v3.0.9-linux-amd64]# ll
total 37744
drwxr-xr-x 11 9003 9003     4096 Sep 15 16:43 Documentation
-rwxr-xr-x  1 9003 9003 20165472 Sep 15 16:43 etcd
-rwxr-xr-x  1 9003 9003 18427584 Sep 15 16:43 etcdctl
-rw-r--r--  1 9003 9003    29370 Sep 15 16:43 README-etcdctl.md
-rw-r--r--  1 9003 9003     5628 Sep 15 16:43 README.md
-rw-r--r--  1 9003 9003     7935 Sep 15 16:43 READMEv2-etcdctl.md
[root@slave713 etcd-v3.0.9-linux-amd64]# ./etcd --version
etcd Version: 3.0.9
Git SHA: 494c012
Go Version: go1.6.3
Go OS/Arch: linux/amd64
[root@slave713 etcd-v3.0.9-linux-amd64]# 
```

至此，etcd安装完成。下面我们来试着启动一个单点的etcd服务（只需要运行`etcd`命令就行）。

```shell
[root@slave713 etcd-v3.0.9-linux-amd64]# ./etcd
2016-09-18 17:34:50.899048 I | etcdmain: etcd Version: 3.0.9
2016-09-18 17:34:50.899354 I | etcdmain: Git SHA: 494c012
2016-09-18 17:34:50.899371 I | etcdmain: Go Version: go1.6.3
2016-09-18 17:34:50.899385 I | etcdmain: Go OS/Arch: linux/amd64
2016-09-18 17:34:50.899406 I | etcdmain: setting maximum number of CPUs to 4, total number of available CPUs is 4
2016-09-18 17:34:50.899425 W | etcdmain: no data-dir provided, using default data-dir ./default.etcd
2016-09-18 17:34:50.928377 I | etcdmain: listening for peers on http://localhost:2380
2016-09-18 17:34:50.928925 I | etcdmain: listening for client requests on localhost:2379
2016-09-18 17:34:50.940112 I | etcdserver: name = default
2016-09-18 17:34:50.940166 I | etcdserver: data dir = default.etcd
2016-09-18 17:34:50.940184 I | etcdserver: member dir = default.etcd/member
2016-09-18 17:34:50.940319 I | etcdserver: heartbeat = 100ms
2016-09-18 17:34:50.940341 I | etcdserver: election = 1000ms
2016-09-18 17:34:50.940354 I | etcdserver: snapshot count = 10000
2016-09-18 17:34:50.940422 I | etcdserver: advertise client URLs = http://localhost:2379
2016-09-18 17:34:50.940448 I | etcdserver: initial advertise peer URLs = http://localhost:2380
2016-09-18 17:34:50.940494 I | etcdserver: initial cluster = default=http://localhost:2380
2016-09-18 17:34:50.981028 I | etcdserver: starting member 8e9e05c52164694d in cluster cdf818194e3a8c32
2016-09-18 17:34:50.981212 I | raft: 8e9e05c52164694d became follower at term 0
2016-09-18 17:34:50.981342 I | raft: newRaft 8e9e05c52164694d [peers: [], term: 0, commit: 0, applied: 0, lastindex: 0, lastterm: 0]
2016-09-18 17:34:50.981371 I | raft: 8e9e05c52164694d became follower at term 1
2016-09-18 17:34:51.030524 I | etcdserver: starting server... [version: 3.0.9, cluster version: to_be_decided]
2016-09-18 17:34:51.032612 I | membership: added member 8e9e05c52164694d [http://localhost:2380] to cluster cdf818194e3a8c32
2016-09-18 17:34:51.182694 I | raft: 8e9e05c52164694d is starting a new election at term 1
2016-09-18 17:34:51.182779 I | raft: 8e9e05c52164694d became candidate at term 2
2016-09-18 17:34:51.182807 I | raft: 8e9e05c52164694d received vote from 8e9e05c52164694d at term 2
2016-09-18 17:34:51.182877 I | raft: 8e9e05c52164694d became leader at term 2
2016-09-18 17:34:51.182907 I | raft: raft.node: 8e9e05c52164694d elected leader 8e9e05c52164694d at term 2
2016-09-18 17:34:51.184966 I | etcdserver: published {Name:default ClientURLs:[http://localhost:2379]} to cluster cdf818194e3a8c32
2016-09-18 17:34:51.185028 I | etcdserver: setting up the initial cluster version to 3.0
2016-09-18 17:34:51.185167 I | etcdmain: ready to serve client requests
2016-09-18 17:34:51.187575 E | etcdmain: forgot to set Type=notify in systemd service file?
2016-09-18 17:34:51.206074 N | etcdmain: serving insecure client requests on localhost:2379, this is strongly discouraged!
2016-09-18 17:34:51.250660 N | membership: set the initial cluster version to 3.0
2016-09-18 17:34:51.250847 I | api: enabled capabilities for version 3.0
```

从上面的输出中，我们可以看到很多信息：

- etcd默认将数据存放到当前路径的`default.etcd/`目录下面
- 在`http://localhost:2380`和集群中其它节点通信
- 在`http://localhost:2379`提供HTTP API服务，供客户端交互
- 改节点的名称默认为`default`
- `heartbeat`为`100ms`，后面会说明这个配置的作用
- `election`为`1000ms`，后面会说明这个配置的作用
- `snapshot count`为`10000`，后面会说明这个配置的作用
- 集群和每个节点都会生成一个`uuid`
- 启动的时候，会运行`raft`，选举出`leader`

###3、集群信息

在安装和启动etcd服务的时候，各个节点需要知道集群中其它节点的信息（一般是`IP`和`Port`信息）。根据你是否可以提前知道每个节点的IP，有几种不同的启动方案：

- 静态配置：在启动`etcd server`的时候，通过`--initial-cluster`参数配置好所有的节点信息
- 使用已有的`etcd cluster`来注册和启动，比如官方提供的`discovery.etcd.io`
- 使用**DNS**启动

etcd的安装文档上面已经给出，当然，你也可以通过Docker来安装etcd，具体的文档可以看[这里](https://coreos.com/etcd/docs/latest/docker_guide.html)。

下面给出常用的配置参数和他们的解释，方便理解：

- `--name`：方便理解的节点名称，默认为`default`，在集群中应该保持唯一，可以使用**hostname**
- `--data-dir`：服务运行时数据保存的路径，默认为`${name}.etcd`
- `--snapshot-count`：指定有多少事物（transaction）被提交时，触发截取快照保存到磁盘
- `--heartbeat-interval`：**leader**多久发送一次心跳到**followers**。默认值是`100ms`
- `--eletion-timeout`：重新投票的超时时间，如果**follow**在该事件间隔没有收到心跳包，会触发重新投票，默认为`100ms`
- `--listen-peer-urls`：和同伴通信的地址，比如`http://ip:2380`，如果有多个，使用逗号分隔。需要所有节点都能够访问，**素以不要使用 localhost**
- `--listen-client-urls`：对外提供服务的地址；比如`http://ip:2379, http://127.0.0.1:2379`，客户端会连接到这里和etcd交互
- `--advertise-client-urls`： 对外公告的该节点客户端监听地址，这个值会告诉集群中其它节点
- `--initial-advertise-peer-urls`：该节点同伴监听地址，这个值会告诉集群中其它节点
- `--initial-cluster`：集群中所有节点的信息，格式为`node1=http://ip1:2380, node2=http://ip2:2380, ...`。注意，这里的`node1`是节点的`--name`指定的名字；后面的`ip1:2380`是`--initial-advertise-peer-urls`指定的值
- `--initial-cluster-state`：新建集群的时候，这个值为`new`；假如是已经存在的集群，这个值为`existing`
- `--initial-cluster-token`：创建集群的`token`，这个值每个集群保持唯一。这样的话，如果你要重新创建集群，即使配置和之前一样，也会再次生成新的集群和节点uuid；否则会导致多个集群之间的冲突，造成未知的错误

> *NOTE*：所有以`--init`开头的配置都是在 bootstrap 集群的时候才会用到，后续节点的重启会被忽略。

> *NOTE*：所有的参数也可以通过环境变量进行设置，`--my-flag`对应的环境变量的`ETCD_MY_FLAG`；但是命令行指定的参数会覆盖环境变量对应的值；

###4、etcd基础知识

每个`etcd cluster`都是由若干个**member**组成的，每个**member**是一个独立运行的`etcd`实例，单台机器上可以运行多个**member**。

在正常运行的状态下，集群中会有一个leader，其余的member都是followers。leader向followers同步日志，保证数据在各个member都有副本。leader还会定时向所有的member发送心跳报文，如果在规定的时间里follower没有收到心跳，就会重新进行选举。

客户端所有的请求都会先发送给leader，leader向所有的followers同步日志，等收到超过半数的确认后就把该日志存储到磁盘，并返回响应客户端。

每个etcd服务由三大主要部分组成：`raft实现`、`WAL日志存储`、`数据的存储和索引`。WAL会在本地磁盘（就是之前提到的`--data-dir`）上存储日志内容（wal file）和快照（snapshot）。

###5、API 文档

etcd 通过 HTTP API对外提供服务，这种方式方便测试（通过curl或者其他工具就能和etcd交互），也很容易集成到各种语言中（每个语言封装 HTTP API 实现自己的 client 就行）。

下面我们就介绍 etcd 通过 HTTP API 提供了哪些功能，并使用`http`来交互；

http安装：
```shell
yum install -y python-setuptools
easy_install pip
pip install httpie
```

首先，我们在两台机器上用集群的方式来启动etcd，如下：

第一台
```shell
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcd \
--name jasontom0 \
--initial-advertise-peer-urls http://172.30.16.122:2380 \
--listen-peer-urls http://172.30.16.122:2380 \
--listen-client-urls http://172.30.16.122:2379,http://127.0.0.1:2379 \
--advertise-client-urls http://172.30.16.122:2379 \
--initial-cluster-token etcd-cluster-1 \
--initial-cluster jasontom0=http://172.30.16.122:2380,jasontom1=http://172.30.16.121:2380 \
--initial-cluster-state new
```

第二台

```shell
[root@slave713 etcd-v3.0.9-linux-amd64]# ./etcd \
--name jasontom1 \
--initial-advertise-peer-urls http://172.30.16.121:2380 \
--listen-peer-urls http://172.30.16.121:2380 \
--listen-client-urls http://172.30.16.121:2379,http://127.0.0.1:2379 \
--advertise-client-urls http://172.30.16.121:2379 \
--initial-cluster-token etcd-cluster-1 \
--initial-cluster jasontom0=http://172.30.16.122:2380,jasontom1=http://172.30.16.121:2380 \
--initial-cluster-state new
```
####获取 etcd 服务的版本信息

```shell
[root@slave713 etcd-v3.0.9-linux-amd64]# http get http://172.30.16.122:2379/version
HTTP/1.1 200 OK
Content-Length: 44
Content-Type: application/json
Date: Sat, 10 Dec 2016 03:07:55 GMT

{
    "etcdcluster": "3.0.0", 
    "etcdserver": "3.0.9"
}
```

####key 的增删查改

etcd 的数据按照树形的结构组织，类似于Linux的文件系统，也有目录和文件的区别，不过一般被称为nodes。数据的endpoint都是以`/v2/keys`开头（v2表示当前API的版本），比如`/v2/keys/names/cizixs`。

要创建一个值，只要使用`PUT`方法在对应的 url endpoint 设置就行。如果对应的key已经存在，`PUT`也会对key进行更新。

```shell
[root@slave713 etcd-v3.0.9-linux-amd64]# http put http://127.0.0.1:2379/v2/keys/hello value=="hello, etcd"
HTTP/1.1 201 Created
Content-Length: 100
Content-Type: application/json
Date: Sun, 18 Sep 2016 23:10:16 GMT
X-Etcd-Cluster-Id: 321081c7f73246ae
X-Etcd-Index: 18
X-Raft-Index: 3269
X-Raft-Term: 68

{
    "action": "set", 
    "node": {
        "createdIndex": 18, 
        "key": "/hello", 
        "modifiedIndex": 18, 
        "value": "hello, etcd"
    }
}
```

上面这个命令通过`PUT`方法把`/message`设置为`hello, etcd`。返回的格式中，各个字段的意思是：

- `action`：请求触发的动作，这里因为是新建一个 key 并设置它的值，所以是 `set`
- `node.key`：key 的 HTTP 路径(URI)
- `node.value`：请求处理之后 key 的值
- `node.createdIndex`：createdIndex是一个递增的值，每次有 key 被创建时都会增加
- `node.modifiedIndex`：同上，只不过每次有 key 被修改的时候增加

除了返回的 json 体外，上面的情况还包含了一些特殊的 HTTP 头部信息，这些信息说明了 etcd cluster 的一些情况。他们具体的含义如下：

- `X-Etcd-Index`：当前 etcd 集群的 index
- `X-Raft-Index`：raft 集群的 index
- `X-Raft-Term`：raft 集群的任期，每次有 leader 选举的时候，这个值就会增加

查看信息比较简单，使用`GET`方法，url 指向要查看的值就行：

```shell
[root@slave713 etcd-v3.0.9-linux-amd64]# http  get http://172.30.16.121:2379/v2/keys/hello
HTTP/1.1 200 OK
Content-Length: 100
Content-Type: application/json
Date: Sun, 18 Sep 2016 23:17:08 GMT
X-Etcd-Cluster-Id: 321081c7f73246ae
X-Etcd-Index: 18
X-Raft-Index: 4093
X-Raft-Term: 68

{
    "action": "get", 
    "node": {
        "createdIndex": 18, 
        "key": "/hello", 
        "modifiedIndex": 18, 
        "value": "hello, etcd"
    }
}
```

这里的`action`变成了`get`，其它返回值就和上面的含义一样了；

前面已经提供，`PUT`也可以用来更新 key 的值，例如：

```shell
[root@slave713 etcd-v3.0.9-linux-amd64]# http put http://127.0.0.1:2379/v2/keys/hello value=="hello, world"
HTTP/1.1 200 OK
Content-Length: 188
Content-Type: application/json
Date: Sun, 18 Sep 2016 23:21:38 GMT
X-Etcd-Cluster-Id: 321081c7f73246ae
X-Etcd-Index: 19
X-Raft-Index: 4634
X-Raft-Term: 68

{
    "action": "set", 
    "node": {
        "createdIndex": 19, 
        "key": "/hello", 
        "modifiedIndex": 19, 
        "value": "hello, world"
    }, 
    "prevNode": {
        "createdIndex": 18, 
        "key": "/hello", 
        "modifiedIndex": 18, 
        "value": "hello, etcd"
    }
}
```

和第一次执行`PUT`命令不同的是，返回中多了一个字段`prevNode`，它保存着更新之前该 key 的信息。他的格式和`node`是一样的，如果之前没有这个信息，这个字段会被省略。

删除 key 可以通过`DELETE`方法，比如：

```shell
[root@slave713 etcd-v3.0.9-linux-amd64]# http delete http://172.30.16.121:2379/v2/keys/message
HTTP/1.1 200 OK
Content-Length: 172
Content-Type: application/json
Date: Sun, 18 Sep 2016 23:31:49 GMT
X-Etcd-Cluster-Id: 321081c7f73246ae
X-Etcd-Index: 20
X-Raft-Index: 5857
X-Raft-Term: 68

{
    "action": "delete", 
    "node": {
        "createdIndex": 17, 
        "key": "/message", 
        "modifiedIndex": 20
    }, 
    "prevNode": {
        "createdIndex": 17, 
        "key": "/message", 
        "modifiedIndex": 17, 
        "value": "hello, etcd"
    }
}
```

注意，这个`action`是`delete`。

####TTL属性

etcd 中，key 可以有 TTL 属性，超过这个时间会被自动删除。我们来设置一个看看：

```shell
[root@slave713 etcd-v3.0.9-linux-amd64]# http put http://127.0.0.1:2379/v2/keys/tempkey value=="template key" ttl==5
HTTP/1.1 201 Created
Content-Length: 157
Content-Type: application/json
Date: Mon, 19 Sep 2016 00:21:16 GMT
X-Etcd-Cluster-Id: 321081c7f73246ae
X-Etcd-Index: 21
X-Raft-Index: 11792
X-Raft-Term: 68

{
    "action": "set", 
    "node": {
        "createdIndex": 21, 
        "expiration": "2016-09-19T00:21:21.483611836Z", 
        "key": "/tempkey", 
        "modifiedIndex": 21, 
        "ttl": 5, 
        "value": "template key"
    }
}
```

除了一般 key 返回的信息之外，上面多了两个字段：

- `expiration`：代表 key 过期被删除的时间
- `ttl`：表示 key 还要多少秒可以存活（这个值是动态的，会根据你请求的时间和过期的时间进行计算）

如果我们在 5s 之后再去请求查看该 key，会发现报错信息：

```shell
[root@slave713 etcd-v3.0.9-linux-amd64]# http http://172.30.16.121:2379/v2/keys/template
HTTP/1.1 404 Not Found
Content-Length: 75
Content-Type: application/json
Date: Mon, 19 Sep 2016 00:30:50 GMT
X-Etcd-Cluster-Id: 321081c7f73246ae
X-Etcd-Index: 39

{
    "cause": "/template", 
    "errorCode": 100, 
    "index": 39, 
    "message": "Key not found"
}
```

http 返回为 `404`，并且返回体中给出了 `errorCode` 和错误信息。

TTL 也可通过`PUT`方法进行取消，只要设置空值`ttl==`就行，这样 key 就不会过期被删除。比如：

```shell
[root@slave713 etcd-v3.0.9-linux-amd64]# http put http://172.30.16.121:2379/v2/keys/template1 value==bar ttl== prevExist==True
```

> **NOTE：需要设置 value==bar，不然 key 会变成空值**

> **如果只是想更新 TTL，可以添加上 `refresh==true` 参数**

####监听变化

etcd 提供了监听的机制，可以让客户端使用 long pulling 监听某个 key，当发生变化的时候接受通知因为；因为 etcd 经常被用作服务发现，集群中的信息有更新的时候需要及时被检测，做出对应的处理。因此需要有监听机制来告诉客户端特定 key 的变化情况。

监听动作只需要 `GET` 方法添加上 `wait==true` 参数就行。使用 `recursive==true`(没测试成功) 参数也能监听某个目录。

```shell
[root@slave713 etcd-v3.0.9-linux-amd64]# http put http://172.30.16.121:2379/v2/keys/foo wait==true
HTTP/1.1 200 OK
Content-Type: application/json
Date: Mon, 19 Sep 2016 01:16:05 GMT
Transfer-Encoding: chunked
X-Etcd-Cluster-Id: 321081c7f73246ae
X-Etcd-Index: 43
X-Raft-Index: 18383
X-Raft-Term: 68
```

这个时候，客户端会阻塞在这里，如果在另外的 terminal 修改 key 的值，监听的客户端会接收到消息，打印出更新的值：

```json
{
    "action": "set", 
    "node": {
        "createdIndex": 44, 
        "key": "/foo", 
        "modifiedIndex": 44, 
        "value": "test23 34df"
    }, 
    "prevNode": {
        "createdIndex": 43, 
        "key": "/foo", 
        "modifiedIndex": 43, 
        "value": "test23 34"
    }
}
```

除了这种最简单的监听之外，还可以提供基于 index 的监听。如果通过 `waitIndex` 指定了 index，那么会返回从 index 开始出现的第一个事件，这包含了两种情况：

- 给出的 index 小于等于当前 index，即事件已经发生，那么监听会立即返回该事件
- 给出的 index 大于当前 index，等待 index 之后的时间发生并返回

目前 etcd 只会保存最近 1000 个事件（整个集群范围内），再早之前的事件会被清理，如果监听被清理的事件会报错。如果出现漏过太多事件（超过1000）的情况，需要重新获取当前的 index 值（`X-Etcd-Index`），然后从`X-Etcd-Index + 1`开始监听。

因为监听的时候如果事件被触发就会直接返回，因此需要客户端编写循环逻辑保持监听状态。在两次监听的间隔中出现的事件，很可能被漏过。所以最好把时间处理逻辑做成异步的，不要阻塞监听逻辑。

> NOTE: **监听 key 时会出现因为长时间没有返回导致连接被 close 的情况，客户端需要处理这种错误并自动重试。**

####自动创建有序的 keys

在有些情况下，我们需要 key 是有序的，etcd 提供了这个功能。对某个目录使用 `POST` 方法，能自动生成有序的 key，这种模式可以用于队列处理等场景。

```shell
[root@slave713 etcd-v3.0.9-linux-amd64]# http POST http://172.30.16.121:2379/v2/keys/queue value==job1
HTTP/1.1 201 Created
Content-Length: 117
Content-Type: application/json
Date: Mon, 19 Sep 2016 02:02:16 GMT
X-Etcd-Cluster-Id: 321081c7f73246ae
X-Etcd-Index: 45
X-Raft-Index: 23927
X-Raft-Term: 68

{
    "action": "create", 
    "node": {
        "createdIndex": 45, 
        "key": "/queue/00000000000000000045", 
        "modifiedIndex": 45, 
        "value": "job1"
    }
}
```

创建的 key 会使用 etcd index，只能保证递增，无法保证是连续的（因为两次创建之间可能会有其它事件发生）。然后用相同的命令创建多个值，在获取值的时候使用`sorted==true`参数就会返回已经排序的值：

```shell
[root@slave713 etcd-v3.0.9-linux-amd64]# http http://172.30.16.122:2379/v2/keys/queue sorted==true
HTTP/1.1 200 OK
Content-Length: 549
Content-Type: application/json
Date: Sat, 10 Dec 2016 06:06:06 GMT
X-Etcd-Cluster-Id: 321081c7f73246ae
X-Etcd-Index: 49
X-Raft-Index: 24479
X-Raft-Term: 68

{
    "action": "get", 
    "node": {
        "createdIndex": 45, 
        "dir": true, 
        "key": "/queue", 
        "modifiedIndex": 45, 
        "nodes": [
            {
                "createdIndex": 45, 
                "key": "/queue/00000000000000000045", 
                "modifiedIndex": 45, 
                "value": "job1"
            }, 
            {
                "createdIndex": 46, 
                "key": "/queue/00000000000000000046", 
                "modifiedIndex": 46, 
                "value": "job2"
            }, 
            {
                "createdIndex": 47, 
                "key": "/queue/00000000000000000047", 
                "modifiedIndex": 47, 
                "value": "job3"
            }, 
            {
                "createdIndex": 48, 
                "key": "/queue/00000000000000000048", 
                "modifiedIndex": 48, 
                "value": "job4"
            }, 
            {
                "createdIndex": 49, 
                "key": "/queue/00000000000000000049", 
                "modifiedIndex": 49, 
                "value": "job5"
            }
        ]
    }
}
```

####设置目录的 TTL

和 key 类似，目录（dir）也可以有过期时间。设置的方法也一样，只不过多了 `dir==true` 参数来说明这是一个目录。

```shell
[root@slave713 etcd-v3.0.9-linux-amd64]# http put http://172.30.16.121:2379/v2/keys/dir dir==true ttl=5 prevExist==true
HTTP/1.1 201 Created
Content-Length: 111
Content-Type: application/json
Date: Mon, 19 Sep 2016 02:10:34 GMT
X-Etcd-Cluster-Id: 321081c7f73246ae
X-Etcd-Index: 50
X-Raft-Index: 24927
X-Raft-Term: 68

{
    "action": "create", 
    "node": {
        "createdIndex": 50, 
        "dir": true, 
        "key": "/dir/00000000000000000050", 
        "modifiedIndex": 50
    }
}
```

####比较更新的原子操作

在分布式环境中，我们需要解决多个客户端的竞争问题，etcd 提供了原子操作`CompareAndSwap`（CAS），通过这个操作可以很容易实现分布式锁。

简单来说，这个命令只有在客户端提供的条件成立的情况下才会更新对应的值。目前支持的条件包括：

- `prevValue`：检查 key 之间的值是否和客户端提供的一致
- `prevIndex`：检查 key 之间的 `modifiedIndex` 是否和客户端提供的一致
- `prevExixt`：检查 key 是否已经存在。如果存在就执行更新操作，如果不存在，执行 `create` 操作

举个例子，比如目前 `/foo` 的值为 `bar`，要把它更新成 `changed`，可以使用：

```shell
[root@slave712 etcd-v3.0.9-linux-amd64]# http put http://172.30.16.121:2379/v2/keys/foo prevValue==bar value==changed
HTTP/1.1 200 OK
Content-Length: 182
Content-Type: application/json
Date: Mon, 19 Sep 2016 02:31:01 GMT
X-Etcd-Cluster-Id: 321081c7f73246ae
X-Etcd-Index: 56
X-Raft-Index: 27390
X-Raft-Term: 68

{
    "action": "compareAndSwap", 
    "node": {
        "createdIndex": 55, 
        "key": "/foo", 
        "modifiedIndex": 56, 
        "value": "changed"
    }, 
    "prevNode": {
        "createdIndex": 55, 
        "key": "/foo", 
        "modifiedIndex": 55, 
        "value": "bar"
    }
}
```

如果提供的条件不对，会报 `412` 错误：

```shell
[root@slave712 etcd-v3.0.9-linux-amd64]# http put http://172.30.16.121:2379/v2/keys/foo prevValue==bar value==changed
HTTP/1.1 412 Precondition Failed
Content-Length: 83
Content-Type: application/json
Date: Mon, 19 Sep 2016 02:30:19 GMT
X-Etcd-Cluster-Id: 321081c7f73246ae
X-Etcd-Index: 54

{
    "cause": "[bar != changed]", 
    "errorCode": 101, 
    "index": 54, 
    "message": "Compare failed"
}
```

> **NOTE：匹配条件是 `prevIndex==0` 的话，也会通过检查。**

这些条件也可以组合起来使用，只有当都满足的时候，才会执行对应的操作。

####比较删除的原子操作

和条件更新类似，etcd 也支持条件删除操作，只有在客户端提供的条件成立的情况下，才会执行删除操作。支持`prevValue`和`prevIndex`两种条件检查，没有`prevExist`，因为删除不存在的值本身就会报错。

我们来删除上面例子中更新的`/foo`，先看一下提供的条件不对的情况：

```shell
[root@slave712 etcd-v3.0.9-linux-amd64]# http delete http://172.30.16.121:2379/v2/keys/foo prevValue==bar
HTTP/1.1 412 Precondition Failed
Content-Length: 83
Content-Type: application/json
Date: Mon, 19 Sep 2016 02:44:11 GMT
X-Etcd-Cluster-Id: 321081c7f73246ae
X-Etcd-Index: 56

{
    "cause": "[bar != changed]", 
    "errorCode": 101, 
    "index": 56, 
    "message": "Compare failed"
}
```

如果提供的条件成立，对应的 key 就会被删除：

```shell
[root@slave712 etcd-v3.0.9-linux-amd64]# http delete http://172.30.16.121:2379/v2/keys/foo prevValue==changed
HTTP/1.1 200 OK
Content-Length: 170
Content-Type: application/json
Date: Mon, 19 Sep 2016 02:44:55 GMT
X-Etcd-Cluster-Id: 321081c7f73246ae
X-Etcd-Index: 57
X-Raft-Index: 29059
X-Raft-Term: 68

{
    "action": "compareAndDelete", 
    "node": {
        "createdIndex": 55, 
        "key": "/foo", 
        "modifiedIndex": 57
    }, 
    "prevNode": {
        "createdIndex": 55, 
        "key": "/foo", 
        "modifiedIndex": 56, 
        "value": "changed"
    }
}
```

####操作目录

在创建 key 的时候，如果它所在路径的目录不存在，会自动被创建，所以在多数情况下我们不需要关心目录的创建。目录的操作和 key 的操作基本一致，唯一的区别是需要加上 `dir==true` 参数指明操作的对象是目录。

比如，如果想要显示的创建目录，可以使用 `PUT` 方法，并设置 `dir==true`：

```shell
[root@slave712 etcd-v3.0.9-linux-amd64]# http put http://172.30.16.121:2379/v2/keys/home dir==true
HTTP/1.1 201 Created
Content-Length: 88
Content-Type: application/json
Date: Mon, 19 Sep 2016 02:49:58 GMT
X-Etcd-Cluster-Id: 321081c7f73246ae
X-Etcd-Index: 58
X-Raft-Index: 29665
X-Raft-Term: 68

{
    "action": "set", 
    "node": {
        "createdIndex": 58, 
        "dir": true, 
        "key": "/home", 
        "modifiedIndex": 58
    }
}
```

创建目录的操作不能重复执行，再次执行上面的命令回报`HTTP 403`错误。

如果 `GET` 方法对应的 url 是目录的话，etcd 会列出该目录下所有节点的信息（不需要指定 `dir==true` ）。比如要列出根目录下所有的节点：

```shell
[root@slave712 etcd-v3.0.9-linux-amd64]# http http://172.30.16.121:2379/v2/keys
HTTP/1.1 200 OK
Content-Length: 384
Content-Type: application/json
Date: Mon, 19 Sep 2016 02:54:38 GMT
X-Etcd-Cluster-Id: 321081c7f73246ae
X-Etcd-Index: 58
X-Raft-Index: 30226
X-Raft-Term: 68

{
    "action": "get", 
    "node": {
        "dir": true, 
        "nodes": [
            {
                "createdIndex": 19, 
                "key": "/hello", 
                "modifiedIndex": 19, 
                "value": "hello, world"
            }, 
            {
                "createdIndex": 58, 
                "dir": true, 
                "key": "/home", 
                "modifiedIndex": 58
            }, 
            {
                "createdIndex": 45, 
                "dir": true, 
                "key": "/queue", 
                "modifiedIndex": 45
            }, 
            {
                "createdIndex": 50, 
                "dir": true, 
                "key": "/dir", 
                "modifiedIndex": 50
            }, 
            {
                "createdIndex": 42, 
                "key": "/template", 
                "modifiedIndex": 42, 
                "value": ""
            }
        ]
    }
}
```

如果添加上 `recursive==true` 参数，就会递归的列出所有的值：

```shell
[root@slave712 etcd-v3.0.9-linux-amd64]# http http://172.30.16.121:2379/v2/keys recursive==true
HTTP/1.1 200 OK
Content-Length: 938
Content-Type: application/json
Date: Mon, 19 Sep 2016 02:56:47 GMT
X-Etcd-Cluster-Id: 321081c7f73246ae
X-Etcd-Index: 58
X-Raft-Index: 30485
X-Raft-Term: 68

{
    "action": "get", 
    "node": {
        "dir": true, 
        "nodes": [
            {
                "createdIndex": 42, 
                "key": "/template", 
                "modifiedIndex": 42, 
                "value": ""
            }, 
            {
                "createdIndex": 19, 
                "key": "/hello", 
                "modifiedIndex": 19, 
                "value": "hello, world"
            }, 
            {
                "createdIndex": 58, 
                "dir": true, 
                "key": "/home", 
                "modifiedIndex": 58
            }, 
            {
                "createdIndex": 45, 
                "dir": true, 
                "key": "/queue", 
                "modifiedIndex": 45, 
                "nodes": [
                    {
                        "createdIndex": 46, 
                        "key": "/queue/00000000000000000046", 
                        "modifiedIndex": 46, 
                        "value": "job2"
                    }, 
                    {
                        "createdIndex": 47, 
                        "key": "/queue/00000000000000000047", 
                        "modifiedIndex": 47, 
                        "value": "job3"
                    }, 
                    {
                        "createdIndex": 48, 
                        "key": "/queue/00000000000000000048", 
                        "modifiedIndex": 48, 
                        "value": "job4"
                    }, 
                    {
                        "createdIndex": 49, 
                        "key": "/queue/00000000000000000049", 
                        "modifiedIndex": 49, 
                        "value": "job5"
                    }, 
                    {
                        "createdIndex": 45, 
                        "key": "/queue/00000000000000000045", 
                        "modifiedIndex": 45, 
                        "value": "job1"
                    }
                ]
            }, 
            {
                "createdIndex": 50, 
                "dir": true, 
                "key": "/dir", 
                "modifiedIndex": 50, 
                "nodes": [
                    {
                        "createdIndex": 50, 
                        "dir": true, 
                        "key": "/dir/00000000000000000050", 
                        "modifiedIndex": 50
                    }
                ]
            }
        ]
    }
}
```

和 linux 删除目录的设计一样，要区别空目录和非空目录。删除空目录很简单，使用 `DELETE` 方法，并添加上 `dir==true` 参数，类似于 `rmdir`；而对于非空目录，需要添加上 `recursive==true`，类似于 `rm -rf`。

```shell
[root@slave712 etcd-v3.0.9-linux-amd64]# http delete http://172.30.16.121:2379/v2/keys/queue dir==true
HTTP/1.1 403 Forbidden
Content-Length: 78
Content-Type: application/json
Date: Mon, 19 Sep 2016 03:00:33 GMT
X-Etcd-Cluster-Id: 321081c7f73246ae
X-Etcd-Index: 58

{
    "cause": "/queue", 
    "errorCode": 108, 
    "index": 58, 
    "message": "Directory not empty"
}

[root@slave712 etcd-v3.0.9-linux-amd64]# http delete http://172.30.16.121:2379/v2/keys/queue dir==true recursive==true
HTTP/1.1 200 OK
Content-Length: 168
Content-Type: application/json
Date: Mon, 19 Sep 2016 03:00:44 GMT
X-Etcd-Cluster-Id: 321081c7f73246ae
X-Etcd-Index: 59
X-Raft-Index: 30962
X-Raft-Term: 68

{
    "action": "delete", 
    "node": {
        "createdIndex": 45, 
        "dir": true, 
        "key": "/queue", 
        "modifiedIndex": 59
    }, 
    "prevNode": {
        "createdIndex": 45, 
        "dir": true, 
        "key": "/queue", 
        "modifiedIndex": 45
    }
}
```

####隐藏的节点

etcd 中节点也可以是隐藏的，类似于 linux 中以 `.` 开头的文件或者目录；而在 etcd 中以 `_` 开头的节点也是隐藏的，不会在列出目录的时候显示。只有知道隐藏节点的完整路径，才能够访问它的信息。

####查看集群数据信息

etcd 还保存了集群的数据信息，包括节点之间的网络信息，操作的统计信息。

- `/v2/stats/leader` 会返回集群中 leader 的信息，以及 followers 的基本信息

```shell
[root@slave712 etcd-v3.0.9-linux-amd64]# http http://172.30.16.121:2379/v2/stats/leader
HTTP/1.1 403 Forbidden
Content-Length: 32
Content-Type: application/json
Date: Mon, 19 Sep 2016 03:02:24 GMT

{
    "message": "not current leader"
}

[root@slave712 etcd-v3.0.9-linux-amd64]# http http://172.30.16.122:2379/v2/stats/leader
HTTP/1.1 200 OK
Content-Length: 241
Content-Type: application/json
Date: Sat, 10 Dec 2016 07:01:51 GMT

{
    "followers": {
        "a1f59b6c1c4d9277": {
            "counts": {
                "fail": 0, 
                "success": 82189
            }, 
            "latency": {
                "average": 0.011158177931353232, 
                "current": 0.013741, 
                "maximum": 9.097328, 
                "minimum": 0.001063, 
                "standardDeviation": 0.050009141831879614
            }
        }
    }, 
    "leader": "fe9a5154049cffbb"
}
```

- `/v2/stats/self` 会返回当前节点的信息

```shell
[root@slave712 etcd-v3.0.9-linux-amd64]# http http://172.30.16.122:2379/v2/stats/self
HTTP/1.1 200 OK
Content-Length: 362
Content-Type: application/json
Date: Sat, 10 Dec 2016 07:09:09 GMT

{
    "id": "fe9a5154049cffbb", 
    "leaderInfo": {
        "leader": "fe9a5154049cffbb", 
        "startTime": "2016-12-09T21:42:27.88347131-05:00", 
        "uptime": "4h26m41.259814822s"
    }, 
    "name": "jasontom0", 
    "recvAppendRequestCnt": 0, 
    "sendAppendRequestCnt": 84462, 
    "sendBandwidthRate": 389.2318169284011, 
    "sendPkgRate": 5.192526906728936, 
    "startTime": "2016-12-09T21:40:48.874469626-05:00", 
    "state": "StateLeader"
}
```

- `/v2/stats/store` 会返回各种命令的统计信息

```shell
[root@slave712 etcd-v3.0.9-linux-amd64]# http http://172.30.16.122:2379/v2/stats/store
HTTP/1.1 200 OK
Content-Length: 292
Content-Type: application/json
Date: Sat, 10 Dec 2016 07:07:29 GMT

{
    "compareAndDeleteFail": 0, 
    "compareAndDeleteSuccess": 2, 
    "compareAndSwapFail": 1, 
    "compareAndSwapSuccess": 1, 
    "createFail": 0, 
    "createSuccess": 8, 
    "deleteFail": 1, 
    "deleteSuccess": 2, 
    "expireCount": 10, 
    "getsFail": 32, 
    "getsSuccess": 2, 
    "setsFail": 1, 
    "setsSuccess": 37, 
    "updateFail": 1, 
    "updateSuccess": 0, 
    "watchers": 0
}
```

####成员管理

etcd 在 `/v2/members` 下保存这集群中各个成员的信息

```shell
[root@slave712 etcd-v3.0.9-linux-amd64]# http http://172.30.16.121:2379/v2/members
HTTP/1.1 200 OK
Content-Length: 272
Content-Type: application/json
Date: Mon, 19 Sep 2016 03:14:54 GMT
X-Etcd-Cluster-Id: 321081c7f73246ae

{
    "members": [
        {
            "clientURLs": [
                "http://172.30.16.121:2379"
            ], 
            "id": "a1f59b6c1c4d9277", 
            "name": "jasontom1", 
            "peerURLs": [
                "http://172.30.16.121:2380"
            ]
        }, 
        {
            "clientURLs": [
                "http://172.30.16.122:2379"
            ], 
            "id": "fe9a5154049cffbb", 
            "name": "jasontom0", 
            "peerURLs": [
                "http://172.30.16.122:2380"
            ]
        }
    ]
}
```

可以通过 `POST` 方法添加成员：

```shell
[root@slave712 etcd-v3.0.9-linux-amd64]# http post http://10.0.0.10:2379/v2/members 
-H "Content-Type: application/json" -d '{"peerURLs":["http://10.0.0.10:2380"]}'
```

也可以通过 `DELETE` 方法删除成员：

```shell
[root@slave712 etcd-v3.0.9-linux-amd64]# http delete http://10.0.0.10:2379/v2/members/272e204152
```

或者通过 `PUT` 更新成员的 peer url：

```shell
[root@slave712 etcd-v3.0.9-linux-amd64]# http put http://10.0.0.10:2379/v2/members/272e204152  -H "Content-Type: application/json" -d '{"peerURLs":["http://10.0.0.10:2380"]}'
```

###6、etcdctl 命令行工具

除了 HTTP API 外，etcd 还提供了 `etcdctl` 命令行工具和 etcd 服务交互。这个命令行用 go 语言编写，也是对 HTTP API 的封装，日常使用起来也更容易。

```shell
# 设置一个 key 的值
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl set name "jasontom"
jasontom

# 获取 key 的值
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl get name
jasontom

# 获取包含元数据的 key 的值
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl -o extended get name
Key: /name
Created-Index: 60
Modified-Index: 60
TTL: 0
Index: 60

jasontom

# 获取不存在的 key 值会报错
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl get notexist
Error:  100: Key not found (/notexist) [60]

# 设置 key 的 ttl 过期后会自动删除
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl set tempkey "tempkey" --ttl 5
tempkey
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl get tempkey
tempkey
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl get tempkey
Error:  100: Key not found (/tempkey) [64]

# 如果 key 的值是 “hello etcd”，就把它替换为 “goodbye， etcd”
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl set --swap-with-value "hello, world" name "goodbye, etcd"
Error:  101: Compare failed ([hello, world != jasontom]) [64]
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl set --swap-with-value "jasontom" name "goodbye, etcd"
goodbye, etcd

# 仅当 key 不存在的时候创建
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl mk foo bar
bar
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl mk foo bar
Error:  105: Key already exists (/foo) [66]

# 自动创建排序的 key
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl mk --in-order queue job1
job1
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl mk --in-order queue job2
job2
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl ls --sort queue
/queue/00000000000000000067
/queue/00000000000000000068

# 更新 key 的值或者 ttl，只有当 key 已经存在的时候才会生效，否则报错
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl get foo
bar
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl update foo "jasontom"
jasontom
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl get foo
jasontom
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl update notexist "jasontom"
Error:  100: Key not found (/notexist) [69]
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl update --ttl 3 foo "lance"
lance
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl get foo
Error:  100: Key not found (/foo) [71]

# 删除某个 key
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl rm foo
PrevNode.Value: bar
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl get foo
Error:  100: Key not found (/foo) [73]

# 只有当 key 的值匹配时才进行删除
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl rm --with-value mistake foo
Error:  101: Compare failed ([mistake != bar]) [74]
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl rm --with-value bar foo

# 创建一个目录
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl mkdir dir
Error:  105: Key already exists (/dir) [75]
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl mkdir dir1

# 删除空目录
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl rmdir dir1

# 删除非空目录
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl rmdir dir
Error:  108: Directory not empty (/dir) [77]
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl rm --recursive dir

# 列出目录的内容
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl ls /
/queue
/template
/hello
/home
/name
/dir

# 递归列出目录的内容
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl ls --recursive /
/template
/hello
/home
/name
/dir
/dir/queue
/dir/queue/00000000000000000087
/dir/queue/00000000000000000088
/dir/queue/00000000000000000089
/dir/queue/00000000000000000086
/queue
/queue/00000000000000000067
/queue/00000000000000000068

# 监听某个 key，当 key 改变的时候会打印出变化
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl watch foo
bar

# 监听某个目录，当目录中任何 node 改变的时候，都会打印出来
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl watch --recursive /
[set] /message
changed

# 一直监听，除非 CTL+C 导致退出监听
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl watch --forever /message
new value
chaned again
Wola

# 监听目录，并在发生变化的时候执行一个命令
[root@slave712 etcd-v3.0.9-linux-amd64]# ./etcdctl exec-watch --recursive / -- sh -c "echo change detected."
change detected.
change detected.
```

###7、总结
etcd 默认只保存 1000 个历史事件，所以不适合有大量更新操作的场景，这样会导致数据的丢失。etcd 典型的应用场景是配置管理和服务发现，这些场景都是读多写少。

相比于 zookeeper，etcd 使用起来要简单很多。不过要实现真正的服务发现功能，etcd还需要和其他工具（比如registrator、confd等）一起使用来实现服务的自动注册和更新。

目前 etcd 还没有图形化工具。
