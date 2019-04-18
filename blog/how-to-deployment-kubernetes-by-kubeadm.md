---
date: 2019-04-18
---

# 如何使用 Kubeadm 部署 Kubernetes

## 本指南特点

- 基于 Kubernetes 1.13.1
- 负载均衡使用 HAProxy 和 KeepAlived
- 支持 HA
- POD 网络使用 Flannel host-gw
- 使用 etcd stacked 布局
- 各个环节均使用国内镜像加速

## 准备

- 硬件：2G RAW, 2 CPUs
- 网络：私有网络连接所有节点
- OS: CentOS 7
- SWAP：关闭

| 节点 | IP           | 角色                   |
| :--- | :----------- | :--------------------- |
| -    | 172.17.8.100 | LB VIP                 |
| m1   | 172.17.8.101 | k8s master + etcd + LB |
| m2   | 172.17.8.102 | k8s master + etcd + LB |
| m3   | 172.17.8.103 | k8s master + etcd + LB |
| w1   | 172.17.8.111 | k8s worker             |
| w2   | 172.17.8.112 | k8s worker             |

## 安装 Runtime

> 使用 Docker CE 18.06

所有 Master + Worker 节点安装 Docker

```sh
## 添加 Docker 仓库
wget -O /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sed -i 's+download.docker.com+mirrors.aliyun.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo

## 安装 Docker
yum install -y docker-ce-18.06.1.ce

## 配置 Docker daemon
mkdir /etc/docker
cat > /etc/docker/daemon.json <<EOF
{
  "live-restore": true,
  "bridge": "none",
  "iptables": false,
  "registry-mirrors": ["https://yjgetdw3.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

# 使配置生效
systemctl enable docker.service && systemctl restart docker
```

## 安装部署 Kubernetes 基础组件

> 包含 kubeadm, kubelet 和 kubectl

在所有 Master + Worker 节点完成以下安装配置

```sh
## 配置 Kubernetes 仓库
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF

# 关闭 SELINUX
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config

## 安装组件
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

## 启动 kubelet
systemctl enable kubelet && systemctl start kubelet

## 如果此时查看 kubelet 状态报告 failed to load Kubelet config file /var/lib/kubelet/config.yaml,
## 这个是正常的，这个文件会在后 kubeadm init 生成
```

> 在 CentOS/RHEL 7 下存在路由不正确的问题，需要配置如下

```sh
cat << EOF > /etc/modules-load.d/br_netfilter.conf
br_netfilter
EOF
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

## 为 kube-apiserver 配置 LB

> 使用 haproxy + keepalived

在第一个 master 节点完成以下安装配置

```sh
## 安装 haproxy 和 keepalived
yum install -y haproxy keepalived

## 配置 haproxy
cat << EOF > /etc/haproxy/haproxy.cfg
global
    log /dev/log local0
    chroot /var/lib/haproxy
    user haproxy
    group haproxy

defaults
    log global
    timeout connect         10s
    timeout client          1m
    timeout server          1m

frontend k8s-api
  bind *:443
  mode tcp
  option tcplog
  default_backend k8s-api

backend k8s-api
  mode tcp
  option tcplog
  option tcp-check
  balance roundrobin
  default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
  server k8s-api-1 172.17.8.101:6443 check
  server k8s-api-2 172.17.8.102:6443 check
  server k8s-api-3 172.17.8.103:6443 check
EOF

## 配置 keepalived
yum install -y psmisc
cat << EOF > /etc/keepalived/keepalived.conf
vrrp_script haproxy-check {
    script "killall -0 haproxy"
    interval 2
    weight 20
}

vrrp_instance haproxy-vip {
    state BACKUP
    priority 101
    interface eth1
    virtual_router_id 100
    advert_int 3

    virtual_ipaddress {
	172.17.8.100
    }

    track_script {
        haproxy-check weight 20
    }
}
EOF

## 启动服务
systemctl start haproxy && systemctl enable haproxy
systemctl start keepalived && systemctl enable keepalived
```

剩余 master 节点安装 haproxy + keepalived 按照上述方法配置，其中只需要修改 keepalived 中的 `priority` 优先级（值越大优先级越高）

## 创建初始 Master

```sh
## 生成 kubeadm 配置
cat << EOF > kubeadm.yml
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: v1.13.1
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
apiServer:
  certSANs:
  - "172.17.8.100"
controlPlaneEndpoint: "172.17.8.100:443"
networking:
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
---
apiVersion: kubeadm.k8s.io/v1beta1
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 172.17.8.101
EOF

## 初始化第一个 Master
kubeadm init --config kubeadm.yml

## 按照上一步输出设置 kubectl 配置
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

## 记录下 kubeadm init 输出最后一部分用于之后其它节点加入集群，例如：
cat << EOF > kubeadm-join.record
kubeadm join 172.17.8.100:443 --token xcja08.ix7lv4e2yve5y2xx --discovery-token-ca-cert-hash sha256:d16fc0f3ab2f0720be9516bd79537d5119110bc894332b6a5e070d59e5e5ea80
EOF
```

## 安装 pod 网络

```sh
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
sed -i 's/vxlan/host-gw/' kube-flannel.yml
kubectl apply -f kube-flannel.yml

## 如果内网连接的网口不是第一个，则需要在 command 的 args 中通过 `--iface` 指定网口, 例如:
command:
- /opt/bin/flanneld
args:
- --ip-masq
- --kube-subnet-mgr
- --iface
- eth1
```

## 加入剩余 Master 节点

```sh
## 从第一个 Master 拷贝证书文件到其它 Master 节点
cat << "EOF" > copy-cert.sh
masters="172.17.8.102 172.17.8.103"
for i in $masters; do
    ssh $i "mkdir -p /etc/kubernetes/pki/etcd"
    scp /etc/kubernetes/pki/ca.*  $i:/etc/kubernetes/pki
    scp /etc/kubernetes/pki/sa.*  $i:/etc/kubernetes/pki
    scp /etc/kubernetes/pki/front-proxy-ca.*  $i:/etc/kubernetes/pki
    ssh $i "mkdir -p /etc/kubernetes/pki/etcd"
    scp /etc/kubernetes/pki/etcd/ca.* $i:/etc/kubernetes/pki/etcd
    scp /etc/kubernetes/admin.conf $i:/etc/kubernetes/pki
done
EOF
bash copy-cert.sh

## 依次在剩余 Master 节点执行 kubeadm-join.record 记录命令，额外加上参数 --experimental-control-plane 和 --apiserver-advertise-address。例如：
kubeadm join 172.17.8.100:443 --token xcja08.ix7lv4e2yve5y2xx --discovery-token-ca-cert-hash sha256:d16fc0f3ab2f0720be9516bd79537d5119110bc894332b6a5e070d59e5e5ea80 --experimental-control-plane --apiserver-advertise-address 172.17.8.102

## 查看节点状态
kubectl get nodes
```

## 加入 Worker 节点

```sh
## 加入 Worker 节点到集群中
kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>

## 例如，直接在 Worker 节点执行 kubeadm init 输出命令
kubeadm join 172.17.8.100:443 --token xcja08.ix7lv4e2yve5y2xx --discovery-token-ca-cert-hash sha256:d16fc0f3ab2f0720be9516bd79537d5119110bc894332b6a5e070d59e5e5ea80

## 如果 Token 过期，可以重新生成
kubeadm token create
kubeadm token list
## 获取 --discovery-token-ca-cert-hash 的值
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
```

## 重置集群

> 移除 Master 节点同下

```sh
## 移除 Worker 节点
kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
kubectl delete node <node name>

## 在 Worker 上执行重置
kubeadm reset
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

## 参考

- https://kubernetes.io/docs/setup/independent/install-kubeadm/
- https://prefetch.net/blog/2018/02/20/getting-the-flannel-host-gw-working-with-kubernetes/
