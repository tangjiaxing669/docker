# Project Calico 实现

### Project Calico 简介

Project Calico 是一个纯三层协议，支持VM、Docker、Rocket、OpenStack、Kubernetes，也可直接在物理机上使用。其[官网](https://www.projectcalico.org/)指出可以支持上万个主机、上百万的工作负载；更易于调试，支持IPv6以及灵活的安全策略。

Project Calico 是纯三层的 SDN 实现，它基于 BGP 协议和 Linux 自己的路由转发机制，不依赖特殊硬件，没有使用 NAT 和 Tunnel 等技术。它自带的基于 iptables 的 ACL 管理组件非常灵活，能够满足比较复杂的安全隔离需求。

### 实践样例

本次使用两台 `CentOS 7.2` 的机器来实现，详情如下：

|IP           |HOSTNAME|Docker Version|Etcd version              |Calicoctl version|
|:-----------:|:------:|:------------:|:------------------------:|:---------------:|
|172.30.16.121| slave3 |    1.12.1    |cluster 3.0.0/server 3.0.9|0.22.0-dev       |
|172.30.16.122| slave2 |    1.12.1    |cluster 3.0.0/server 3.0.9|0.22.0-dev       |

Docker 的安装请参照[官网](https://docs.docker.com/engine/installation/linux/centos/#/install)。

### Requirements

- Docker 版本在 v1.6 或更高
- [etcd](https://github.com/tangjiaxing669/docker/blob/master/etcd.md)，推荐集群方式部署
- 需要加载`ipset`，`iptables` 以及 `ip6tables` 内核模块
- 下载 `calicoctl` 可执行文件到你的 `$PATH`

> **Note**：`Calico` 默认将配置信息存储到 `etcd` 的 `http://127.0.0.1:2379/v2/keys/calico` 中；如果你的 `etcd` 服务没有部署在本地，那么你可以通过 `export ETCD_AUTHORITY=172.30.16.123:2379` 来指定 `etcd` 服务的地址，然后再执行 `calicoctl node --ip=...` 等操作步骤。

### 下载 `calicoctl` 二进制文件

需要在两个节点上都下载`calicoctl`二进制文件。

```shell
[root@slave712 ~]# wget http://www.projectcalico.org/builds/calicoctl
[root@slave712 ~]# chmod +x calicoctl
[root@slave712 ~]# mv calicoctl /usr/bin/calicoctl

[root@slave713 ~]# wget http://www.projectcalico.org/builds/calicoctl
[root@slave713 ~]# chmod +x calicoctl
[root@slave713 ~]# mv calicoctl /usr/bin/calicoctl

[root@slave712 ~]# calicoctl version
0.22.0-dev

[root@slave712 ~]# calicoctl node show
+----------+---------------+-----------+-------------------+--------------+--------------+
| Hostname |   Bird IPv4   | Bird IPv6 |       AS Num      | BGP Peers v4 | BGP Peers v6 |
+----------+---------------+-----------+-------------------+--------------+--------------+
| slave712 | 172.30.16.122 |           | 64511 (inherited) |              |              |
| slave713 | 172.30.16.121 |           | 64511 (inherited) |              |              |
+----------+---------------+-----------+-------------------+--------------+--------------+

[root@slave712 ~]# calicoctl status
calico-node container is running. Status: Up 5 hours
Running felix version 1.4.2.dev1

IPv4 BGP status
IP: 172.30.16.122    AS Number: 64511 (inherited)
+---------------+-------------------+-------+----------+-------------+
|  Peer address |     Peer type     | State |  Since   |     Info    |
+---------------+-------------------+-------+----------+-------------+
| 172.30.16.121 | node-to-node mesh |   up  | 00:46:32 | Established |
+---------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 address configured.
```

### 配置 etcd 集群

分别在两台机器上下载 etcd 二进制文件，并放到合适的路径：

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

分别启动`slave712`和`slave713`上的`etcd`服务:

slave712:

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

......

[root@slave712 etcd-v3.0.9-linux-amd64]# http get http://172.30.16.121:2379/version
HTTP/1.1 200 OK
Content-Length: 44
Content-Type: application/json
Date: Sat, 10 Dec 2016 03:07:55 GMT

{
    "etcdcluster": "3.0.0", 
    "etcdserver": "3.0.9"
}
```

slave713:

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

......

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

至此，etcd 配置完成。

### 配置 Calico

> **Note**：`Calico` 默认将配置信息存储到 `etcd` 的 `http://127.0.0.1:2379/v2/keys/calico` 中；如果你的 `etcd` 服务没有部署在本地，那么你可以通过 `export ETCD_AUTHORITY=172.30.16.123:2379` 来指定 `etcd` 服务的地址，然后再执行 `calicoctl node --ip=...` 等操作步骤。

- 第一步

```shell
# slave713
[root@slave713 ~]# calicoctl node --ip=172.30.16.121

# slave712
[root@slave713 ~]# calicoctl node --ip=172.30.16.122
```

完成后通过`docker ps`应该可以看到一个`calico`的容器，如下：

```shell
[root@slave713 mnt]# docker ps -a
CONTAINER ID   IMAGE                COMMAND               CREATED        STATUS       PORTS   NAMES
39229d314e55   calico/node:latest   "/sbin/start_runit"   21 hours ago   Up 21 hours          calico-node
```

- 第二步

为 Calico 网络添加可用的 IP Pool(任意一台 node 操作都可以，Calico 会通过 Etcd 服务进行同步)

```shell
# slave713
[root@slave713 ~]# calicoctl pool add 192.168.1.0/24 --ipip --nat-outgoing
[root@slave713 ~]# calicoctl pool show
+----------------+-------------------+
|   IPv4 CIDR    |      Options      |
+----------------+-------------------+
| 192.168.0.0/16 | ipip,nat-outgoing |
| 192.168.1.0/24 | ipip,nat-outgoing |
+----------------+-------------------+
+--------------------------+---------+
|        IPv6 CIDR         | Options |
+--------------------------+---------+
| fd80:24e2:f998:72d6::/64 |         |
+--------------------------+---------+
```

> **Note**： calico 默认自带有一个 IP Pool，范围是 192.168.0.0/16

【1】`--ipip`：选项表示支持跨子网的主机上的 Docker 间网络通信

【2】`--nat-outgoing`：选项表示允许 Docker Container 访问外网

- 第三步

开始启动 Container (以 nginx 为例)，并为 Container 在 Clico 中注册独立的 IP

```shell
# slave713
[root@slave713 ~]# docker run --net=none --name workerload-1 -tid nginx
[root@slave713 ~]# docker run --net=none --name workerload-2 -tid nginx

[root@slave713 ~]# calicoctl container add workerload-1 192.168.1.10
IP 192.168.1.10 added to workerload-1
[root@slave713 ~]# calicoctl container add workerload-2 192.168.1.20
IP 192.168.1.20 added to workerload-2

[root@slave713 ~]# docker exec workerload-1 hostname -I
192.168.1.10 
[root@slave713 ~]# docker exec workerload-2 hostname -I
192.168.1.20

# slave712
[root@slave712 ~]# docker run --net=none --name workerload-3 -tid nginx
[root@slave712 ~]# docker run --net=none --name workerload-4 -tid nginx

[root@slave712 ~]# calicoctl container add workerload-3 192.168.1.30
IP 192.168.1.30 added to workerload-3
[root@slave712 ~]# calicoctl container add workerload-4 192.168.1.40
IP 192.168.1.40 added to workerload-4

[root@slave712 ~]# docker exec workerload-4 hostname -I
192.168.1.40 
[root@slave712 ~]# docker exec workerload-3 hostname -I
192.168.1.30
```

- 第四步

使用 Calico 的 profile 来配置安全策略（ACL，任意一台 node 操作都可以，Calico 会通过 Etcd 服务进行同步)

```shell
# slave713
[root@slave713 ~]# calicoctl profile add PROF_1
[root@slave713 ~]# calicoctl profile add PROF_2
[root@slave713 ~]# calicoctl profile show
+------------+
|    Name    |
+------------+
|   PROF_1   |
|   PROF_2   |
+------------+
```

- 第五步

应用安全策略到 Container；注意，默认情况下，只有位于同一个安全策略中的 Container 才能相互通信

```shell
# slave713
[root@slave713 ~]# calicoctl container workerload-1 profile append PROF_1
Profile(s) PROF_1 appended.

[root@slave713 ~]# calicoctl container workerload-2 profile append PROF_2
Profile(s) PROF_2 appended.

# slave712
[root@slave712 ~]# calicoctl container workerload-3 profile append PROF_1
Profile(s) PROF_1 appended.

[root@slave712 ~]# calicoctl container workerload-4 profile append PROF_2
Profile(s) PROF_2 appended.
```

> **Note**：可以使用 `calicoctl profile show --detailed` 来查看哪个 Container 应用了哪个安全策略。

- 第六步
现在来测试下...

```shell
[root@slave712 ~]# docker exec workerload-3 ping -c2 192.168.1.10
PING 192.168.1.10 (192.168.1.10): 56 data bytes
64 bytes from 192.168.1.10: icmp_seq=0 ttl=62 time=1.525 ms
64 bytes from 192.168.1.10: icmp_seq=1 ttl=62 time=0.737 ms
--- 192.168.1.10 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.737/1.131/1.525/0.394 ms

[root@slave712 ~]# docker exec workerload-3 ping -c2 192.168.1.20
PING 192.168.1.20 (192.168.1.20): 56 data bytes
--- 192.168.1.20 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss

[root@slave712 ~]# docker exec workerload-3 ping -c2 192.168.1.30
PING 192.168.1.30 (192.168.1.30): 56 data bytes
64 bytes from 192.168.1.30: icmp_seq=0 ttl=64 time=0.312 ms
64 bytes from 192.168.1.30: icmp_seq=1 ttl=64 time=0.209 ms
--- 192.168.1.30 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.209/0.261/0.312/0.052 ms

[root@slave712 ~]# docker exec workerload-3 ping -c2 192.168.1.40
PING 192.168.1.40 (192.168.1.40): 56 data bytes
--- 192.168.1.40 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss

[root@slave712 ~]# docker exec workerload-3 ping -c2 www.google.com
PING www.google.com (43.245.144.113): 56 data bytes
64 bytes from 43.245.144.113: icmp_seq=0 ttl=58 time=3.601 ms
64 bytes from 43.245.144.113: icmp_seq=1 ttl=58 time=3.052 ms
--- www.google.com ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 3.052/3.327/3.601/0.275 ms
```

> **Note**：位于同一个安全策略组下的 Container 可相互的**PING**通，而位于不同的安全策略组下的 Container 是不能通信的。

# 整合 Calico 到 Docker Network

Calico 可以被整合到 Docker Network 中，但是需求 Docker 版本在 v1.9 以上才行；Calico 会运行一个额外的 Container 来作为 Docker network 的插件，并整合到 Docker的 `docker network` 命令中。

> **Note**：整合 Calico 需要 `Docker Engine` 运行在集群模式下，同样也是用 `etcd` 来存储配置信息。`etcd` 服务还是参照之前的。

- 第一步

停止两个节点( `slave712` / `slave713` )的 Docker Daemon，并配置集群参数

```shell
# slave712
[root@slave712 ~]# grep -v '^#' /usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target

[Service]
Type=notify
ExecStart=/usr/bin/docker daemon -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --cluster-store=etcd://172.30.16.122:2379  --cluster-advertise=172.30.16.122:2375
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target

# slave713
[root@slave713 ~]# grep -v '^#' /usr/lib/systemd/system/docker.service 
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target

[Service]
Type=notify
ExecStart=/usr/bin/docker daemon -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --cluster-store=etcd://172.30.16.121:2379  --cluster-advertise=172.30.16.121:2375
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target
```

> **Note**：主要修改 `ExecStart=` 这一行记录

【1】`--cluster-store=` 选项表示分布式的存储后端地址

【2】`--cluster-advertise` 对外通告的地址和接口

然后重新加载 `systemd` 并重启 `docker` 服务

```shell
# slave712
[root@slave712 ~]# systemctl daemon-reload
[root@slave712 ~]# systemctl restart docker
[root@slave712 ~]# systemctl status docker
[root@slave712 ~]# ps aux | grep dockerd
root       1119  0.4  3.6 740060 36636 ?        Ssl  17:23   0:37 dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --cluster-store=etcd://172.30.16.122:2379 --cluster-advertise=172.30.16.122:2375

# slave713
[root@slave713 ~]# systemctl daemon-reload
[root@slave713 ~]# systemctl restart docker
[root@slave713 ~]# systemctl status docker
[root@slave713 ~]# ps aux | grep dockerd
root       1438  7.5  1.7 739772 34804 ?        Ssl  19:57   0:01 dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --cluster-store=etcd://172.30.16.121:2379 --cluster-advertise=172.30.16.121:2375
```

看到如上信息，说明 Docker 的集群配置已经完成，并且已启动成功。

- 第二步

使用 `--libnetwork` 参数来运行 `Calico`

```shell
# slave712
[root@slave712 ~]# calicoctl node --libnetwork --ip=172.30.16.122
[root@slave712 ~]# docker ps
CONTAINER ID        IMAGE                           COMMAND               CREATED                  STATUS              PORTS               NAMES
d5fd2b0f99cb        calico/node-libnetwork:latest   "./start.sh"          Less than a second ago   Up 2 hours                              calico-libnetwork
c9606f6a5370        calico/node:latest              "/sbin/start_runit"   Less than a second ago   Up 2 hours                              calico-node

# slave713
[root@slave713 ~]# calicoctl node --libnetwork --ip=172.30.16.121
[root@slave713 ~]# docker ps
CONTAINER ID        IMAGE                           COMMAND               CREATED             STATUS              PORTS               NAMES
e5e3b46eed8c        calico/node-libnetwork:latest   "./start.sh"          45 hours ago        Up 7 minutes                            calico-libnetwork
39229d314e55        calico/node:latest              "/sbin/start_runit"   45 hours ago        Up 7 minutes                            calico-node
```

- 第三步

使用 `docker network` 为 `calico` 创建一个逻辑网络，并指定相应的 IP Pool(任意一台 node 操作都可以，所有配置会通过 Etcd 服务进行同步)

```shell
[root@slave713 ~]# docker network create --driver=calico --opt ipip=true --opt nat-outgoing=true --subnet=192.168.100.0/24 calico1
8ebaea68aa82bea884f396c3ac87224d4ef84737e0662565afd3ef0335161f25

[root@slave713 ~]# docker network create --driver=calico --opt ipip=true --opt nat-outgoing=true --subnet=192.168.200.0/24 calico2
ba32a3b095fa27ced9faf40996f50baf1b8f358f8a5434f0577d686943550a88

[root@slave713 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
c4313c40a76f        bridge              bridge              local               
8ebaea68aa82        calico1             calico              global              
ba32a3b095fa        calico2             calico              global              
53cce434a584        docker_gwbridge     bridge              local               
a14063f778eb        host                host                local               
ee95dfd25e2c        none                null                local  

[root@slave713 ~]# calicoctl pool show
+------------------+-------------------+
|    IPv4 CIDR     |      Options      |
+------------------+-------------------+
|  192.168.0.0/16  | ipip,nat-outgoing |
|  192.168.1.0/24  | ipip,nat-outgoing |
| 192.168.100.0/24 | ipip,nat-outgoing |
| 192.168.200.0/24 | ipip,nat-outgoing |
+------------------+-------------------+
+--------------------------+---------+
|        IPv6 CIDR         | Options |
+--------------------------+---------+
| fd80:24e2:f998:72d6::/64 |         |
+--------------------------+---------+
```

> 上面输出看到，`calico` 也将 `docker network` 创建的网络信息同步过来了。

- 第四步

现在使用上面创建的网络来启动 Container

```shell
# slave713 
[root@slave713 ~]# docker run --net calico1 --name workerload-1 -tid nginx
0fc95a4bb27a696b1f62affa96a6a32d5e848cbfafabcf382ac199f317eaf256

[root@slave713 ~]# docker run --net calico2 --name workerload-2 -tid nginx
66a445288519e52c3b150ba970dd402aa28d04dbfc402658485950a3f0c6cc1d

[root@slave713 ~]# docker ps -a
CONTAINER ID        IMAGE                           COMMAND                  CREATED             STATUS              PORTS               NAMES
66a445288519        nginx                           "nginx -g 'daemon off"   3 minutes ago       Up 3 minutes        80/tcp, 443/tcp     workerload-2
0fc95a4bb27a        nginx                           "nginx -g 'daemon off"   3 minutes ago       Up 3 minutes        80/tcp, 443/tcp     workerload-1
e5e3b46eed8c        calico/node-libnetwork:latest   "./start.sh"             45 hours ago        Up 30 minutes                           calico-libnetwork
39229d314e55        calico/node:latest              "/sbin/start_runit"      45 hours ago        Up 30 minutes                           calico-node

# slave712
[root@slave712 ~]# docker run --net calico1 --name workerload-3 -tid nginx
273dfcc1ae88734e00554746d02af1775e541a3dd6d3a877ad9b57bc2a5c5622

[root@slave712 ~]# docker run --net calico2 --name workerload-4 -tid nginx
cc15245e3d3d5f20320af4c96193523327d6276f9255e81ae8bca97655e3ed32

[root@slave712 ~]# docker ps -a
CONTAINER ID        IMAGE                           COMMAND                  CREATED                  STATUS              PORTS               NAMES
d5fd2b0f99cb        calico/node-libnetwork:latest   "./start.sh"             Less than a second ago   Up 3 hours                              calico-libnetwork
c9606f6a5370        calico/node:latest              "/sbin/start_runit"      Less than a second ago   Up 3 hours                              calico-node
cc15245e3d3d        nginx                           "nginx -g 'daemon off"   34 seconds ago           Up 31 seconds       80/tcp, 443/tcp     workerload-4
273dfcc1ae88        nginx                           "nginx -g 'daemon off"   44 seconds ago           Up 40 seconds       80/tcp, 443/tcp     workerload-3
```

> **Note**：你也可以使用如下命令为 Container 指定固定 IP

```shell
[root@slave712 ~]# docker run --net calico2 --name workerload-4 --ip 192.168.200.* -tid nginx
```

现在我们看一下 Container 的 IP...

```shell
# slave713
[root@slave713 ~]# docker exec workerload-1 hostname -I
192.168.100.2 172.18.0.2 

[root@slave713 ~]# docker exec workerload-2 hostname -I
192.168.200.2 172.18.0.3

# slave712
[root@slave712 ~]# docker exec workerload-3 hostname -I
192.168.100.3 172.19.0.2 

[root@slave712 ~]# docker exec workerload-4 hostname -I
192.168.200.3 172.19.0.3
```

- 第五步

现在我们来测试下......

```shell
[root@slave713 ~]# docker exec workerload-1 ping -c2 192.168.100.2
PING 192.168.100.2 (192.168.100.2): 56 data bytes
64 bytes from 192.168.100.2: icmp_seq=0 ttl=64 time=0.157 ms
64 bytes from 192.168.100.2: icmp_seq=1 ttl=64 time=0.769 ms
--- 192.168.100.2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.157/0.463/0.769/0.306 ms

[root@slave713 ~]# docker exec workerload-1 ping -c2 192.168.200.2
PING 192.168.200.2 (192.168.200.2): 56 data bytes
--- 192.168.200.2 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss

[root@slave713 ~]# docker exec workerload-1 ping -c2 192.168.100.3
PING 192.168.100.3 (192.168.100.3): 56 data bytes
64 bytes from 192.168.100.3: icmp_seq=0 ttl=62 time=0.916 ms
64 bytes from 192.168.100.3: icmp_seq=1 ttl=62 time=0.632 ms
--- 192.168.100.3 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.632/0.774/0.916/0.142 ms

[root@slave713 ~]# docker exec workerload-1 ping -c2 192.168.200.3
PING 192.168.200.3 (192.168.200.3): 56 data bytes
--- 192.168.200.3 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss
```

> **Note**：效果跟之前一样，位于同一网络中的主机可以通信，而不在同一网络中的主机则不可以通信。

- 第六步

现在我们来分析路由是怎么走的；先看看 `container` 和本地的路由信息。

workerload-1 Container 内路由信息:
```shell
# slave713
[root@slave713 ~]# docker exec workerload-1 ip route show
default via 172.18.0.1 dev eth1 
172.18.0.0/16 dev eth1  proto kernel  scope link  src 172.18.0.2 
192.168.100.0/24 dev cali0  proto kernel  scope link  src 192.168.100.2 
```

本机路由信息
```shell
[root@slave713 ~]# ip route show
172.18.0.0/16 dev docker_gwbridge  proto kernel  scope link  src 172.18.0.1 
blackhole 192.168.1.0/26  proto bird 
192.168.60.192/26 via 172.30.16.122 dev tunl0  proto bird onlink 
192.168.100.0/26 via 172.30.16.122 dev tunl0  proto bird onlink 
192.168.100.2 dev calide59e9d45c7  scope link 
blackhole 192.168.135.192/26  proto bird 
192.168.200.2 dev calid4945a71845  scope link 
192.168.200.3 via 172.30.16.122 dev tunl0  proto bird onlink
```

本机网卡信息
```shell
calid4945a71845: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::b024:2cff:fe44:d447  prefixlen 64  scopeid 0x20<link>
        ether b2:24:2c:44:d4:47  txqueuelen 1000  (Ethernet)
        RX packets 2914  bytes 244840 (239.1 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2938  bytes 246408 (240.6 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

calide59e9d45c7: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::8094:1bff:fe14:4b6e  prefixlen 64  scopeid 0x20<link>
        ether 82:94:1b:14:4b:6e  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

docker_gwbridge: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.18.0.1  netmask 255.255.0.0  broadcast 0.0.0.0
        inet6 fe80::42:97ff:fe3d:c9f3  prefixlen 64  scopeid 0x20<link>
        ether 02:42:97:3d:c9:f3  txqueuelen 0  (Ethernet)
        RX packets 8  bytes 648 (648.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 9  bytes 690 (690.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eno16777736: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.30.16.121  netmask 255.255.255.0  broadcast 172.30.16.255
        inet6 fe80::20c:29ff:fea8:cd65  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:a8:cd:65  txqueuelen 1000  (Ethernet)
        RX packets 578550  bytes 65894367 (62.8 MiB)
        RX errors 0  dropped 82  overruns 0  frame 0
        TX packets 545354  bytes 53342969 (50.8 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

现在我们看看从 `workerload-1` 中执行 `ping -c2 192.168.100.3` 是怎么走的：

> **Note**：每启动一个 Container，就会在你本机上创建一块网卡并将其指向你所创建的 Container（例如，**workerload-1**对应**calide59e9d45c7**；**workerload-2**对应**calid4945a71845**）；我们再来看看本机的路由表，`slave713` 上的 **workerload-1**（192.168.100.2） 要访问 `slave712` 上的 **workerload-3**（192.168.100.3），通过路由表得知需要将包转给 `172.30.16.122`，也就是`slave712`。类似如下结构：

```shell
workerload-1[cali0] -> slave713[calid4945a71845] -> route -> slave712[cali3b1b4ff6fbe] -> workerload-3[cali0]
```

![](https://github.com/tangjiaxing669/docker/blob/master/picture/1.bmp)

### Calico 的高级网络策略

```shell
[root@slave712 ~]# calicoctl profile PROF_1 rule show
Inbound rules:
   1 allow from tag PROF_1
Outbound rules:
   1 allow
```

也有 `json` 格式的显示样例

```shell
[root@slave712 ~]# calicoctl profile PROF_1 rule json
{
  "inbound_rules": [
    {
      "action": "allow", 
      "src_tag": "PROF_1"
    }
  ], 
  "outbound_rules": [
    {
      "action": "allow"
    }
  ]
}
```

这个规则表示入连接中，只允许来自 `profile` 名是 `PROF_1` 的实例，出连接没有限制；最后还有一条隐藏的默认规则是不匹配的全部 Drop；所以，只有位于同一 profile 下的 Container 才能通信；

可以使用 `calicoctl profile --help` 或者访问[官网](https://github.com/projectcalico/calico-containers/blob/master/docs/AdvancedNetworkPolicy.md)来查看更详细的用法。
