#### k8s主机初始化

设置主机名

```
hostnamectl set-hostname k8s-master01
```

修改host域名解析

```
vi /etc/hosts
192.168.92.10 k8s-master01
192.168.92.20 k8s-node01
192.168.92.21 k8s-node02
#分发hosts
scp /etc/hosts root@k8s-node01:/etc/hosts
scp /etc/hosts root@k8s-node02:/etc/hosts
```

安装依赖包

```
yum install -y conntrack ntpdate ntp ipvsadm ipset jq iptables curl sysstat libseccomp wget vim net-tools git
```

设置防火墙为Iptables并设置空规则

```
systemctl stop firewalld && systemctl disable firewalld
yum install -y iptables-services && systemctl start iptables && systemctl enable iptables && iptables -F && service iptables save


iptables -P INPUT ACCEPT 
iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat
iptables -P FORWARD ACCEPT
iptables -P INPUT ACCEPT
service iptables save

```

关闭selinux

```
swapoff -a && sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
setenforce 0 && sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

调整内核参数，对于k8s

```
cat > kubernetes.conf << EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
vm.swapiness=0 #禁止使用swap空间,只有当系统oom时才允许使用
vm.overcommit_memory=1 #不检查物理内存是否够用
vm.panic_on_oom=0 #开启OOM
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF
cp kubernetes.conf /etc/sysctl.d/kubernetes.conf
sysctl -p /etc/sysctl.d/kubernetes.conf

```

关闭系统不需要的服务

```
systemctl stop postfix && systemctl disable postfix
```

设置 rsyslogd 和 systemd journald

```
mkdir /var/log/journal # 持久化保存日志的目录
mkdir /etc/systemd/journald.conf.d
cat > /etc/systemd/journald.conf.d/99-prophet.conf <<EOF
[Journal]
# 持久化保存到磁盘
Storage=persistent

# 压缩历史日志
Compress=yes

SyncIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000

# 最大占用空间 10G
SystemMaxUse=10G

# 单日志文件最大 200M
SystemMaxFileSize=200M

# 日志保存时间 2 周
MaxRetentionSec=2week

# 不将日志转发到 syslog
ForwardToSyslog=no
EOF
systemctl restart systemd-journald

```

升级系统内核为4.4

Centos 7.x 系统自带的3.10x内核存在一些bugs

```
export Kernel_Version=4.20.13-1
wget  http://mirror.rc.usf.edu/compute_lock/elrepo/kernel/el7/x86_64/RPMS/kernel-ml{,-devel}-${Kernel_Version}.el7.elrepo.x86_64.rpm
yum localinstall -y kernel-ml*
# 通过命令查看默认启动顺序
awk -F\' '$1=="menuentry " {print $2}' /etc/grub2.cfg
# 设置开机从新内核启动
grub2-set-default 0
```

重新生成 grub2 配置文件：

```
cp /boot/grub2/grub.cfg{,.bak}
grub2-mkconfig -o /boot/grub2/grub.cfg

#docker官方的内核检查脚本建议(RHEL7/CentOS7: User namespaces disabled; add 'user_namespace.enable=1' to boot command line)
grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"
# 重启系统
reboot
# 查看系统的内核版本
uname -r
```

## 加载内核模块

```
modprobe br_netfilter
```