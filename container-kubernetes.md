# 简介
Kubernetes项目是Google在2014年启动的。Kubernetes构建在[Google公司十几年的大规模高负载生产系统运维经验](https://research.google.com/pubs/pub43438.html)之上，同时结合了社区中各项最佳设计和实践。

Kubernetes一个用于容器集群的自动化部署、扩容以及运维的开源平台。Kubernetes构建于Docker之上，采用docker作为容器引擎。


## 特性

使用Kubernetes，你可以快速高效地响应客户需求：

 - 动态地对应用进行扩容。
 - 无缝地发布新特性。
 - 仅使用需要的资源以优化硬件使用。

它有如下特性：

* **简洁的**：轻量级，简单，易上手
* **可移植的**：公有，私有，混合，多重云（multi-cloud）
* **可扩展的**: 模块化, 插件化, 可挂载, 可组合
* **可自愈的**: 自动布置, 自动重启, 自动复制

## 基本概念
Google在Kubernetes中，抽象了以下几个容器调度组件：

* kubctl：操作整个调度系统使用的命令行工具。
* pod：一组容器的集合，pod内的容器一起被部署。
* volume：可供容器进行存储的卷。
* label：标识pod以便分组管理的标签。
* replication controller：用以保证集群内的容器在任何时刻都有指定数据的副本在运行。
* service：service是后端服务的前端网络代理，以便后端更新或调整时前端不受影响。

## 系统架构
![](http://ruizeng.net/content/images/2015/12/kub-architecture.png)

图中可以看到，一个完整的Kubernetes集群可分为两部分，控制中心和若干节点。

### 节点
一般一个节点是一个单独的物理机或虚拟机，其上安装了Docker来负责镜像下载和运行。

除了Docker，每个节点还运行着一个kubelet程序，kubelet程序负责管理pods以及其依赖的volume，镜像以及容器。

节点上还运行这kube-proxy程序，实现的就是上述的service组件的功能，可以实现tcp和upd转发和负载均衡。

### 控制中心
控制中心负责节点的管理和调度工作。由以下几个组件组成：

* etcd：etcd负责配置信息的存储，利用etd的watch功能，能够及时发现集群的状态变化
* api server：主要负责提供rest接口并将对集群的状态操作写入etcd供其他模块执行。
* scheduler：通过api对pod进行调度
* controller manager：包含若干的控制组件，实现上述的`replication controller`等功能。

