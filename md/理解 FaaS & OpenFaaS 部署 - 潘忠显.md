> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [panzhongxian.cn](https://panzhongxian.cn/cn/2021/03/faas-note-and-openfaas-deployment/)

> 最近在了解和学习使用 FaaS，其中的一些理解和实践过程，在这里做简单的笔记。

### 潘忠显 / 2021-03-26

* * *

最近在了解和学习使用 FaaS，其中的一些理解和实践过程，在这里做简单的笔记。理解不对的，请不吝指正。

一、FaaS 是什么？
-----------

*   FaaS (Function-as-a-Service，函数即服务)
    
*   一种**云计算**服务
    
*   FaaS 是 Serverless（无服务器计算）的子集
    
*   F/Function 是一小段具有完整功能的代码
    
*   **事件驱动**，运行代码以响应事件
    
*   使用者专注于开发功能函数，**无需关注部署**，FaaS 框架会管理内存、CPU、网络和其他资源均衡的计算机群
    
*   每个函数有单独的版本
    

### 1. 与 Serverless 的关系

Serverless 和 FaaS 通常相互混淆，实际上 FaaS 是 Serverless 的子集。CNCF 的白皮书中提到：

> Serverless 计算平台能够提供 FaaS 或 / 和 BaaS。

Serverless 专注于任何服务类别，包括计算、存储、数据库、消息传递、API 网关等，对用户屏蔽了服务器的配置、管理和计费。

而 FaaS 是 Serverless 架构中的核心技术，但它专注于事件驱动的计算范例，仅当需要响应事件或请求时，应用程序代码或容器才会运行。

### 2. 与微服务的对比

接口形式类似微服务类似。微服务将服务拆解成更小的服务（函数集合），然后打到镜像，运行在容器上。这些容器的维护和扩缩容，仍然需要服务提供者自己去关心。

FaaS 是微服务更进一步，会将服务拆成单个的函数，每个函数部署在 FaaS 平台，平台会规范函数、扩缩容、打补丁、维护环境。

![](https://panzhongxian.cn/images/faas-base/fn-mono-to-funcs.png)

### 3. 与 PaaS 的区别

FaaS 变更时间更短，在毫秒几倍。

对于 FaaS 来讲，不需要进行任何的容量规划。而 PaaS 提供了扩容能力但是相对缓慢，并且容量评估复杂。

用户部署的 FaaS 无法维护长连接，因为函数用完就会被销毁，但可以将连接维护在外部服务中以保持长连接。同样的，对于服务的状态，也只能存储在外部资源中来提供有状态的服务。

FaaS 可以不需要空闲流量，只在请求时才使用资源（一些云厂商支持预留一部分资源在空闲时使用）。而 PaaS 一般需要一定的资源来保持空闲容量。

FaaS 计费粒度更细，通常以计算时间、调用次数、使用的流量等计费，代码未运行时不产生费用，而 PaaS 通常以资源和时间粒度计费。以腾讯云的 SCF 为例：

> *   **资源使用费用**：0.00011108 元 / GBs
> *   **调用次数费用**：0.0133 元 / 万次
> *   **外网出流量费用**：各地域均有不同定价，中国大陆 0.80 元 / GB
> *   **预置并发闲置费用**：0.00005471 元 / GBs

![](https://panzhongxian.cn/images/faas-base/iaas-paas-saas-diagram.png)

### 4. FaaS 平台架构

FaaS 的简单架构包括以下四种元素：

*   事件源：触发函数实例执行的事件或者流
*   函数实例：单个函数实例，可以通过指令扩缩容
*   FaaS 控制器：部署、控制、监控函数实例
*   平台服务：必要的集群或者云服务

这四种元素的关系如下：

![](https://panzhongxian.cn/images/faas-base/serverless-framework.png)

更具体的可以参考后文中 OpenFaaS 的架构图。

另外一种角度看，函数开发完成后，会经历构建、部署、扩缩容等不同生命阶段：

![](https://panzhongxian.cn/images/faas-base/faas_framework_pipeline.png)

二、FaaS 供应商
----------

目前主流的云厂商，都有提供 FaaS 服务，并且会集成各种同步和异步的事件源，负责函数的扩缩容和部署，按照调用计费。

<table><thead><tr><th>厂商</th><th>FaaS 称呼</th><th>文档入口</th><th>使用框架</th></tr></thead><tbody><tr><td>Google</td><td>Cloud Functions</td><td><a href="https://cloud.google.com/functions">链接</a></td><td></td></tr><tr><td>AWS</td><td>Lambda</td><td><a href="https://aws.amazon.com/cn/serverless/getting-started/?serverless.sort-by=item.additionalFields.createdDate&amp;serverless.sort-order=desc">链接</a></td><td></td></tr><tr><td>腾讯云</td><td>SCF / Serverless Cloud Function / 云函数</td><td><a href="https://cloud.tencent.com/document/product/583">链接</a></td><td></td></tr><tr><td>阿里云</td><td>Function Compute / 函数计算</td><td><a href="https://help.aliyun.com/product/50980.html">链接</a></td><td></td></tr></tbody></table>

### 1. 腾讯云 SCF

本节动图演示使用腾讯云的云函数（[文档地址](https://cloud.tencent.com/document/product/583)），通过其提供的 Web IDE 编辑 Python 函数，快速的创建一个云函数。

_如果以下. gif 文件不能正常显示，请使用电脑浏览器打开_

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

几点说明：

*   腾讯云的云函数目前每个月有提供免费的额度（访问次数 + 运行资源 + 出流量）
*   如果通过 HTTP 请求访问函数，需要配置 API 网关触发
*   网关触发区分是否启用集成响应。若启用，需要按照[指定格式](https://cloud.tencent.com/document/product/583/12513#apiStructure)返回 JSON 结构，不然会报错。如果不启用，则会直接将变量 dump 出来。

### 2. 腾讯云 Serverless

本节动图演示使用腾讯云的 Serverless 服务（[文档地址](https://cloud.tencent.com/document/product/1154)），在几分钟内，将一个静态网站部署上去并能正常访问。

目前腾讯云的 Serverless 服务是免费的，对于想搭建自己网站的用户而言，无需购买云服务器，只需要花几块钱购置域名，配置其指向 Serverless 服务即可拥有自己的网站了。

_如果以下. gif 文件不能正常显示，请使用电脑浏览器打开_

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

几点说明：

*   静态页面配置实际是将产生的页面存放在腾讯云的对象存储服务 (COS)
*   Serverless 服务创建的桶命名为`my-bucket-XXX`，如果创建多个静态网站可能会有问题，建议删除桶 + 注销 Serverless 服务清理后，重试
*   如果需要使用单独的域名，可以根据 Serverless 页面的提示，将 API 网关域名地址配置到独立域名的 CNAME 中。CNAME 记录用于将一个域名（同名）映射到另一个域名（真实名称），域名解析服务器遇到 CNAME 记录会以映射到的目标重新开始查询
*   用到了 serverless framework(sls 指令) 进行部署，操作简单
*   需要注意 sls 有 npm 依赖，也可以直接[安装二进制文件](https://www.serverless.com/cn/framework/docs/quickstart/installation#binary)

### 3. 常见事件源

*   请求 API 网关的事件
*   定时触发
*   消息队列触发
*   COS Bucket 中的对象创建和对象删除事件

### 4. 应用场景汇总

各大厂商均在云函数的介绍页面，挂了一些应用场景的描述。下边以 Google Cloud Functions 和 AWS Lambda 的几个示图做简单的介绍。

*   Git push 触发 slack 消息推送

![](https://cloudx-bricks-prod-bucket.storage.googleapis.com/e95d4c33e1af335c2eeff991f3731fc78d46e448b7a578933b4503767da4c293.svg)

*   新用户注册触发短信发送

![](https://cloudx-bricks-prod-bucket.storage.googleapis.com/05160504d56cfbe1aaf1a16ef9f374f6d8e1087ec34a4f21d0e5dc26689f6e99.svg)

*   实时文件处理，数据库 / 流触发。函数功能包括审核、压缩、识别等。

![](https://cloudx-bricks-prod-bucket.storage.googleapis.com/a1d59ecf9e16669e82f5a8a82b46b334a0fcbc0e08d135840c954013354d7820.svg)

![](https://cloudx-bricks-prod-bucket.storage.googleapis.com/4253788a8b8a6e49d886bf49322e0faef425d15b251e5a96932f2008fca6d9ca.svg)

*   物联网传感器信息采集

![](https://d1.awsstatic.com/product-marketing/Lambda/Diagrams/product-page-diagram_Lambda-IoTBackends.3440c7f50a9b73e6a084a242d44009dc0fbe5fab.png)

*   无状态请求，比如天气状况，拉取数据库内容

![](https://d1.awsstatic.com/product-marketing/Lambda/Diagrams/product-page-diagram_Lambda-WebApplications%202.c7f8cf38e12cb1daae9965ca048e10d676094dc1.png)

*   用户触发状态变更推送

![](https://d1.awsstatic.com/product-marketing/Lambda/Diagrams/product-page-diagram_Lambda-MobileBackends_option2.00f6421e67e8d6bdbc59f3a2db6fa7d7f8508073.png)

这些应用场景具有一些特点，也可以说是哪些场景适合使用 FaaS：

*   并发相对较低
*   功能独立，函数间交互较少
*   多为无状态的
*   部分请求计算量大的

三、FaaS 平台的架构
------------

### 1. 当前 FaaS 框架

这里列举了当前较为流行的 FaaS 框架，以及他们的主页链接、Github 地址和 Star 数。Openfaas 是目前 Star 最多的，而且也更新也是几个项目中较频繁的，像 Fn 项目很久没有更新，这里就没有列出来。

<table><thead><tr><th>框架</th><th>链接</th><th>Star</th></tr></thead><tbody><tr><td>OpenFaaS</td><td><a href="https://www.openfaas.com/">主页</a> <a href="https://github.com/openfaas/faas">Github</a></td><td>19.5k</td></tr><tr><td>OpenWhisk</td><td><a href="https://openwhisk.apache.org/">主页</a> <a href="https://github.com/apache/openwhisk">Github</a></td><td>5.2k</td></tr><tr><td>Fission</td><td><a href="https://fission.io/">主页</a> <a href="https://github.com/fission/fission">Github</a></td><td>6k</td></tr><tr><td>Kubeless</td><td><a href="https://kubeless.io/">主页</a> <a href="https://github.com/kubeless/kubeless">Github</a></td><td>6.4k</td></tr><tr><td>Knative</td><td><a href="https://knative.dev/">主页</a> <a href="https://github.com/knative/serving">Github</a></td><td>3.6k</td></tr></tbody></table>

### 2. 部署工具 – Serverless Framework

**不是 Serverless 框架**，而是一个部署 Serverless 应用的工具。

说明文档：[https://github.com/serverless/serverless/blob/HEAD/README_CN.md](https://github.com/serverless/serverless/blob/HEAD/README_CN.md)

支持腾讯云 SCF、AWS Lambda 等部署。

### 3. OpenFaaS

OpenFaaS 主要是在 K8s 之上做了一些抽象的封装，每个函数或者服务会构建成镜像，以供编排。使用 PLONK 栈，分别是：

*   Prometheus
*   Linux/Linkerd*
*   OpenFaaS
*   NATS
*   Kubernetes

![](https://github.com/openfaas/faas/raw/master/docs/of-layer-overview.png)

![](https://raw.githubusercontent.com/openfaas/faas/master/docs/of-workflow.png)

OpenFaaS 默认会创建两个命名空间，`openfaas`和`openfaas-fn`。前者是用于维护 OpenFaaS 自身组件，后者用于维护创建的函数部署。“faas-netes" 子项目，用于部署 OpenFaaS 到 K8s 集群。

接下来一节会介绍在 MacOS 上部署 OpenFaaS 以及创建、部署函数的过程。

四、OpenFaaS 部署实践 [MacOS]
-----------------------

环境：Macbook (macOS 10.15.7)

部署时间：2021-03-21

部署完成后各个组件的版本情况：

*   minikube version: v1.18.1
*   Docker version: 18.09.2
*   kubectl client version: v1.20.5
*   faas-cli: version: 0.13.9

### 1. 安装 Docker

因为机器上之前就有安装 docker，这次没有重新安装。如果有更新需求，可以参考这里的[官方文档](https://docs.docker.com/docker-for-mac/install/)。

### 2. 安装 K8s 相关组件

安装 minikube

启动 minikube

```
brew install minikube
```

安装 kubectl

```
➜  ~ minikube start
😄  Darwin 10.15.7 上的 minikube v1.18.1
✨  根据现有的配置文件使用 docker 驱动程序
👍  Starting control plane node minikube in cluster minikube
🏃  Updating the running docker "minikube" container ...
🐳  正在 Docker 20.10.3 中准备 Kubernetes v1.20.2…
🔎  Verifying Kubernetes components...
    ▪ Using image kubernetesui/dashboard:v2.1.0
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v4
    ▪ Using image kubernetesui/metrics-scraper:v1.0.4
🌟  Enabled addons: storage-provisioner, default-storageclass, dashboard
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

查看 kubectl 版本信息

### 3. 通过 faas-nets 部署 OpenFaaS

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

设置 base-auth 用户名 (admin) 和密码，这个密码后边有用。如果为了简单测试，也可以将其设置为简单易记的短字符串。

```
kubectl version
```

### 4. 拉起并展示 minikube 的控制面板

启动 minikube 控制面板、获得访问地址。注意，启动之后不能 `Ctrl + C` 结束掉，不然 HTTP 服务就关掉了。

```
git clone https://github.com/openfaas/faas-netes
cd faas-netes
kubectl apply -f namespaces.yml
kubectl apply -f ./yaml/
```

点击最后的地址即可访问，minikube 面板如下（选择了命名空间 `openfaas`）：

![](https://panzhongxian.cn/images/faas-base/openfaas-namespace-minikube-dashboard.png)

### 5. 拉起并展示 OpenFaaS 的控制面板

查询 OpenFaaS 控制面板地址，这里涉及到 minikube 获得服务访问地址的方法：

```
export PASSWORD=$(head -c 12 /dev/urandom | shasum| cut -d' ' -f1)
echo $PASSWORD
kubectl -n openfaas create secret generic basic-auth --from-literal=basic-auth-user=admin --from-literal=basic-auth-password=$PASSWORD
```

点开最后的地址即可访问，用户名和密码就是在 [3. 通过 faas-nets 部署 OpenFaaS](#3-%E9%80%9A%E8%BF%87-faas-nets-%E9%83%A8%E7%BD%B2-openfaas) 小节中设置的 admin 和`$PASSWORD`。OpenFaaS 面板如下：

![](https://panzhongxian.cn/images/faas-base/openfaas_dashboard.png)

页面提示中可以看到，当前没有任何的函数部署。如果要部署新的函数，有两种方式：一种是通过**页面进行操作**，就是点击页面上的 "Deploy New Function" 按钮，一种是通过 **faas-cli** 进行部署。

### 6. 通过页面部署函数

点击![](https://panzhongxian.cn/images/faas-base/openfaas_deploy_button.png) 按钮，进入选择常用函数页面，也可以选择自定义的函数部署，不过要事先创建好镜像。这里选择常用函数 shasum 来部署（点选 shasum- 点击右下角 DEPLOY 即可）：

![](https://panzhongxian.cn/images/faas-base/deploy_faas_shasum.png)

#### 调用测试

部署完成之后，在页面左侧会出现一个 "shasum" 的函数，点击进入函数信息页面。其中包括对应的镜像地址、函数状态、函数 URL、函数的进程

![](https://panzhongxian.cn/images/faas-base/invoke_faas_shasum_function.png)

### 7. 通过 faas-cli 部署函数

#### 7.1 安装 faas-cli

Linux 上可以通过下边的方式安装：

```
➜  ~ minikube dashboard --url
🤔  正在验证 dashboard 运行情况 ...
🚀  Launching proxy ...
🤔  正在验证 proxy 运行状况 ...
http://127.0.0.1:51502/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/
```

#### 7.2 查询 OpenFaaS 提供的模板

```
➜  ~ minikube service gateway-external --url -n openfaas
🏃  Starting tunnel for service gateway-external.
|-----------|------------------|-------------|------------------------|
| NAMESPACE |       NAME       | TARGET PORT |          URL           |
|-----------|------------------|-------------|------------------------|
| openfaas  | gateway-external |             | http://127.0.0.1:53156 |
|-----------|------------------|-------------|------------------------|
http://127.0.0.1:53156
❗  Because you are using a Docker driver on darwin, the terminal needs to be open to run it.
```

#### 7.3 创建函数

根据[文档指引](https://docs.openfaas.com/cli/templates/#python-3-templates)，创建一个 Python3 的简单函数：

```
brew install faas-cli
```

得到的主要文件

```
curl -sSL https://cli.openfaas.com | sh
```

#### 7.4 构建函数镜像

有很多构建选项，可以参考[文档](https://docs.openfaas.com/cli/build/)。这里简单的进行构建：

```
faas-cli template store list
```

#### 7.5 登入

在之前的 [3. 通过 faas-nets 部署 OpenFaaS](#3-%E9%80%9A%E8%BF%87-faas-nets-%E9%83%A8%E7%BD%B2-openfaas) 小节中有提到过的密码，同时需要指定网关地址，即[#拉起并展示 OpenFaaS 的控制面板](#%E6%8B%89%E8%B5%B7%E5%B9%B6%E5%B1%95%E7%A4%BAOpenFaaS%E7%9A%84%E6%8E%A7%E5%88%B6%E9%9D%A2%E6%9D%BF)小节中提到的链接。

```
mkdir fn && cd fn
faas-cli new pycon --lang python3
```

其实如果不想记住这个忘关地址，可以每次去动态获取也可以：

```
pycon.yml
pycon/handler.py
pycon/requirements.txt
```

#### 7.6 推送镜像（使用 Minikube 才有的问题）

在 OpenFaaS [ISSUE #135](https://github.com/openfaas/faas-netes/issues/135) 中有提到 minikube 情况有点特殊，不能直接使用 docker 中存在的镜像。一定需要取从 DockerHub 或本地的 Registry 才行。

> `minikube` is a special case - you will probably have to push to a registry for that - either the Docker Hub or a local registry - see also the [Minkube registry add-ons](https://github.com/kubernetes/minikube/blob/master/docs/addons.md).

不然到话，部署到时候会有如下的错误，提示拉取不到镜像：

> Failed to pull image “pycon:latest”: rpc error: code = Unknown desc = Error response from daemon: pull access denied for pycon, repository does not exist or may require ‘docker login’: denied: requested access to the resource is denied

[不使用 Docker Hub 的本地推送方法](https://github.com/openfaas/faas-netes/issues/135#issuecomment-386859420)看上去有些麻烦，所以我直接登录并推送镜像到 Docker Hub 了。

直接 docker push 不行：

```
faas-cli build -f pycon.yml
```

> The push refers to repository [docker.io/library/pycon]
> 
> …
> 
> denied: requested access to the resource is denied

原因是第一行显示的那样，他会认为要将那个镜像直接推送到`docker.io/library/pycon`而不是你的个人目录。可以通过打一个 tag（简单的认为是对那个镜像加了个软链一样）来指向自己的仓库，然后再进行推送：

```
faas-cli login --password 9013a977f4ded57a72f395f03dff3218df254583 --gateway http://127.0.0.1:53156
```

最后将 pycon.yml 文件做一下修改：

```
faas-cli login --password 9013a977f4ded57a72f395f03dff3218df254583 --gateway $(minikube service gateway-external --url)
```

#### 7.7 部署函数

同样需要指定网关：

```
docker image push pycon:latest
```

成功后会列出函数的访问地址：

```
docker login
docker tag pycon panzhongxian/pycon
docker image push panzhongxian/pycon:latest
```

查看是否成功部署了 pod

```
image: pycon:latest
# 修改为⬇️
image: panzhongxian/pycon:latest
```

返回我们通过页面部署的一个 pod 和另外一个通过 faas-cli 部署的 pod

> NAME READY STATUS RESTARTS AGE pycon-76f6d9876b-t8zs9 1/1 Running 1 3m shasum-67dddcdb6-6jm9t 1/1 Running 1 175m

#### 7.8 测试函数

```
faas-cli deploy -f pycon.yml --gateway http://127.0.0.1:53156
```

#### 7.9 修改函数

修改 "pycon/handler.py" 文件，最初的文件内容为：

```
Deploying: pycon.

Deployed. 202 Accepted.
URL: http://127.0.0.1:53156/function/pycon
```

`req`是 HTTP POST 的内容。可以对其进行计算之后，然后再返回结果。再根据上述过程的部署即可。

### 8. 清理 OpenFaaS 部署

当我们测试完之后，需要清理对应的进程、资源。具体步骤如下：

清理账号密码

```
kubectl get pods -n openfaas-fn
```

进入目录 faas-netes，清理资源文件

清理函数命名空间 openfaas-fn

```
curl http://127.0.0.1:53156/function/pycon -X POST --data "test"
```

清理框架命名空间 openfaas

```
def handle(req):
    """handle a request to the function
    Args:
        req (str): request body
    """

    return req
```

五、附录
----

### K8s 的一些常用操作

之前没有利用 K8s 进行容器编排的经验，这里记录一下在部署过程中的一些技巧。

#### JSONPath 支持

Kubectl 使用 JSONPath 表达式来过滤 JSON 对象中的特定字段并格式化输出。本节列出一些查询的常用操作，其他详细参数可以参考下边链接：

[https://kubernetes.io/zh/docs/reference/kubectl/jsonpath/](https://kubernetes.io/zh/docs/reference/kubectl/jsonpath/)

**显示所有 pod 中的容器名、镜像名**

```
kubectl delete secret basic-auth -n openfaas
```

#### 登录运行中的容器和镜像调试

参考[使用容器 exec 进行调试](https://kubernetes.io/zh/docs/tasks/debug-application-cluster/debug-running-pod/#container-exec)，主要的用法是：

```
kubectl delete -f ./yaml
```

也就是上述命令让运行的容器运行指定的指令：

```
kubectl delete namespace openfaas-fn
```

如果要通过终端进入到容器的 shell，需要加上`-t -i`或者长指令`--stdin --tty`：

```
kubectl delete namespace openfaas
```

TODO
----

*   压测触发 OpenFaaS 的自动扩容
*   OpenFaaS 框架细节

With Fn, each function is a Docker container. Containers are lightweight and can be customized to include just the tools and languages you need to execute your function. Thus, containers are an ideal option for running function code. https://github.com/fnproject/docs/blob/master/fn/general/introduction.md https://github.com/fnproject/docs/blob/master/fn/general/introduction.md FaaS（功能即服务）是一种云计算服务，它使您能够响应事件执行代码，而无需构建和启动微服务应用程序通常会遇到的复杂基础架构。 在 Internet 上托管软件应用程序通常需要配置和管理虚拟或物理服务器，以及管理操作系统和 Web 服务器托管进程。使用 FaaS，物理服务，虚拟机操作系统和 Web 服务器软件管理全部由云服务提供商自动处理。这使您可以只专注于应用程序代码中的各个功能。 概念区分 FaaS 与 Serverless Serverless 和功能即服务（FaaS）通常相互混淆，但事实是 FaaS 实际上是 Serverless 的子集。Serverless 专注于任何服务类别，包括计算，存储，数据库，消息传递，API 网关等，其中最终用户看不到服务器的配置，管理和计费。另一方面，虽然 FaaS 也许是 Serverless 体系结构中最核心的技术，但它专注于事件驱动的计算范例，其中应用程序代码或容器仅响应事件或请求而运行。 如果您希望将应用程序高效，经济高效地迁移到云中，那么 FaaS 是一个非常有价值的工具。以下是您将享受的一些好处： 将更多精力放在代码上，而不是基础架构上：借助 FaaS，您可以将服务器划分为可以自动独立缩放的功能，因此您不必管理基础架构。这使您可以专注于应用程序代码，并可以大大缩短产品上市时间。 使用时，仅需为使用的资源付费：使用 FaaS，只需在发生操作时才付费。完成该操作后，一切都将停止 - 没有代码运行，没有服务器空闲，也没有发生任何费用。因此，FaaS 具有成本效益，特别是对于动态工作负载或计划任务。FaaS 还为高负载场景提供了卓越的总拥有成本。 自动按比例放大或缩小：使用 FaaS，可以根据需要自动，独立和即时地按比例缩放功能。当需求下降时，FaaS 会自动缩减。 获得强大的云基础架构的所有好处： FaaS 提供固有的高可用性，因为它分布在每个地理区域的多个可用性区域中，并且可以在不增加成本的情况下跨任意数量的区域进行部署。 原则和最佳做法 您可以遵循一些最佳实践，以使 FaaS 的部署更容易，更有效： 使每个功能仅执行一个动作： FaaS 功能应设计为响应事件而完成一项工作。使您的代码范围受到限制，高效且轻量，从而使函数可以快速加载和执行。 不要让函数调用其他函数： FaaS 的价值在于函数的隔离。功能过多会增加您的成本，并消除功能隔离的价值。 在函数中使用尽可能少的库：使用太多的库会降低函数的速度，并使它们难以扩展。 用例 由于 FaaS 可以轻松隔离事务和扩展事务，因此非常适合大容量和令人尴尬的并行工作负载。它也可以用于创建后端系统或用于诸如数据处理，格式转换，编码或数据聚合之类的活动。 FaaS 还是 Web 应用程序，后端，数据 / 流处理或为 IoT 设备创建在线聊天机器人或后端的好工具。FaaS 可以帮助您管理和使用第三方服务。例如，如果您正在考虑开发 Android 应用，则可以采用 FaaS 方法来控制成本。由于仅当您的应用程序连接到云以执行特定功能（例如批处理）时才需要付费，因此其成本可能比使用传统方法要低得多。 FaaS 还可以大大提高计算性能。例如，最近有两名学生与 IBM 工程师合作，探索如何利用 IBM Cloud Functions 进行蒙特卡洛模拟（用于估算某些难以预测的事件的未来结果的数学方法）来估算股票价格。蒙特卡洛模拟被认为是重要的高性能计算工作量。蒙特卡洛和 IBM Cloud Functions 的结合使团队能够大规模运行计算，并使他们能够专注于业务逻辑。使用 FaaS，该团队在大约 90 秒内完成了 1000 次并发调用，从而完成了整个 Monte Carlo 模拟。相比之下，在具有四个 CPU 核心的笔记本电脑上运行相同的流程需要 247 分钟的时间，并且 CPU 利用率几乎达到 100％。 要查看更多 FaaS 用例示例，请查看“IBM Cloud Functions 提供的关键优势概述” 。 FaaS 与 PaaS，容器和 VM FaaS，PaaS（平台即服务），容器和虚拟机（VM）在无服务器生态系统中都起着至关重要的作用。由于 FaaS 是无服务器堆栈中最核心和最具定义性的元素，因此值得探讨 FaaS 与当今市场上其他常见计算模型在关键属性上的不同之处： 设置时间：毫秒，相比其他型号的分钟和小时。 正在进行的管理：没有，与 PaaS，容器和 VM 分别从轻到重的滑动比例相比。 弹性缩放：与其他提供自动但缓慢缩放的模型相比，每个动作总是即时且固有地缩放，而其他模型则需要仔细调整自动缩放规则。 容量规划：与其他模型相比，不需要进行任何规划，而其他模型则需要一些自动缩放和一些容量规划的组合。 持久连接和状态：持久连接和状态的能力必须保留在外部服务 / 资源中。其他模型可以利用 http，长时间保持打开的套接字或连接的状态，并且可以在两次调用之间将状态存储在内存中。 维护：所有维护均由 FaaS 提供商管理。PaaS 也是如此。容器和 VM 需要大量维护，包括更新 / 管理操作系统，容器映像，连接等。 高可用性（HA）和灾难恢复（DR）：同样，HA 在 FaaS 模型中是固有的，而无需付出额外的努力或成本。其他模型需要额外的成本和管理工作。对于虚拟机和容器，可以自动重新启动基础架构。 资源利用率：资源永远不会空闲 - 仅在请求时才调用它们。所有其他型号至少具有一定程度的空闲容量。 资源限制： FaaS 是唯一在代码大小，并发激活，内存，运行长度等方面具有资源限制的模型。 计费粒度和计费：每 100 毫秒的块，相比其他模型的小时（有时是分钟）。 Kubernetes / Knative 和 FaaS Kubernetes 和 Knative 是 FaaS 背后 “管道” 的一种实现。Kubernetes 是一个开源的容器编排工具，对管理云应用程序至关重要。通过 Knative，您可以在 Kubernetes 集群中无服务器运行。 Knative 和 Kubernetes 的结合意味着您可以利用 Kubernetes 的功能（例如监视，安全性，日志记录和身份验证），并将它们与 Knative 的优点（例如自动化容器构建，完全可移植性以及在混合环境中工作）相结合。 这项技术的创建者认为，开发人员在构建云应用程序时不必在无服务器和容器之间进行选择。目的是通过无服务器的强大扩展和按需访问来提高容器的可用性和一致性。 https://translate.google.com/translate?hl=en&sl=en&tl=zh-CN&u=https%3A%2F%2Fwww.ibm.com%2Fcloud%2Flearn%2Ffaas https://www.ibm.com/cloud/learn/faas FaaS 背景知识准备 FaaS 对于开放平台的意义 当前架构的 ### 并发、效率如何???