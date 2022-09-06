> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [segmentfault.com](https://segmentfault.com/a/1190000040107263)

> 学习 Kubernetes 的关键一步就是要学会搭建一套 k8s 集群。在今天的文章中作者将最近新总结的搭建技巧，无偿分享给大家！废话不多说，直接上干货！

![](https://segmentfault.com/img/remote/1460000040107265)

学习 Kubernetes 的关键一步就是要学会搭建一套 k8s 集群。在今天的文章中作者将最近新总结的搭建技巧，无偿分享给大家！废话不多说，直接上干货！

### **01、系统环境准备**

要安装部署 Kubernetes 集群，首先需要准备机器，最直接的办法可以到公有云（如阿里云等）申请几台虚拟机。而如果条件允许，拿几台本地物理服务器来组建集群自然是最好不过了。但是这些机器需要满足以下几个条件：

*   要求 64 位 Linux 操作系统，且内核版本要求 3.10 及以上，能满足安装 Docker 项目所需的要求；
*   机器之间要保持网络互通，这是未来容器之间网络互通的前提条件；
*   要有外网访问权限，因为部署的过程中需要拉取相应的镜像，要求能够访问到 gcr.io、quay.io 这两个 docker registry, 因为有小部分镜像需要从这里拉取；
*   单机可用资源建议 2 核 CPU、8G 内存或以上，如果小一点也可以但是能调度的 Pod 数量就比较有限了；
*   磁盘空间要求在 30GB 以上，主要用于存储 Docker 镜像及相关日志文件；

在本次实验中我们准备了两台虚拟机，其具体配置如下：

*   2 核 CPU、2GB 内存，30GB 的磁盘空间；
*   Unbantu 20.04 LTS 的 Sever 版本，其 Linux 内核为 5.4.0；
*   内网互通，外网访问权限不受控制；

### **02、Kubernetes 集群部署工具 Kubeadm 介绍**

作为典型的分布式系统，Kubernetes 的部署一直是困扰初学者进入 Kubernetes 世界的一大障碍。在发布早期 Kubernetes 的部署主要依赖于社区维护的各种脚本，但这其中会涉及二进制编译、配置文件以及 kube-apiserver 授权配置文件等诸多运维工作。目前各大云服务厂商常用的 Kubernetes 部署方式是使用 SaltStack、Ansible 等运维工具自动化地执行这些繁琐的步骤，但即使这样，这个部署的过程对于初学者来说依然是非常繁琐的。

正是基于这样的痛点，在志愿者的推动下 Kubernetes 社区终于发起了 kubeadm 这一独立的一键部署工具，使用 kubeadm 我们可以通过几条简单的指令来快速地部署一个 kubernetes 集群。在接下来的内容中，就将具体演示如何使用 kubeadm 来部署一个简单结构的 Kubernetes 集群。

### **03、安装 kubeadm 及 Docker 环境**

正是基于这样的痛点，在志愿者的推动下 Kubernetes 社区终于发起了 kubeadm 这一独立的一键部署工具，使用 kubeadm 我们可以通过几条简单的指令来快速地部署一个 kubernetes 集群。在接下来的内容中，就将具体演示如何使用 kubeadm 来部署一个简单结构的 Kubernetes 集群。

前面简单介绍了 Kubernetes 官方发布一键部署工具 kubeadm，只需要添加 kubeadm 的源，然后直接用 yum 安装即可，具体操作如下：

1)、编辑操作系统安装源配置文件，添加 kubernetes 镜像源，命令如下：

```
#添加Docker阿里镜像源

[root@centos-linux ~]# wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo

#安装Docker

[root@centos-linux ~]# yum -y install docker-ce-18.09.9-3.el7

#启动Docker并设置开机启动

[root@centos-linux ~]# systemctl enable docker

添加Kubernetes yum镜像源，由于网络原因，也可以换成国内Ubantu镜像源，如阿里云镜像源地址：

添加阿里云Kubernetes yum镜像源

# cat > /etc/yum.repos.d/kubernetes.repo << EOF

[kubernetes]

name=Kubernetes

baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64

enabled=1

gpgcheck=0

repo_gpgcheck=0

gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg

EOF
```

  2)、完成上述步骤后就可以通过 yum 命令安装 kubeadm 了，如下：

```
[root@centos-linux ~]# yum install -y kubelet-1.20.0 kubeadm-1.20.0 kubectl-1.20.0

当前版本是最新版本1.21，这里安装1.20。

#查看安装的kubelet版本信息

[root@centos-linux ~]# kubectl version

Client Version: version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.0", GitCommit:"af46c47ce925f4c4ad5cc8d1fca46c7b77d13b38", GitTreeState:"clean", BuildDate:"2020-12-08T17:59:43Z", GoVersion:"go1.15.5", Compiler:"gc", Platform:"linux/amd64"}

The connection to the server localhost:8080 was refused - did you specify the right host or port?

在上述安装kubeadm的过程中，kubeadm和kubelet、kubectl、kubernetes-cni这几个kubernetes核心组件的二进制文件也都会被自动安装好。
```

3)、Docker 服务启动及限制修改

在具体运行 kubernetes 部署之前需要对 Docker 的配置信息进行一些调整。首先，编辑系统 / etc/default/grub 文件，在配置项 GRUB_CMDLINE_LINUX 中添加如下参数：

```
GRUB_CMDLINE_LINUX=" cgroup_enable=memory swapaccount=1"
```

完成编辑后保存执行如下命令，并重启服务器，命令如下：

```
root@kubernetesnode01:/opt/kubernetes-config# reboot
```

上述修改主要解决的是可能出现的 “docker 警告 WARNING: No swap limit support” 问题。其次，编辑创建 / etc/docker/daemon.json 文件，添加如下内容：

```
# cat > /etc/docker/daemon.json <<EOF

{

 "registry-mirrors": ["https://6ze43vnb.mirror.aliyuncs.com"],

 "exec-opts": ["native.cgroupdriver=systemd"],

 "log-driver": "json-file",

 "log-opts": {

   "max-size": "100m"

 },

 "storage-driver": "overlay2"

}

EOF

完成保存后执行重启Docker命令，如下：

# systemctl restart docker

此时可以查看Docker的Cgroup信息，如下：

# docker info | grep Cgroup

Cgroup Driver: systemd
```

上述修改主要解决的是 “Docker cgroup driver. The recommended driver is "systemd"” 的问题。需要强调的是以上修改只是作者在具体安装操作是遇到的具体问题的解决整理，如在实践过程中遇到其他问题还需要自行查阅相关资料！ 最后，需要注意由于 kubernetes 禁用虚拟内存，所以要先关闭掉 swap 否则就会在 kubeadm 初始化 kubernetes 的时候报错，具体如下：

```
# swapoff -a
```

该命令只是临时禁用 swap，如要保证系统重启后仍然生效则需要 “vim /etc/fstab” 文件，并注释掉 swap 那一行。

### **04、部署 Kubernetes 的 Master 节点**

在 Kubernetes 中 **Master 节点是集群的控制节点**，它是由三个紧密协作的独立组件组合而成，分别是**负责 API 服务的 kube-apiserver**、**负责调度的 kube-scheduler** 以及**负责容器编排的 kube-controller-manager**，其中整个集群的持久化数据由 kube-apiserver 处理后保存在 Etcd 中。 要部署 Master 节点可以直接通过 kubeadm 进行一键部署，但这里我们希望能够部署一个相对完整的 Kubernetes 集群，可以通过配置文件来开启一些实验性的功能。具体在系统中新建 / opt/kubernetes-config / 目录，并创建一个给 kubeadm 用的 YAML 文件（kubeadm.yaml），具体内容如下：

```
apiVersion: kubeadm.k8s.io/v1beta2

kind: ClusterConfiguration

controllerManager:

extraArgs:

    horizontal-pod-autoscaler-use-rest-clients: "true"

    horizontal-pod-autoscaler-sync-period: "10s"

    node-monitor-grace-period: "10s"

apiServer:

 extraArgs:

    runtime-config: "api/all=true"

kubernetesVersion: "v1.20.0"
```

在上述 yaml 配置文件中 “horizontal-pod-autoscaler-use-rest-clients: "true"” 这个配置，表示将来部署的 kuber-controller-manager 能够使用自定义资源（Custom Metrics）进行自动水平扩展，感兴趣的读者可以自行查阅相关资料！而 “v1.20.0” 就是要 kubeadm 帮我们部署的 Kubernetes 版本号。

需要注意的是，如果执行过程中由于国内网络限制问题导致无法下载相应的 Docker 镜像，可以根据报错信息在国内网站（如阿里云）上找到相关镜像，然后再将这些镜像重新 tag 之后再进行安装。具体如下：

```
#从阿里云Docker仓库拉取Kubernetes组件镜像

docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64:v1.20.0

docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64:v1.20.0

docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64:v1.20.0

docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64:v1.20.0

docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2

docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0

docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0
```

下载完成后再将这些 Docker 镜像重新 tag 下，具体命令如下：

```
#重新tag镜像

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64:v1.20.0 k8s.gcr.io/kube-scheduler:v1.20.0

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64:v1.20.0 k8s.gcr.io/kube-controller-manager:v1.20.0

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64:v1.20.0 k8s.gcr.io/kube-apiserver:v1.20.0

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64:v1.20.0 k8s.gcr.io/kube-proxy:v1.20.0

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2 k8s.gcr.io/pause:3.2

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0 k8s.gcr.io/etcd:3.4.13-0

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0 k8s.gcr.io/coredns:1.7.0

此时通过Docker命令就可以查看到这些Docker镜像信息了，命令如下：

root@kubernetesnode01:/opt/kubernetes-config# docker images

REPOSITORY                                                                          TAG                 IMAGE ID            CREATED             SIZE

k8s.gcr.io/kube-proxy                                                               v1.18.1             4e68534e24f6        2 months ago        117MB

registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64                v1.18.1             4e68534e24f6        2 months ago        117MB

k8s.gcr.io/kube-controller-manager                                                  v1.18.1             d1ccdd18e6ed        2 months ago        162MB

registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64   v1.18.1             d1ccdd18e6ed        2 months ago        162MB

k8s.gcr.io/kube-apiserver                                                           v1.18.1             a595af0107f9        2 months ago        173MB

registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64            v1.18.1             a595af0107f9        2 months ago        173MB

k8s.gcr.io/kube-scheduler                                                           v1.18.1             6c9320041a7b        2 months ago        95.3MB

registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64            v1.18.1             6c9320041a7b        2 months ago        95.3MB

k8s.gcr.io/pause                                                                    3.2                 80d28bedfe5d        4 months ago        683kB

registry.cn-hangzhou.aliyuncs.com/google_containers/pause                           3.2                 80d28bedfe5d        4 months ago        683kB

k8s.gcr.io/coredns                                                                  1.6.7               67da37a9a360        4 months ago        43.8MB

registry.cn-hangzhou.aliyuncs.com/google_containers/coredns                         1.6.7               67da37a9a360        4 months ago        43.8MB

k8s.gcr.io/etcd                                                                     3.4.3-0             303ce5db0e90        8 months ago        288MB

registry.cn-hangzhou.aliyuncs.com/google_containers/etcd-amd64                      3.4.3-0             303ce5db0e90        8 months ago        288MB
```

解决镜像拉取问题后再次执行 kubeadm 部署命令就可以完成 Kubernetes Master 控制节点的部署了，具体命令及执行结果如下：

```
root@kubernetesnode01:/opt/kubernetes-config# kubeadm init --config kubeadm.yaml --v=5

...

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

kubeadm join 10.211.55.13:6443 --token yi9lua.icl2umh9yifn6z9k \

   --discovery-token-ca-cert-hash sha256:074460292aa167de2ae9785f912001776b936cec79af68cec597bd4a06d5998d
```

从上面部署执行结果中可以看到，部署成功后 kubeadm 会生成如下指令：

```
kubeadm join 10.211.55.13:6443 --token yi9lua.icl2umh9yifn6z9k \

   --discovery-token-ca-cert-hash sha256:074460292aa167de2ae9785f912001776b936cec79af68cec597bd4a06d5998d
```

这个 kubeadm join 命令就是用来给该 Master 节点添加更多 Worker(工作节点) 的命令，后面具体部署 Worker 节点的时候将会使用到它。此外，kubeadm 还会提示我们第一次使用 Kubernetes 集群所需要配置的命令：

```
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

而需要这些配置命令的原因在于 Kubernetes 集群默认是需要加密方式访问的，所以这几条命令就是将刚才部署生成的 Kubernetes 集群的安全配置文件保存到当前用户的. kube 目录, 之后 kubectl 会默认使用该目录下的授权信息访问 Kubernetes 集群。如果不这么做的化，那么每次通过集群就都需要设置 “export KUBECONFIG 环境变量” 来告诉 kubectl 这个安全文件的位置。

执行完上述命令后，现在我们就可以使用 kubectl get 命令来查看当前 Kubernetes 集群节点的状态了，执行效果如下：

```
#  kubectl get nodes

NAME                  STATUS     ROLES                  AGE     VERSION

centos-linux.shared   NotReady   control-plane,master   6m55s   v1.20.0
```

在以上命令输出的结果中可以看到 Master 节点的状态为 “NotReady”，为了查找具体原因可以通过“kuberctl describe” 命令来查看下该节点（Node）对象的详细信息，命令如下：

```
# kubectl describe node centos-linux.shared
```

该命令可以非常详细地获取节点对象的状态、事件等详情，这种方式也是调试 Kubernetes 集群时最重要的排查手段。根据显示的如下信息：

```
...

Conditions

...

Ready False... KubeletNotReady runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized

...
```

可以看到节点处于 “NodeNotReady” 的原因在于尚未部署任何网络插件，为了进一步验证这一点还可以通过 kubectl 检查这个节点上各个 Kubernetes 系统 Pod 的状态，命令及执行效果如下：

```
# kubectl get pods -n kube-system

NAME                                       READY   STATUS    RESTARTS   AGE

coredns-66bff467f8-l4wt6                   0/1     Pending   0          64m

coredns-66bff467f8-rcqx6                   0/1     Pending   0          64m

etcd-kubernetesnode01                      1/1     Running   0          64m

kube-apiserver-kubernetesnode01            1/1     Running   0          64m

kube-controller-manager-kubernetesnode01   1/1     Running   0          64m

kube-proxy-wjct7                           1/1     Running   0          64m

kube-scheduler-kubernetesnode01            1/1     Running   0          64m
```

命令中 “kube-system” 表示的是 Kubernetes 项目预留的系统 Pod 空间（Namespace），需要注意它并不是 Linux Namespace，而是 Kuebernetes 划分的不同工作空间单位。回到命令输出结果，可以看到 coredns 等依赖于网络的 Pod 都处于 Pending（调度失败）的状态，这样说明了该 Master 节点的网络尚未部署就绪。

### **05、部署 Kubernetes 网络插件**

前面部署 Master 节点中由于没有部署网络插件，所以节点状态显示 “NodeNotReady” 状态。接下来的内容我们就来具体部署下网络插件。在 Kubernetes“一切皆容器”的设计理念指导下，网络插件也会以独立 Pod 的方式运行在系统中，所以部署起来也很简单只需要执行 “kubectl apply” 指令即可，例如以 Weave 网络插件为例：

```
# kubectl apply -f https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')

serviceaccount/weave-net created

clusterrole.rbac.authorization.k8s.io/weave-net created

clusterrolebinding.rbac.authorization.k8s.io/weave-net created

role.rbac.authorization.k8s.io/weave-net created

rolebinding.rbac.authorization.k8s.io/weave-net created

daemonset.apps/weave-net created
```

部署完成后通过 “kubectl get” 命令重新检查 Pod 的状态：

```
# kubectl get pods -n kube-system

NAME                                       READY   STATUS    RESTARTS   AGE

coredns-66bff467f8-l4wt6                   1/1     Running   0          116m

coredns-66bff467f8-rcqx6                   1/1     Running   0          116m

etcd-kubernetesnode01                      1/1     Running   0          116m

kube-apiserver-kubernetesnode01            1/1     Running   0          116m

kube-controller-manager-kubernetesnode01   1/1     Running   0          116m

kube-proxy-wjct7                           1/1     Running   0          116m

kube-scheduler-kubernetesnode01            1/1     Running   0          116m

weave-net-746qj                            2/2     Running   0          14m
```

可以看到，此时所有的系统 Pod 都成功启动了，而刚才部署的 Weave 网络插件则在 kube-system 下面新建了一个名叫 “weave-net-746qj” 的 Pod，而这个 Pod 就是容器网络插件在每个节点上的控制组件。

到这里，Kubernetes 的 Master 节点就部署完成了，如果你只需要一个单节点的 Kubernetes，那么现在就可以使用了。但是在默认情况下，**Kubernetes 的 Master 节点是不能运行用户 Pod 的，需要通过额外的操作进行调整**，在本文的最后将会介绍到它。

### **06、部署 Worker 节点**

为了构建一个完整的 Kubernetes 集群，这里还需要继续介绍如何部署 Worker 节点。实际上 Kubernetes 的 Worker 节点和 Master 节点几乎是相同的，它们都运行着一个 kubelet 组件，主要的区别在于 “kubeadm init” 的过程中，kubelet 启动后，Master 节点还会自动启动 kube-apiserver、kube-scheduler 及 kube-controller-manager 这三个系统 Pod。

在具体部署之前与 Master 节点一样，也需要在所有 Worker 节点上执行前面 “安装 kubeadm 及 Decker 环境” 小节中的所有步骤。之后在 Worker 节点执行部署 Master 节点时生成的 “kubeadm join” 指令即可，具体如下：

```
root@kubenetesnode02:~# kubeadm join 10.211.55.6:6443 --token jfulwi.so2rj5lukgsej2o6     --discovery-token-ca-cert-hash sha256:d895d512f0df6cb7f010204193a9b240e8a394606090608daee11b988fc7fea6 --v=5


...

This node has joined the cluster:

* Certificate signing request was sent to apiserver and a response was received.

* The Kubelet was informed of the new secure connection details.


Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

完成集群加入后为了便于在 Worker 节点执行 kubectl 相关命令，需要进行如下配置：

```
#创建配置目录

root@kubenetesnode02:~# mkdir -p $HOME/.kube

#将Master节点中$/HOME/.kube/目录中的config文件拷贝至Worker节点对应目录

root@kubenetesnode02:~# scp root@10.211.55.6:$HOME/.kube/config $HOME/.kube/

#权限配置

root@kubenetesnode02:~# sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

之后可以在 Worker 或 Master 节点执行节点状态查看命令 “kubectl get nodes”，具体如下：

```
root@kubernetesnode02:~# kubectl get nodes

NAME               STATUS     ROLES    AGE   VERSION

kubenetesnode02    NotReady   <none>   33m   v1.18.4

kubernetesnode01   Ready      master   29h   v1.18.4
```

通过节点状态显示此时 Work 节点还处于 NotReady 状态，具体查看节点描述信息如下：

```
root@kubernetesnode02:~# kubectl describe node kubenetesnode02

...

Conditions:

...

Ready False ... KubeletNotReady runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized

...
```

根据描述信息，发现 Worker 节点 NotReady 的原因也在于网络插件没有部署，继续执行 “部署 Kubernetes 网络插件” 小节中的步骤即可。但是要注意部署网络插件时会同时部署 kube-proxy，其中会涉及从 k8s.gcr.io 仓库获取镜像的动作，如果无法访问外网可能会导致网络部署异常，这里可以参考前面安装 Master 节点时的做法，通过国内镜像仓库下载后通过 tag 的方式进行标记，具体如下：

```
#从阿里云拉取必要镜像

docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64:v1.20.0

docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2

#将镜像重新打tag

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64:v1.20.0 k8s.gcr.io/kube-proxy:v1.20.0

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2 k8s.gcr.io/pause:3.2

如若一切正常，则继续查看节点状态，命令如下：

root@kubenetesnode02:~# kubectl get node

NAME               STATUS   ROLES    AGE     VERSION

kubenetesnode02    Ready    <none>   7h52m   v1.20.0

kubernetesnode01   Ready    master   37h     v1.20.0
```

可以看到此时 Worker 节点的状态已经变成 “Ready”，不过细心的读者可能会发现 Worker 节点的 ROLES 并不像 Master 节点那样显示“master” 而是显示了 < none>，这是因为新安装的 Kubernetes 环境 Node 节点有时候会丢失 ROLES 信息，遇到这种情况可以手工进行添加，具体命令如下：

```
root@kubenetesnode02:~# kubectl label node kubenetesnode02 node-role.kubernetes.io/worker=worker
```

再次运行节点状态命令就能看到正常的显示了，命令效果如下：

```
root@kubenetesnode02:~# kubectl get node

NAME               STATUS   ROLES    AGE   VERSION

kubenetesnode02    Ready    worker   8h    v1.18.4

kubernetesnode01   Ready    master   37h   v1.18.4
```

到这里就部署完成了具有一个 Master 节点和一个 Worker 节点的 Kubernetes 集群了，作为实验环境它已经具备了基本的 Kubernetes 集群功能！

### **07、部署 Dashboard 可视化插件**

在 Kubernetes 社区中，有一个很受欢迎的 Dashboard 项目，它可以给用户一个可视化的 Web 界面来查看当前集群中的各种信息。该插件也是以容器化方式进行部署，操作也非常简单，具体可在 Master、Worker 节点或其他能够安全访问 Kubernetes 集群的 Node 上进行部署，命令如下：

```
root@kubenetesnode02:~# kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.3/aio/deploy/recommended.yaml
```

部署完成后就可以查看 Dashboard 对应的 Pod 运行状态，执行效果如下：

```
root@kubenetesnode02:~# kubectl get pods -n kubernetes-dashboard

NAME                                         READY   STATUS    RESTARTS   AGE

dashboard-metrics-scraper-6b4884c9d5-xfb8b   1/1     Running   0          12h

kubernetes-dashboard-7f99b75bf4-9lxk8        1/1     Running   0          12h
```

除此之外还可以查看 Dashboard 的服务（Service）信息，命令如下：

```
root@kubenetesnode02:~# kubectl get svc -n kubernetes-dashboard

NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE

dashboard-metrics-scraper   ClusterIP   10.97.69.158    <none>        8000/TCP   13h

kubernetes-dashboard        ClusterIP   10.111.30.214   <none>        443/TCP    13h
```

需要注意的是，由于 Dashboard 是一个 Web 服务，从安全角度出发 Dashboard 默认只能通过 Proxy 的方式在本地访问。具体方式为在本地机器安装 kubectl 管理工具，并将 Master 节点 $HOME/.kube / 目录中的 config 文件拷贝至本地主机相同目录，之后运行 “kubectl proxy” 命令，如下:

```
qiaodeMacBook-Pro-2:.kube qiaojiang$ kubectl proxy

Starting to serve on 127.0.0.1:8001
```

本地 proxy 代理启动后，访问 Kubernetes Dashboard 地址，具体如下：

```
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

如果访问正常，就会看到如下图所示界面：

![](https://segmentfault.com/img/remote/1460000040107266)

如上图所示 Dashboard 访问需要进行身份认证，主要有 Token 及 Kubeconfig 两种方式，这里我们选择 Token 的方式，而 Token 的生成步骤如下：

**1)、创建一个服务账号**

首先在命名空间 kubernetes-dashboard 中创建名为 admin-user 的服务账户，具体步骤为在本地目录创建类似 “dashboard-adminuser.yaml” 文件，具体内容如下：

```
apiVersion: v1

kind: ServiceAccount

metadata:

 name: admin-user

 namespace: kubernetes-dashboard
```

编写文件后具体执行创建命令：

```
qiaodeMacBook-Pro-2:.kube qiaojiang$ kubectl apply -f dashboard-adminuser.yaml

Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply

serviceaccount/admin-user configured
```

**2)、创建 ClusterRoleBinding**

在使用 kubeadm 工具配置完 Kubernetes 集群后，集群中已经存在 ClusterRole 集群管理，可以使用它为上一步创建的 ServiceAccount 创建 ClusterRoleBinding。具体步骤为在本地目录创建类似 “dashboard-clusterRoleBingding.yaml” 的文件，具体内容如下：

```
apiVersion: rbac.authorization.k8s.io/v1

kind: ClusterRoleBinding

metadata:

 name: admin-user

roleRef:

 apiGroup: rbac.authorization.k8s.io

 kind: ClusterRole

 name: cluster-admin

subjects:

- kind: ServiceAccount

 name: admin-user

 namespace: kubernetes-dashboard
```

执行创建命令：

```
qiaodeMacBook-Pro-2:.kube qiaojiang$ kubectl apply -f dashboard-clusterRoleBingding.yaml

clusterrolebinding.rbac.authorization.k8s.io/admin-user created
```

**3)、获取 Bearer Token**

接下来执行获取 Bearer Token 的命令，具体如下：

```
qiaodeMacBook-Pro-2:.kube qiaojiang$ kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')

Name:         admin-user-token-xxq2b

Namespace:    kubernetes-dashboard

Labels:       <none>

Annotations:  kubernetes.io/service-account.name: admin-user

             kubernetes.io/service-account.uid: 213dce75-4063-4555-842a-904cf4e88ed1


Type:  kubernetes.io/service-account-token


Data

====

ca.crt:     1025 bytes

namespace:  20 bytes

token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IlplSHRwcXhNREs0SUJPcTZIYU1kT0pidlFuOFJaVXYzLWx0c1BOZzZZY28ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXh4cTJiIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIyMTNkY2U3NS00MDYzLTQ1NTUtODQyYS05MDRjZjRlODhlZDEiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.MIjSewAk4aVgVCU6fnBBLtIH7PJzcDUozaUoVGJPUu-TZSbRZHotugvrvd8Ek_f5urfyYhj14y1BSe1EXw3nINmo4J7bMI94T_f4HvSFW1RUznfWZ_uq24qKjNgqy4HrSfmickav2PmGv4TtumjhbziMreQ3jfmaPZvPqOa6Xmv1uhytLw3G6m5tRS97kl0i8A1lqnOWu7COJX0TtPkDrXiPPX9IzaGrp3Hd0pKHWrI_-orxsI5mmFj0cQZt1ncHarCssVnyHkWQqtle4ljV2HAO-bgY1j0E1pOPTlzpmSSbmAmedXZym77N10YNaIqtWvFjxMzhFqeTPNo539V1Gg
```

获取 Token 后回到前面的认证方式选择界面，将获取的 Token 信息填入就可以正式进入 Dashboard 的系统界面，看到 Kubernetes 集群的详细可视化信息了，如图所示：

![](https://segmentfault.com/img/remote/1460000040107267)

到这里就完成了 Kubernetes 可视化插件的部署并通过本地 Proxy 的方式进行了登录。在实际的生产环境中如果觉得每次通过本地 Proxy 的方式进行访问不够方便，也可以使用 Ingress 方式配置集群外访问 Dashboard，感兴趣的读者可以自行尝试下。也可以先通过通过暴露端口，设置 dashboard 的访问，例如：

```
#查看svc名称

# kubectl get sc -n kubernetes-dashboard


# kubectl edit services -n kubernetes-dashboard kubernetes-dashboard
```

然后修改配置文件，如下：

```
ports:

 - nodePort: 30000

   port: 443

   protocol: TCP

   targetPort: 8443

 selector:

   k8s-app: kubernetes-dashboard

 sessionAffinity: None

 type: NodePort
```

之后就可以通过 IP+nodePort 端口访问了！例如：

```
https://47.98.33.48:30000/
```

### **08、Master** 调整 Taint/Toleration 策略 **

在前面我们提到过，Kubernetes 集群的 Master 节点默认情况下是不能运行用户 Pod 的。而之所以能够达到这样的效果，Kubernetes 依靠的正是 **Taint/Toleration 机制**；而该机制的原理是一旦某个节点被加上 “Taint” 就表示该节点被“打上了污点”，那么所有的 Pod 就不能再在这个节点上运行。

而 Master 节点之所以不能运行用户 Pod，就在于其运行成功后会为自身节点打上 “Taint” 从而达到禁止其他用户 Pod 运行在 Master 节点上的效果（不影响已经运行的 Pod），具体可以通过命令查看 Master 节点上的相关信息，命令执行效果如下：

```
root@kubenetesnode02:~# kubectl describe node kubernetesnode01

Name:               kubernetesnode01

Roles:              master

...

Taints:             node-role.kubernetes.io/master:NoSchedule

...
```

可以看到 Master 节点默认被加上了 “node-role.kubernetes.io/master:NoSchedule” 这样的 “污点”，其中的值“NoSchedule” 意味着这个 Taint 只会在调度新的 Pod 时产生作用，而不会影响在该节点上已经运行的 Pod。如果在实验中只想要一个单节点的 Kubernetes，那么可以在删除 Master 节点上的这个 Taint，具体命令如下：

```
root@kubernetesnode01:~# kubectl taint nodes --all node-role.kubernetes.io/master-
```

上述命令通过在 “nodes --all node-role.kubernetes.io/master” 这个键后面加一个短横线 “-” 表示移除所有以该键为键的 Taint。

到这一步，一个基本的 Kubernetes 集群就部署完成了, 通过 kubeadm 这样的原生管理工具，Kubernetes 的部署被大大简化了，其中像证书、授权及各个组件配置等最麻烦的操作，kubeadm 都帮我们完成了。

### **09、Kubernetes 集群重启命令**

如果服务器断电，或者重启，可通过如下命令重启集群：

```
#重启docker

systemctl daemon-reload

systemctl restart docker


#重启kubelet

systemctl restart kubelet.service
```

以上就是在 CentOS 7 系统环境下搭建一组 Kubernetes 学习集群的详细步骤，其它 Linux 发行版本的部署方法也类似，大家可以根据自己的需求选择！

### **写在最后**

欢迎大家关注我的公众号【**风平浪静如码**】，海量 Java 相关文章，学习资料都会在里面更新，整理的资料也会放在里面。

觉得写的还不错的就点个赞，加个关注呗！点关注，不迷路，持续更新！！！