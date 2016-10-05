# Harbor 使用

### 安装和配置

Harbor 有两种安装方式：

- 在线安装：安装的时候会自动从 `Docker Hub` 下载所需的 `image`。
- 离线安装：在没有联网的情况下可采用这种方式来安装 `Harbor`，安装的同时会预编译所需的 `image`。

> **Note:** [下载地址](https://github.com/vmware/harbor/releases)

### Prerequisites

1. `Python` 版本 >= 2.7
2. `Docker Engine` 版本 >= 1.10；[官网下载](https://docs.docker.com/engine/installation/)。
3. `Docker Compose` 版本 >= 1.6.0；[官网下载](https://docs.docker.com/compose/install/)。

### 安装步骤

> **Note:** 此处以 `harbor-online-installer-0.4.0.tgz` 为例。

- [下载安装包](https://github.com/vmware/harbor/releases)
```shell
[root@slave713 ~]# wget https://github.com/vmware/harbor/releases/download/0.4.0/harbor-online-installer-0.4.0.tgz
[root@slave713 ~]# tar xf harbor-online-installer-0.4.0.tgz
```
- 配置 **harbor.cfg**
```shell
[root@slave713 ~]# cd harbor
[root@slave713 harbor]# grep -v '^#' harbor.cfg | grep -v '^$'
hostname = 172.30.16.121
ui_url_protocol = http
email_server = smtp.mydomain.com
email_server_port = 25
email_username = sample_admin@mydomain.com
email_password = abc
email_from = admin <sample_admin@mydomain.com>
email_ssl = false
harbor_admin_password = harbor12345
auth_mode = db_auth
ldap_url = ldaps://ldap.mydomain.com
ldap_basedn = ou=people,dc=mydomain,dc=com
ldap_uid = uid 
ldap_scope = 3 
db_password = root123
self_registration = off
use_compressed_js = on
max_job_workers = 3 
secret_key = secretkey1234567
token_expiration = 30
verify_remote_cert = off
customize_crt = off
crt_country = CN
crt_state = State
crt_location = CN
crt_organization = organization
crt_organizationalunit = organizational unit
crt_commonname = example.com
crt_email = example@example.com
```
- 安装 `docker-compose`
```shell
[root@slave713 harbor]# curl -L https://github.com/docker/compose/releases/download/1.8.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
[root@slave713 harbor]# chmod +x /usr/local/bin/docker-compose
```
- 执行 **install.sh** 安装脚本并且启动 `Harbor`
```shell
[root@slave713 harbor]# ./install.sh 
……
……
[root@slave713 harbor]# docker ps -a
CONTAINER ID        IMAGE                            COMMAND                  CREATED             STATUS                      PORTS                                      NAMES
0d220c0d5088        vmware/harbor-jobservice:0.4.0   "/harbor/harbor_jobse"   3 days ago          Up 3 days                                                              harbor_jobservice_1
6efa2481e7cc        library/nginx:1.9.0              "nginx -g 'daemon off"   3 days ago          Up 3 days                   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp   harbor_proxy_1
1bbad67e3163        vmware/harbor-ui:0.4.0           "/harbor/harbor_ui"      3 days ago          Up 3 days                                                              harbor_ui_1
8e659471e860        library/registry:2.5.0           "/entrypoint.sh serve"   3 days ago          Up 3 days                   5000/tcp                                   harbor_registry_1
556ceab555de        vmware/harbor-db:0.4.0           "/entrypoint.sh mysql"   3 days ago          Up 3 days                   3306/tcp                                   harbor_mysql_1
f03e98a94586        rsyslog:latest                   "/start"                 3 days ago          Up 3 days                   0.0.0.0:1514->514/tcp                      harbor_log_1
```

> 至此，`Harbor` 安装配置已完成；可以通过 `http://hostname` 登录到 `Harbor`，默认用户名和密码为 `admin:Harbor12345`。

> 详细配置信息可咨询[此处](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md)。

### 管理 **Harbor**

1. 停止 Harbor
```shell
[root@slave713 harbor]# docker-compose stop
Stopping harbor_proxy_1 ... done
Stopping harbor_ui_1 ... done
Stopping harbor_registry_1 ... done
Stopping harbor_mysql_1 ... done
Stopping harbor_log_1 ... done
Stopping harbor_jobservice_1 ... done
```

2. 启动 Harbor
```shell
[root@slave713 harbor]# docker-compose start
Starting harbor_log_1
Starting harbor_mysql_1
Starting harbor_registry_1
Starting harbor_ui_1
Starting harbor_proxy_1
Starting harbor_jobservice_1
```

3. 修改 Harbor 配置文件需要先停止已存在的 Harbor 实例，再更新配置文件，最后再运行 `install.sh` 脚本
```shell
[root@slave713 harbor]# docker-compose down
...
[root@slave713 harbor]# vim harbor.cfg
...
[root@slave713 harbor]# ./install.sh
...
```

4. 移出 Harbor 镜像和其数据文件
```shell
[root@slave713 harbor]# docker-compose rm
Going to remove harbor_proxy_1, harbor_ui_1, harbor_registry_1, harbor_mysql_1, harbor_log_1, harbor_jobservice_1
Are you sure? [yN] y
Removing harbor_proxy_1 ... done
Removing harbor_ui_1 ... done
Removing harbor_registry_1 ... done
Removing harbor_mysql_1 ... done
Removing harbor_log_1 ... done
Removing harbor_jobservice_1 ... done

# 移出数据库文件和镜像数据（清除操作，之后需要`re-installtion`）
[root@slave713 harbor]# rm -r /data/database
[root@slave713 harbor]# rm -r /data/registry
```

> **Note:** 更多操作可以查看[Docker Compose command-line reference](https://docs.docker.com/compose/reference/)

### harbor_log_1 修改

原始的 `Harbor log` 是通过 `CMD cron && rsyslogd -n` 来作为 `syslog` 的 `daemon`；当容器重启后会出现 `PID` 文件残留的异常，导致 `Harbor_log_1` 启动不了；做法是修改 `Dockerfile` 文件重新编译 `Harbor log` 镜像。如下：

```shell
[root@slave713 log]# cat /home/harbor/Deploy/log/Dockerfile
FROM library/ubuntu:14.04

# run logrotate hourly, disable imklog model, provides TCP/UDP syslog reception
RUN mv /etc/cron.daily/logrotate /etc/cron.hourly/ \
        && rm /etc/rsyslog.d/* \
        && rm /etc/rsyslog.conf
ADD rsyslog.conf /etc/rsyslog.conf

# logrotate configuration file for docker
ADD logrotate_docker.conf /etc/logrotate.d/

# rsyslog configuration file for docker
ADD rsyslog_docker.conf /etc/rsyslog.d/

# start script
ADD start /start
RUN chmod 755 /start

VOLUME /var/log/docker/

EXPOSE 514

CMD ["/start"]
```

```shell
[root@slave713 log]# cat /home/harbor/Deploy/log/start
#!/bin/bash

if [ -e /var/run/rsyslogd.pid ]; then
    rm -rf /var/run/rsyslogd.pid
fi

cron && rsyslogd -n
```

然后重新编译 `Dockerfile` 并修改 `docker-compose.yml` 文件的 **log** 镜像地址为你编译后的镜像即可；使用方法依旧。

至此完成。

> **Note:** 其他的一些使用方法请参照[官方地址](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md)。

### 操作注意

- `docker daemon` 配置（这种配置会优先`/usr/lib/systemcd/system/docker.service`中的配置）

```shell
$ sudo mkdir /etc/systemd/system/docker.service.d
$ sudo vi /etc/systemd/system/docker.service.d/docker.conf
```
然后再docker.conf文件中添加启动参数，例如(添加无TLS认证的仓库地址)：
```shell
[Service]  
ExecStart=  
ExecStart=/usr/bin/docker daemon --insecure-registry=172.30.16.122:80

systemctl daemon-reload
```
也可以直接修改 `docker.service` 文件：
```shell
[root@slave713 harbor]# vim /usr/lib/systemd/system/docker.service
```

- 登录仓库(注意，默认的**80**端口也要加上)

```shell
docker login 172.30.16.122:80
username: admin
password: Harbor12345
```

- push image

```shell   
[root@slave712 Deploy]# docker tag nginx 172.30.16.122:80/myproject/myrepo
# myproject 是 harbor 项目名，可登录 Harbor web界面创建
# myrepo 代表仓库名，需要先将 image tag 成相应的名字；例如 docker tag nginx 172.30.16.122:80/myproject/myrepo1
# 此时，Harbor 中就有了 myrepo1 和 myrepo 两个仓库，一个 myproject 项目；

[root@slave712 Deploy]# docker push 172.30.16.122:80/myproject/myrepo
The push refers to a repository [172.30.16.122:80/myproject/myrepo]
69ecf026ff94: Pushed 
d7953e5e5bba: Pushed 
2f71b45e4e25: Pushed 
latest: digest: sha256:e7b139aeafa20bd1299559b4372b44a3b58505688e0cd1455a2d3b8bb6782d65 size: 948
```

- 注意

`Harbor` 启动会新建一个属于自己的桥接网卡，名为 `br-b2b386749c24`

- 在 `push image` 的时候如果出现 `“unauthorized: authentication required”`
   
  * 确认配置正确， 
  可以 独立 `run` 一个 `registry` 的容器，实验一下配置是否正确
  * 使用 `docker login` 命令先登录
  ```shell
  docker login -u admin -p Harbor12345 -e sample_admin@mydomain.com  172.30.0.20:5000
  ```

  * 给镜像打标签的时候， 要多一层 `library`， 这个是缺省的， 否则要建立好了 `project`，才能推镜像
  ```shell
  docker tag busybox  172.30.0.20:5000/library/my
  ```
- 注意，`Harbor` 的默认端口是 `80`，你在 `tag image` 时，需要把 `80` 端口也加上，并且在 `push image` 时，也需要把端口加上以及登录的时候
例如：
```shell
docker login 172.30.16.121:80
docker tag rsyslog:latest 172.30.16.121:80/library/rsyslog
docker push 172.30.16.121:80/library/rsyslog
...
```
还有，默认的 `80` 端口 `pull image` 后，在 `Harbor` 界面中不会显示出 `80` 端口，但是你在 `push` 的时候一定要加上 `80` 端口，否则会出现 `“image not found”` 的异常。

![](/picture/harbor1.bmp)
![](/picture/harbor2.bmp)
![](/picture/harbor3.bmp)
![](/picture/harbor4.bmp)
