# 环境准备

## 实验说明
在本实验中，除了特殊说明外，均使用Ubuntu 22.04作为OS。

在初始阶段中，master机器将会使用`2C8G`, worker机器用`2C8G`.

除非特殊说明，CPU架构均为`aarch64`.

由于实验环境在国外，所以大部分情况下不会因为特殊的网络问题带来镜像无法访问的现象。在此也推荐大家刚开始学习的时候用国外的服务器，减少麻烦。

## 容器运行时

和Docker一样，K8s也需要一个容器运行时。从1.24开始，K8s不再原生地提供对Docker Engine的支持。如果要使用Docker的话，需要手动安装Docker提供的`cri-dockerd`.

我们直接用containerd。根据官网指示：

> The containerd.io packages in DEB and RPM formats are distributed by Docker.

因此我们直接安装docker。

```bash
sudo apt-get update -y

sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
    
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update -y
sudo apt-get install -y containerd.io
```

安装成功后，下面的命令能够正常工作：
``` bash
sudo ctr containers ls
```

重新生成配置文件
```bash
sudo su
containerd config default > /etc/containerd/config.toml
## ** 然后在 config.toml 里面搜索 SystemCGroup ，将其改为true。
## ...
## ** 重启containerd
systemctl restart containerd
```

> 后续命令不再使用sudo前缀。

## 前置配置
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 应用 sysctl 参数而不重新启动
sudo sysctl --system
```

## 禁用分区

使用K8s需要禁用分区。临时禁用只需要使用`swapoff -a`命令。可以使用`free -m`看swap分区使用情况。

如果想要永久禁用分区，就编辑`/etc/fstab`文件，将关于`swap.img`的行注释掉。

禁用分区一定要做，不然后续kubelet无法启动成功。

## 安装kubeadm、kublect、kubelet

`kubeadm`是官方推荐的，用于安装/卸载K8s集群的工具。使用下面的方法安装它。

```bash
sudo apt-get update -y
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
# 这一步将锁定版本，禁止自动更新... 逆命令是`unhold`
sudo apt-mark hold kubelet kubeadm kubectl
```

安装完成后，`kubelet`会不断重启，因为它需要等待来自`kubeadm`的指令。此时使用类似`kubectl get pods`指令会报连接端口失败的错误（因为还没成功启动）。

接下来我们用`kubeadm`创建集群。

```bash
# 这句是核心, 逆命令是kubeadm reset。因为我们用的是flannel作为cni，因此一定要指定10.244.0.0/16.
kubeadm init --pod-network-cidr=10.244.0.0/16
```

> note: 这里的错误可能是千奇百怪的。比如说我这里用containerd作运行时的话，就需要重启containerd。善用搜索引擎。
>
> 如果出了一些奇怪的错，还搜不到的话，就reset重试几次。

等待一段时间后，会有以下的成功提示：
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.31.37.52:6443 --token 27r6xo.4wpm8a6210y8gubk \
        --discovery-token-ca-cert-hash sha256:ad70ebb0b32ac74481d2230ba11805df56127410fd7554688843e3b888814d4f
```

跟着他的指引，root和普通用户都需要跑一些命令。

然后`kubectl get pods --all-namespaces -o wide`命令尝试一下，应该还是报错。**这里一定要等两三分钟再试，前期初始化时容易报错**。

直到出现类似于下面的内容：
```bash
ubuntu@worker1:~$ kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                              READY   STATUS    RESTARTS       AGE   IP              NODE      NOMINATED NODE   READINESS GATES
kube-system   coredns-5dd5756b68-chb65          0/1     Pending   0              61s   <none>          <none>    <none>           <none>
kube-system   coredns-5dd5756b68-fq7g7          0/1     Pending   0              61s   <none>          <none>    <none>           <none>
kube-system   etcd-worker1                      1/1     Running   1 (2m9s ago)   94s   172.31.40.162   worker1   <none>           <none>
kube-system   kube-apiserver-worker1            1/1     Running   1 (99s ago)    94s   172.31.40.162   worker1   <none>           <none>
kube-system   kube-controller-manager-worker1   1/1     Running   1 (2m9s ago)   94s   172.31.40.162   worker1   <none>           <none>
kube-system   kube-proxy-xldlq                  1/1     Running   1 (58s ago)    61s   172.31.40.162   worker1   <none>           <none>
kube-system   kube-scheduler-worker1            1/1     Running   1 (2m9s ago)   94s   172.31.40.162   worker1   <none>           <none>
```

需要确认pods稳定下来了，也就是`coredns`处于pending状态，其他pod处于running（而不能出错）。出错应该是前面某些步骤没有做而导致的。

## 安装cni

上面的提示也说了，还要:

> deploy a pod network to the cluster

估计是因为还需要一个网络层来处理通信。

> 你必须部署一个基于 Pod 网络插件的 容器网络接口 (CNI)，以便你的 Pod 可以相互通信。 在安装网络之前，集群 DNS (CoreDNS) 将不会启动。

一般我们选择使用flannel就行了（简单易用），复杂一点就使用Calico。

```bash
## 需要确保文件里面用的是之前创建集群里面的10.244.0.0/16. 不放心的可以下载下来修改，再做apply。
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

等一段时间，使用`kubectl get pods --all-namespaces -o wide`可以看到所有pod都应该处于Running状态。

```bash
NAMESPACE      NAME                              READY   STATUS    RESTARTS   AGE     IP              NODE      NOMINATED NODE   READINESS GATES
kube-flannel   kube-flannel-ds-sgpgx             1/1     Running   0          22s     172.31.40.162   worker1   <none>           <none>
kube-system    coredns-5dd5756b68-f4755          1/1     Running   0          3m51s   10.244.0.3      worker1   <none>           <none>
kube-system    coredns-5dd5756b68-gm5ps          1/1     Running   0          3m51s   10.244.0.2      worker1   <none>           <none>
kube-system    etcd-worker1                      1/1     Running   5          4m4s    172.31.40.162   worker1   <none>           <none>
kube-system    kube-apiserver-worker1            1/1     Running   2          4m4s    172.31.40.162   worker1   <none>           <none>
kube-system    kube-controller-manager-worker1   1/1     Running   5          4m4s    172.31.40.162   worker1   <none>           <none>
kube-system    kube-proxy-l8c5w                  1/1     Running   0          3m51s   172.31.40.162   worker1   <none>           <none>
kube-system    kube-scheduler-worker1            1/1     Running   6          4m4s    172.31.40.162   worker1   <none>           <none>
```

如果需要直接通过containerd查看容器，使用
```bash
ctr -n k8s.io containers list

## 查看所有的命名空间
ctr ns ls
```

## 安装终端提示命令

```bash
source <(kubectl completion bash) # setup autocomplete in bash into the current shell, bash-completion package should be installed first.
echo "source <(kubectl completion bash)" >> ~/.bashrc # add autocomplete permanently to your bash shell.
```

## 新节点加入

可以随时在master的机器上，使用`kubeadm token create --print-join-command`查看join命令。

启动一台新节点，确保新节点和已有的节点能够互相访问。然后，安装`containerd`,`kubectl`,`kubelet`,`kubeadm`，关闭分区交换等。（也就是把上面的流程再走一遍，除了最后的`kubeadm init`）

在新节点上输入join命令即可。

可以通过`kubectl get nodes`查看新节点，status为ready即代表成功.
