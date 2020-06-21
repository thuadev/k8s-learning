#### centos 在线安装docker

##### 1.确认内核版本高于3.10

```
$ uname -r
$ rpm -q centos-release
```

##### 2.更新yum包

```
$ sudo yum update
```

##### 3.卸载旧版本(如果安装过旧版本的话)

```
$ sudo yum remove docker  docker-common docker-selinux docker-engine
```

##### 4.安装需要的软件包， yum-util 提供yum-config-manager功能，另外两个是devicemapper驱动依赖的

```
$ sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

##### 5.设置yum源

~~~
$ sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
~~~

##### 6、可以查看所有仓库中所有docker版本，并选择特定版本安装

~~~
$ yum list docker-ce --showduplicates | sort -r
~~~

##### 7、安装docker

~~~
$ sudo yum install docker-ce  #由于repo中默认只开启stable仓库，故这里安装的是最新稳定版17.12.0
$ sudo yum install <FQPN>  # 例如：sudo yum install docker-ce-17.12.0.ce
~~~

##### 8、启动并加入开机启动

~~~
$ sudo systemctl start docker
$ sudo systemctl enable docker
~~~

##### 9、验证安装是否成功(有client和service两部分表示docker安装启动都成功了)

~~~
$ docker version
~~~

##### 启动失败问题

Job for docker.service canceled.

```
[root@gs-server-7560 huati@gridsum.com]# systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since 五 2020-06-05 00:17:54 CST; 5min ago
     Docs: https://docs.docker.com
  Process: 53607 ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock (code=exited, status=205/LIMITS)
 Main PID: 53607 (code=exited, status=205/LIMITS)

6月 05 00:17:54 gs-server-7560 systemd[1]: Starting Docker Application Container Engine...
6月 05 00:17:54 gs-server-7560 systemd[1]: docker.service: main process exited, code=exited, status=205/LIMITS
6月 05 00:17:54 gs-server-7560 systemd[1]: Stopped Docker Application Container Engine.
6月 05 00:17:54 gs-server-7560 systemd[1]: Unit docker.service entered failed state.
6月 05 00:17:54 gs-server-7560 systemd[1]: docker.service failed.
```

###### 解决办法

```
sysctl -w fs.nr_open=1048576
```



