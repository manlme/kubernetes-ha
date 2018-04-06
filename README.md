# kubernetes-ha

使用kubeadm安装kubernetes高可用方案

## 架构选择

- centos 7.4
- 前端使用haproxy（f5，nginx），keepalive提供VIP和ha以及failover。
- etcd的高可用
- kubernetes master的高可用
- kubedns
- 网络使用calico


## 版本信息

- kubernetes 1.9.0
- etcd
- kubeadm 1.9.0
- kubernetes-cni 0.6.0 
- dashboard 1.8.3
- calico 3.0.4
- docker-ce <=17.03

## 步骤

### 准备工作(master和node）

* 准备docker私有仓库
* 准备rpms包
  * docker-ce-17.03.0.ce-1.el7.centos.x86_64.rpm
  * docker-ce-selinux-17.03.0.ce-1.el7.centos.noarch.rpm
  * kubeadm-1.9.0-0.x86_64.rpm
  * kubectl-1.9.0-0.x86_64.rpm
  * kubelet-1.9.0-0.x86_64.rpm
  * kubernetes-cni-0.6.0-0.x86_64.rpm
  
* 环境准备
```
# disable selinux
setenforce 0
# config kernel
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system

#disable swap
swapoff  -a
sed  -i /swap/d /etc/fstab
```
*  安装rpms包
```
yum  install -y *.rpm
systemctl enable {docker,kubelet}
systemctl start {docker,kubelet}
```
* 配置docker私有仓库(daemon.json和下面的10-kubeadm.conf文件可以配置一份保存然后分发到每个机器位置)
编辑/etc/docker/daemon.json， 将私有仓库添加进去. 参考如下
```
{
 "insecure-registries":["10.211.55.2:5000"]
}
```
* 配置kubelet的dropin文件/etc/systemd/system/kubelet.service.d/10-kubeadm.conf。 
将KUBELET_KUBECONFIG_ARGS指向能访问的docker仓库。将--cgroup-driver的配置与docker的保持抑制。  
docker的cgroup信息可以通过如下命令查看

`docker info`

10-kubeadm.conf参考文件如下

```
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--pod-infra-container-image=10.211.55.2:5000/pause-amd64:3.0 --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true"
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_DNS_ARGS=--cluster-dns=10.96.0.10 --cluster-domain=cluster.local"
Environment="KUBELET_AUTHZ_ARGS=--authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/ca.crt"
Environment="KUBELET_CADVISOR_ARGS=--cadvisor-port=0"
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"
Environment="KUBELET_CERTIFICATE_ARGS=--rotate-certificates=true --cert-dir=/var/lib/kubelet/pki"
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS $KUBELET_CADVISOR_ARGS $KUBELET_CGROUP_ARGS $KUBELET_CERTIFICATE_ARGS $KUBELET_EXTRA_ARGS
```
* 重启docker和kubelet使之生效
```
systemctl daemon-reload
systemctl restart {docker,kubelet}
```

* 在master1上的配置（需要一点技巧）
虽然前面已经将docker和kubelet都加入到了systemd管理，但是kubelet没有启动。    
准备kubeadm的配置文件config.yaml.(yaml文件，缩进很重要) 参考官方解读。
```
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
api:
  advertiseAddress: 10.37.129.9
etcd:
  endpoints:
  - http://10.37.129.7:2379
  - http://10.37.129.8:2379
  - http://10.37.129.9:2379
networking:
  podSubnet: 192.168.0.0/16
# apiServerCertSANs:
# - <load-balancer-ip>
imageRepository: 10.211.55.2:5000/google_containers
kubernetesVersion: v1.9.0
apiServerExtraArgs:
  apiserver-count: "3"
```
 
启动kubelet
```
# generate certs
kubeadm alpha phase certs all --config config.yaml
# generate config for kubelet
kubeadm alpha phase kubeconfig kubelet --config config.yaml
```
至此kubelet启动。
* 其他master的配置
将config.yaml拷贝到其他master上，修改advertiseAddress相应配置  
将/etc/kubernetes/pki拷贝到其他master的/etc/kubernetes/pki目录下。  
使用对应的config.yaml启动kubelet
`kubeadm alpha phase kubeconfig kubelet --config config.yaml`
至此，所有master的kubelet都已经启动。 用如下命令查看
`ps -ef | grep kubelet`
### kubernets etcd高可用
这次使用的etcd是非安全的http。  
将manifest目录下的etcd.yaml文件相应的修改放到每个master的/etc/kubernetes/manifest目录下。etcd作为staci pod，由kubelet自动启动。
### kubernets master的高可用
在每台master上运行初始化命令  
```
kubeadm init --config config.yaml  --ignore-preflight-errors=all
```
输出结果类似如下，
```
[init] Using Kubernetes version: v1.8.0
[init] Using Authorization modes: [Node RBAC]
[preflight] Running pre-flight checks
[kubeadm] WARNING: starting in 1.8, tokens expire after 24 hours by default (if you require a non-expiring token use --token-ttl 0)
[certificates] Generated ca certificate and key.
[certificates] Generated apiserver certificate and key.
[certificates] apiserver serving cert is signed for DNS names [kubeadm-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.138.0.4]
[certificates] Generated apiserver-kubelet-client certificate and key.
[certificates] Generated sa key and public key.
[certificates] Generated front-proxy-ca certificate and key.
[certificates] Generated front-proxy-client certificate and key.
[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "scheduler.conf"
[controlplane] Wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
[controlplane] Wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
[controlplane] Wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
[etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
[init] Waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests"
[init] This often takes around a minute; or longer if the control plane images have to be pulled.
[apiclient] All control plane components are healthy after 39.511972 seconds
[uploadconfig] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[markmaster] Will mark node master as master by adding a label and a taint
[markmaster] Master master tainted and labelled with key/value: node-role.kubernetes.io/master=""
[bootstraptoken] Using token: <token>
[bootstraptoken] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: kube-dns
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run (as a regular user):

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  http://kubernetes.io/docs/admin/addons/

You can now join any number of machines by running the following on each node
as root:
```

等待初始化完成，查看状态
```
kubectl get nodes
```
得到如下状态。 看到notready不要着急，因为网络还没初始化。
```
[vagrant@master1 ~]$ kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
master1   NotReady     master    1d      v1.9.0
master2   NotReady     master    1d        v1.9.0
master3   NotReady     master    1d        v1.9.0
```

### 安装网络calico
将calico.yaml里的镜像配置成集群能访问的私有群地址，将etcd_endpoints配置成单个etcd节点（这是etcd和calico的bug，配置集群，calico会不稳定删除pod的路由）
任何一台master上，运行如下命令
```
kubectl apply -f calico.yaml
```

等待网络初始化完成

### 加入节点
```
kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
```
### haproxy和keepalive配置



 

