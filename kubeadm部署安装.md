####  kubeadm部署安装

##### kube-proxy开启ipvs的前置条件

```
modprobe br_netfilter
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
# modprobe -- nf_conntrack_ipv4
EOF
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

安装docker

```


## 创建 /etc/docker目录
mkdir /etc/docker
# 配置docker
cat > /etc/docker/daemon.json << EOF
{
	"exec-opts":["native.cgroupdriver-systemd"],
	"log-driver":"json-file",
	"log-opts":{
	"max-size":"100m"
	}
}
EOF
mkdir -p /etc/systemd/system/docker.service.d
#重启docker服务
systemctl daemon-reload && systemctl restart docker && systemctl enable docker
```

安装kubeadm (主从配置)

```
#设置仓库
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
#安装
yum -y install kubeadm kubectl kubelet
systemctl enable kubelet.service
```

获取镜像列表

```
$ kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.18.3
k8s.gcr.io/kube-controller-manager:v1.18.3
k8s.gcr.io/kube-scheduler:v1.18.3
k8s.gcr.io/kube-proxy:v1.18.3
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.3-0
k8s.gcr.io/coredns:1.6.7
```



拉去镜像脚本

```
#cat pull-kube-image.sh
#!/bin/bash
images=(
    kube-apiserver:v1.18.3
    kube-controller-manager:v1.18.3
    kube-scheduler:v1.18.3
    kube-proxy:v1.18.3
    pause:3.2
    etcd:3.4.3-0
    coredns:1.6.7
)
for imageName in ${images[@]};
do
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/${imageName}
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/${imageName} k8s.gcr.io/${imageName}
    docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/${imageName}
done

```

查看下载的镜像

```
docker images
```

镜像打包

```
cat > save-kube-image.sh <<EOF
#!/bin/bash
images=(
    kube-apiserver:v1.18.3
    kube-controller-manager:v1.18.3
    kube-scheduler:v1.18.3
    kube-proxy:v1.18.3
    pause:3.2
    etcd:3.4.3-0
    coredns:1.6.7
)
for imageName in ${images[@]};
do
    docker save -o `echo ${imageName}|awk -F ‘:‘ ‘{print $1}‘`.tar k8s.gcr.io/${imageName}
done
EOF
```

解压缩

```
tar czvf kubeadm-images-1.18.0.tar.gz *.tar
scp kubeadm-images-1.18.0.tar.gz root@k8s-node01:~
mkdir kubeadm-images && tar zvxf kubeadm-images-1.18.0.tar.gz -C kubeadm-images
```

加载镜像

```
# cat load-image.sh 
#!/bin/bash
ls kubeadm-images > images-list.txt
cd kubeadm-images
for i in $(cat images-list.txt)
do
     docker load -i $i
done
```

初始化主节点

```
kubeadm init --kubernetes-version=1.18.3  \
--apiserver-advertise-address=192.168.92.10   \
--service-cidr=10.10.0.0/16 --pod-network-cidr=10.122.0.0/16
```

node节点加入

```
kubeadm join 192.168.92.10:6443 --token rnox1j.11dzsun7neiij0wy \
    --discovery-token-ca-cert-hash sha256:f55deafb52b4230471ae51217b77a2830444203edd5216a8779b8f1adad836ff
```

创建kubectl

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
source <(kubectl completion bash)
```

###  安装calico网络

```
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
 kubectl get pod --all-namespaces
```



https://www.kubernetes.org.cn/7189.html

### 安装kubernetes-dashboard

```
wget  https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc7/aio/deploy/recommended.yaml

vim recommended.yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30000
  selector:
    k8s-app: kubernetes-dashboard
    
    
$ kubectl create -f recommended.yaml    



kubectl get secret -n kube-dashboard

kubectl describe secrets -n kubernetes-dashboard  kubernetes-dashboard-token-bdrsd  | grep token | awk 'NR==3{print $2}'


eyJhbGciOiJSUzI1NiIsImtpZCI6IjRhZERLQ2VxLTU1TXpGV2lPQ3Jfd2hyWHpZd1k2TnlwR0FNUmpnQjZyNVUifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC10b2tlbi1iZHJzZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjcyYWY1ZTkxLWMzMDEtNDc4Ny1hYjdiLTE4YTE2NWFhNGM2OCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDprdWJlcm5ldGVzLWRhc2hib2FyZCJ9.OiO3n-VQXK_HEPaJVeGi2Uibe7HhiB0zDyYxfgR54XL_PNsX-PJh63BgcU9_XdEGHSbY81B-C7H66YrWIlvf8vNtYYMHoCJ8bCgfznS0d6eBgqvdrBN_Z8lzehIrj8SszHycPyeQo6eJvNYtFs41uPVn4J6m8Ujl2iN47gYvNxWfpn5EfqkpzKIM6jMMGUIUa19aebawFQ9DHGSn4UyfWeoQD56r3Nt3QUfK0OQH0-xspOUnkPNza5A-KWpCrS98wHG51Z8351VIGiLFMrWyeKolQiidLZTK-zI47XOVmgJSu-NRdNbFFfZpMpXwZf_hSW9TenhqYiI1XGnY672KYw

kubectl create clusterrolebinding serviceaccount-cluster-admin --clusterrole=cluster-admin --group=system:serviceaccount 

kubectl create clusterrolebinding serviceaccount-cluster-admin –clusterrole=cluster--admin  -–user=system:serviceaccount:kubernetes-dashboard:kubernetes-dashboard
```

