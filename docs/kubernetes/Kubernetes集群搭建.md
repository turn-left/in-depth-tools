##  Kubernetes集群搭建

### 虚拟机准备

- virtualbox

- centos7

  | 节点        | ip            |
  | ----------- | ------------- |
  | kube-master | 192.168.56.20 |
  | kube-node-1 | 192.168.56.21 |
  | kube-node-2 | 192.168.56.22 |

### 基本环境配置

1. 关闭防火墙

   ```
   $ systemctl stop firewalld
   $ systemctl disable firewalld
   ```

2. 关闭selinux

   ```
   setenforce 0
   sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
   ```

3. 关闭swap分区或禁用swap文件

   ```
   swapoff -a
   # 注释掉关于swap分区的行
   yes | cp /etc/fstab /etc/fstab_bak
   cat /etc/fstab_bak |grep -v swap > /etc/fstab
   ```

4. 配置主机域名

   ```bash
   # 分别在三个节点执行 在master所在虚拟机节点 <hostname>替换为kube-master
   hostnamectl set-hostname <hostname>
   ```

5. 配置hosts

   ```bash
   cat >> /etc/hosts << EOF
   192.168.65.160 k8s-master
   192.168.65.203 k8s-node1
   192.168.65.210 k8s-node2
   EOF
   ```

6. 将桥接的IPv4流量传递到iptables

   ```bash
   cat > /etc/sysctl.d/k8s.conf << EOF
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   EOF

   # 配置生效
   sysctl --system
   ```

7. 设置时间同步

   ```bash
   yum install ntpdate -y
   ntpdate time.windows.com
   ```

8. 添加k8s阿里yum源

   ```bash
   cat > /etc/yum.repos.d/kubernetes.repo << EOF
   [kubernetes]
   name=Kubernetes
   baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
   enabled=1
   gpgcheck=0
   repo_gpgcheck=0
   gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
   https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
   EOF
   ```

   或者采用vim编辑文件

   ```bash
   vim /etc/yum.repos.d/kubernetes.repo
   [kubernetes]
   name=Kubernetes
   baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
   enabled=1
   gpgcheck=0
   repo_gpgcheck=0
   gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
   	http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
   ```

9. 之前装过k8s先卸载

   ```bash
   yum remove -y kubelet kubeadm kubectl
   ```

10. 查看可以安装的版本

   ```bash
   yum list kubelet --showduplicates | sort -r
   ```

   ​

11. 安装kubelet、kubeadm、kubectl 指定版本

    ```bash
    yum install -y kubelet-1.18.0 kubeadm-1.18.0 kubectl-1.18.0
    ```

12. 设置kubelet开机自启

    ```bash
    systemctl enable kubelet
    systemctl start kubelet
    ```





------

### Docker安装和配置

#### Docker安装

docker的安装请查看官网文档(Overview of Docker editions)[<https://docs.docker.com/install/overview/]>

#### Docker配置

1. 配置cgroup-driver为systemd

   ```
   # 查看cgroup-driver
   $ docker info | grep -i cgroup
   # 追加 --exec-opt native.cgroupdriver=systemd 参数
   $ sed -i "s#^ExecStart=/usr/bin/dockerd.*#ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd#g" /usr/lib/systemd/system/docker.service
   $ systemctl daemon-reload # 重新加载服务
   $ systemctl enable docker # 启用docker服务(开机自起)
   $ systemctl restart docker # 启动docker服务
   # 或者修改docker配置文件
   $ vim /etc/docker/daemon.json
   {
     "exec-opts": ["native.cgroupdriver=systemd"]
   }
   ```

2. 预先拉取所需镜像

   ```
   # 查看kubeadm所需镜像
   $ kubeadm config images list
   k8s.gcr.io/kube-apiserver:v1.16.3
   k8s.gcr.io/kube-controller-manager:v1.16.3
   k8s.gcr.io/kube-scheduler:v1.16.3
   k8s.gcr.io/kube-proxy:v1.16.3
   k8s.gcr.io/pause:3.1
   k8s.gcr.io/etcd:3.3.15-0
   k8s.gcr.io/coredns:1.6.2
   # 拉取镜像
   $ docker pull kubeimage/kube-apiserver-amd64:v1.16.3
   $ docker pull kubeimage/kube-controller-manager-amd64:v1.16.3
   $ docker pull kubeimage/kube-scheduler-amd64:v1.16.3
   $ docker pull kubeimage/kube-proxy-amd64:v1.16.3
   $ docker pull kubeimage/pause-amd64:3.1
   $ docker pull kubeimage/etcd-amd64:3.3.15-0
   $ docker pull coredns/coredns:1.6.2
   ```

3. 对预先拉取的镜像重新打tag

   ```
   $ docker tag kubeimage/kube-apiserver-amd64:v1.16.3  k8s.gcr.io/kube-apiserver:v1.16.3
   $ docker tag kubeimage/kube-controller-manager-amd64:v1.16.3  k8s.gcr.io/kube-controller-manager:v1.16.3
   $ docker tag kubeimage/kube-scheduler-amd64:v1.16.3  k8s.gcr.io/kube-scheduler:v1.16.3
   $ docker tag kubeimage/kube-proxy-amd64:v1.16.3 k8s.gcr.io/kube-proxy:v1.16.3
   $ docker tag kubeimage/pause-amd64:3.1 k8s.gcr.io/pause:3.1
   $ docker tag kubeimage/etcd-amd64:3.3.15-0 k8s.gcr.io/etcd:3.3.15-0
   $ docker tag coredns/coredns:1.6.2 k8s.gcr.io/coredns:1.6.2
   ```

### Master节点的配置

**以上步骤需要在node节点和master节点执行，当前步骤仅需在master节点执行。**

#### Master节点的初始化

```bash
# 初始化master节点，
# --pod-network-cidr=10.244.0.0/16 指定使用flunnel网络
# --apiserver-advertise-address=192.168.56.20 指向master节点IP
$ kubeadm init --apiserver-advertise-address=192.168.56.20 --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.18.0 --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16
```

执行上述命令的输出为：

```
[init] Using Kubernetes version: v1.18.0
[preflight] Running pre-flight checks
        [WARNING SystemVerification]: this Docker version is not on the list of validated versions: 20.10.15. Latest validated version: 19.03
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kube-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.56.20]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [kube-master localhost] and IPs [192.168.56.20 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [kube-master localhost] and IPs [192.168.56.20 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
W0509 00:04:16.849472   26369 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-scheduler"
W0509 00:04:16.850146   26369 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 15.512387 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.18" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node kube-master as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node kube-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: bylqrh.eu7qnb6q789c3kd7
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.56.20:6443 --token bylqrh.eu7qnb6q789c3kd7 \
    --discovery-token-ca-cert-hash sha256:8c8ecf703ef60d92f32a590b6564c68defb9a2cceaba1b0e2e2f506510292bbd
```

保存输出中的`kubeadm join`部分内容，用于添加node节点，或者使用`kubeadm token list` 和`kubeadm token create --print-join-command`查看

```bash
kubeadm join 192.168.56.20:6443 --token bylqrh.eu7qnb6q789c3kd7 \
    --discovery-token-ca-cert-hash sha256:8c8ecf703ef60d92f32a590b6564c68defb9a2cceaba1b0e2e2f506510292bbd
```

接下来执行剩余的初始化步骤

```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### Calico网络插件的配置

Calico的官方文档地址为： [https://docs.projectcalico.org/v3.10/getting-started/kubernetes/。](https://docs.projectcalico.org/v3.10/getting-started/kubernetes/%E3%80%82) 具体安装步骤：

1. 安装Calico

   ```
   $ kubectl apply -f https://docs.projectcalico.org/v3.10/manifests/calico.yaml
   ```

2. 监听安装进度

   ```
   $ watch kubectl get pods --all-namespaces
   ```

出现以下内容时为安装成功

```
NAMESPACE    NAME                                       READY  STATUS   RESTARTS  AGE
kube-system  calico-kube-controllers-6ff88bf6d4-tgtzb   1/1    Running  0         2m45s
kube-system  calico-node-24h85                          1/1    Running  0         2m43s
kube-system  coredns-846jhw23g9-9af73                   1/1    Running  0         4m5s
kube-system  coredns-846jhw23g9-hmswk                   1/1    Running  0         4m5s
kube-system  etcd-jbaker-1                              1/1    Running  0         6m22s
kube-system  kube-apiserver-jbaker-1                    1/1    Running  0         6m12s
kube-system  kube-controller-manager-jbaker-1           1/1    Running  0         6m16s
kube-system  kube-proxy-8fzp2                           1/1    Running  0         5m16s
kube-system  kube-scheduler-jbaker-1                    1/1    Running  0         5m41s
```

1. 测试

   ```
   $ kubectl get nodes -o wide
   NAME                STATUS     ROLES    AGE     VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
   kubernetes-master   Ready      master   4d12h   v1.16.3   192.168.56.101   <none>        CentOS Linux 7 (Core)   3.10.0-1062.el7.x86_64   docker://19.3.4
   ```

### Node节点的初始化

1. 登录node节点，执行加入集群的命令，完成加入集群操作

   ```
   $ kubeadm join 10.0.0.5:6443 --token kt58np.djd3youoqb0bnz4r \
       --discovery-token-ca-cert-hash sha256:37a3924142dc6d57eac2714e539c174ee3b0cda723746ada2464ac9e8a2091ce
   ```

2. 在master节点上查看添加结果

   ```
   $ kubectl get nodes -o wide
   NAME                STATUS     ROLES    AGE     VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
   kubernetes-master   Ready      master   4d12h   v1.16.3   192.168.56.101   <none>        CentOS Linux 7 (Core)   3.10.0-1062.el7.x86_64   docker://19.3.4
   kubernetes-node-1   Ready      <none>   4d12h   v1.16.3   192.168.56.102   <none>        CentOS Linux 7 (Core)   3.10.0-1062.el7.x86_64   docker://19.3.4
   ```



#### 参考

- [K8S安装过程笔记](http://blog.hungtcs.top/2019/11/27/23-K8S%E5%AE%89%E8%A3%85%E8%BF%87%E7%A8%8B%E7%AC%94%E8%AE%B0/)