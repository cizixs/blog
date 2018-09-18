---
layout: post
title: "容器生态系统简介"
excerpt: "很少有技术能够像 docker 这样一出来就收到关注，并在很短的时间里发展壮大，而且几乎所有的技术公司都开始使用或者希望使用。"
categories: blog
tags: [kubernetes, container, docker, mesos, swarm]
comments: true
share: true
---

很少有技术能够像 docker 这样一出来就收到关注，并在很短的时间里发展壮大，而且几乎所有的技术公司都开始使用或者希望使用。随着 docker 的出现，配置管理、微服务、数据中心自动化、devops 多个领域都重新焕发新机，好像 IT 行业的整个架构都要重新定义一样。

这篇文章我会介绍一下目前（2017年 3月）容器圈子（主要还是 docker） 一些主要的参与者，它们共同组成了繁荣的容器生态圈。

![](https://www.chrisbarra.me/images/docker.png)

## 容器引擎

容器引擎（Engine）或者容器运行时（Runtime）是容器系统的核心，也是很多人使用“容器”这个词语的指代对象。容器引擎能够创建和运行容器，而容器的定义一般是以文本方式保存的，比如 Dockerfile。

- [Docker Engine](https://www.docker.com/docker-engine) ：目前最流行的容器引擎，也是业界的事实标准
- [Rkt](https://coreos.com/rkt/docs/latest/)：CoreOS 团队推出的容器引擎，目前处于活跃发展阶段，被 kubernetes 调度系统支持

## 云提供商（国外篇）

容器飞速发展的时候，很多公司反应迅速，都相继推出了自己的公有云或者私有云的容器解决方案。国外比较有名的容器云提供商包括：

- [Amazon EC2 Container Serveice](https://aws.amazon.com/ecs/)（ECS）：云计算巨头 AWS 推出的容器服务，比较吸引人的是构建在 EC2 上面的 ECS 是免费的，用户只需要为底层的 EC2 资源付费
- [Google Container Engine](https://cloud.google.com/container-engine/)（GKE）：谷歌在错过大数据的福利之后，在云计算领域开始醒悟。在社区推出 kubernetes，企图制定容器调度集群的标准，而同时推出公有的 GKE 服务
- [Azure Container Service](https://azure.microsoft.com/en-us/services/container-service/)：作为巨头，微软也积极推出了容器服务，凭借积累的技术和商业资源迅速崛起
- [Jelastic](https://jelastic.com/docker/)
- [Docker Cloud](https://cloud.docker.com/)：docker 自家的容器云服务，在收购了 Tutum 公司之后，利用 docker swarm 集群管理技术推出了方便使用的产品

## 云提供商（国内篇）

每次新技术的出现都会催生一堆公司，有大公司也有创业公司。不管是大数据、Iaas、人工智能还是现在的容器。国内公司目前对技术已经非常敏感，追逐和使用新技术的脚步从来没有落后过。虽然现在还没有在核心技术上制定标准和掀起浪潮，但是我相信不久之后中国会出现能够影响世界的技术。

目前国内做容器比较知名的公司包括：

- [阿里云](https://www.aliyun.com/product/containerservice)：作为国内云服务的一哥，阿里云反应迅速，和 docker 建立 [官方合作](http://www.infoq.com/cn/news/2016/10/Docker-Aliyun-inPartnership-2016)，也开始为自己的容器产品布道
- [网易蜂巢](https://c.163.com/)：网易在自家的 IaaS 平台成熟之后，在此之上推出了容器云服务，据说内部产品已经大都迁移到容器
- [灵雀云](http://www.alauda.cn/)：云雀科技成立于2014年，由原微软Azure云平台的核心创始团队创立
- [时速云](https://www.tenxcloud.com/)：tenxcloud，基于 Kubernetes 的容器云计算平台，提供公有云和企业云服务
- [数人云](https://www.shurenyun.com/)：基于 mesos 研发的轻量级 Pass 平台
- [才云](https://www.caicloud.io/)：使用的是 kubernetes 方案，2015 年成立的公司，提供私有云服务和人工智能解决方案
- [daocloud](https://www.daocloud.io/)：企业级应用云平台及解决方案，坐标上海

## 容器编排系统

容器与虚拟机相比有个很大的优势就是轻量，这个特性的量变很快就引发了质变，让容器在各个技术角落施展拳脚。不过轻量级的容很容易造成混乱，面对成百上千乃至更多的容器，必须要有统一的管理平台。所以目前容器技术最热烈的激战也在这个领域，所有要使用容器的企业必须要在容器管理系统做出自己的抉择。

- [Kubernetes](https://kubernetes.io/)：Google 家开源的容器管理系统，起源于内部历史悠久的 Borg 系统。因为其丰富的功能被多家公司使用，不仅支持 docker ，还支持其他容器（比如 Rkt），但是也相对复杂，易用性差一点
- [Docker Swarm](https://docs.docker.com/engine/swarm/)： 从 docker1.12.0 版本之后，docker 就推出了 docker swarm 模式。用户能够轻松快速搭建出来 docker 容器集群，几乎完全兼容 docker API 的特性让它很容易被用户接受。可以说潜力很大，至于最后能发展成什么样子还要时间检验
- [Mesosphere](https://mesosphere.com/)：起源于 Apache Mesos 的调度框架，目标是成为数据中心的操作系统，完全接管数据中心的管理工作
- [Rancher](http://rancher.com/)：其易用的界面和完全开源的特性吸引不少刚接触容器的技术人员，同时兼容 kubernetes、mesos 和 swarm 集群系统。但目前社区不是很活跃，二次开发的难度较大
- [Nomad](https://www.nomadproject.io/)：HashiCorp 开源的集群管理和调度系统，如果需要其他功能（比如服务发现、密码管理等）需要自己使用其他工具进行集成

## 容器基础镜像

容器虽然轻量，但是传统的基础镜像并非如此。因此有很多企业尝试着打造专门为容器而生的操作系统，希望能成为容器时代的新选择。

- [CoreOS](https://coreos.com/os/docs/latest)：以容器为中心的操作系统，配置管理、自动扩容、安全等方面有一套完整的工具
- [Project Atomic](https://www.projectatomic.io/)：一个轻量级的操作系统，可以运行 docker、kubernetes、rpm 和 systemd
- [Ubuntu Core](https://www.ubuntu.com/core)：轻量级 ubuntu 操作系统，适合运行 IoT 设备或者容器集群
- [Rancher OS](http://rancher.com/rancher-os/)：只有 50M+ 的操作系统，为运行 docker 容器打造。有趣的是，它可以让系统容器和用户容器运行在不同的 Docker Daemon 上，从而实现隔离效果
- [Project Photon](https://vmware.github.io/photon/)：VMware 开源的项目，旨在提供极简化的容器主机系统

## 镜像 registry

镜像 registry 是存储镜像的地方，可以方便地在团队、公司或者世界各地分享容器镜像，也是运行容器最基本的基础设施。

- [Docker Registry](https://github.com/docker/distribution)：Docker 公司提供的开源镜像服务器，也是目前最流行的自建 registry 的方案
- [Dockerhub](https://hub.docker.com/)：docker 公司提供的公共镜像 registry，可以通过 UI 来查看和管理镜像，上面也有大量的标准镜像可以下载
- [Quay](https://quay.io/)：提供镜像管理和安全检查服务的  公有 registry
- [Harbor](https://github.com/vmware/harbor)：企业级的镜像 registry，提供了权限控制和图形界面

## 容器监控

- [cAdvisor](https://github.com/google/cadvisor)：Google 开源的容器使用率和性能监控工具
- [Datadog Docker](https://www.datadoghq.com/blog/monitor-docker-datadog/)：能够收集 docker 的运行信息，并发送到 Datadog 进行分析
- [NewRelic Docker](https://newrelic.com/partner/docker)：收集 docker 的运行信息，并发送到 NewRelic 进行分析
- [Sysdig](https://sysdig.com/)：同时提供开源版本和企业版本，能够监控容器使用率和性能，并对性能就行分析
- [logdns](https://logdns.com)：能够收集容器或者集群的日志，发送到 logdna 服务器端就行分析、设置告警等

## 网络

容器的大规模使用，也对网络提供了更高的要求。网络的不灵活也是很多企业的短板，目前也有很多公司和项目在尝试解决这些问题，希望提出容器时代的网络方案。

- [Weave Net](https://www.weave.works/products/weave-net/)：weaveworks 给出的网络的方案，使用 vxlan 技术， 支持网络的隔离和安全
- [Flannel](https://github.com/coreos/flannel)：CoreOS 开源的网络方案，为 kubernetes 设计，支持不同的后端实现
- [Calico](https://www.projectcalico.org/)：一个纯三层的网络解决方案，使用 BGP 协议进行路由，可以集成到 openstack 和 docker
- [Contiv](https://contiv.github.io/): 能够打通物理机、虚拟机和容器之间连通性的网络方案

## 服务发现

容器和微服务的结合创造了另外的热潮，也让服务发现成功了热门名词。可以轻松扩展微服务的同时，也要有工具来实现服务之间相互发现的需求。目前主要有三种工具，当然它们可能已经集成到其他的容器管理系统中。

- [etcd](https://github.com/coreos/etcd)：CoreOS 开源的分布式 key-value 存储，通过 HTTP 协议提供服务，因此使用起来简单。但是 etcd 只是一个 key-value 存储，默认不支持服务发现，需要三方工具来集成。kubernetes 默认就使用 etcd 作为存储
- [consul](https://www.consul.io/)：HashiCorp 开源的服务发现和配置管理工具，自带服务发现特性（DNS Server）。它是强一致性的数据存储，使用 gossip 协议形成动态集群
- [zookeeper](https://zookeeper.apache.org/)：比较悠久的服务发现项目，起源于 Hadoop 社区，优点是成熟、可靠、功能丰富，缺点是使用 Java 开发，配置比较麻烦

## 参考资料

- [A brief guide to the docker ecosystem](http://blog.dennybritz.com/2015/10/01/a-brief-guide-to-the-docker-ecosystem/)
- [服务发现：Zookeeper vs etcd vs Consul](http://dockone.io/article/667)
- [Linux Container Operating Systems](http://linoxide.com/containers/linux-container-operating-systems/)
