---
layout: post
title: "kubernetes 简介： kubelet 和 pod"
excerpt: "kubelet 是 kubernetes 中的 worker 组件，负责在各个节点上收集信息和执行任务。"
categories: blog
tags: [docker, container, kubernetes, kubelet]
comments: true
share: true
---

## 简介

[kubernetes](http://cizixs.com/2016/07/12/kubernetes-intro) 是一个分布式的集群管理系统，在每个节点（node）上都要运行一个 worker 对容器进行生命周期的管理，这个 worker 程序就是 `kubelet`。

简单地说，`kubelet` 的主要功能就是定时从某个地方获取节点上 pod/container 的期望状态（运行什么容器、运行的副本数量、网络或者存储如何配置等等），并调用对应的容器平台接口达到这个状态。

集群状态下，kubelet 会从 master 上读取信息，但其实 kubelet 还可以从其他地方获取节点的 pod 信息。目前 kubelet 支持三种数据源：

![kubelet architecture](https://cizixs-blog.oss-cn-beijing.aliyuncs.com/006y8lVagw1f8vkhdub9zj31hc0u0djl.jpg)

1. 本地文件
2. 通过 url 从网络上某个地址来获取信息
3. API Server：从 kubernetes master 节点获取信息

从管理的对象来说，kubelet 目前支持 [docker](https://www.docker.com/) 和 [rkt](https://coreos.com/rkt/)，默认情况下使用的 docker。

## kubelet 主要功能

### pod 管理
 在 kubernetes 的设计中，最基本的管理单位是 pod，而不是 container。pod 是 kubernetes 在容器上的一层封装，由一组运行在同一主机的一个或者多个容器组成。如果把容器比喻成传统机器上的一个进程（它可以执行任务，对外提供某种功能），那么 pod 可以类比为传统的主机：它包含了多个容器，为它们提供共享的一些资源。

![](https://cizixs-blog.oss-cn-beijing.aliyuncs.com/006y8lVagw1f93i3nzo9aj30yg0lptc6.jpg)

之所以费功夫提供这一层封装，主要是因为容器推荐的用法是里面只运行一个进程，而一般情况下某个应用都由多个组件构成的。

pod 中所有的容器最大的特性也是最大的好处就是共享了很多资源，比如网络空间。pod 下所有容器共享网络和端口空间，也就是它们之间可以通过 `localhost` 访问和通信，对外的通信方式也是一样的，省去了很多容器通信的麻烦。

除了网络之外，定义在 pod 里的 volume 也可以 mount 到多个容器里，以实现共享的目的。

最后，定义在 pod 的资源限制（比如 CPU 和 Memory） 也是所有容器共享的。

### 容器健康检查

创建了容器之后，kubelet 还要查看容器是否正常运行，如果容器运行出错，就要根据设置的重启策略进行处理。检查容器是否健康主要有两种方式：在容器中执行命令和通过 HTTP 访问预定义的 endpoint。

先来说说执行命令的方式，简单来说就是在容器中执行某个 shell 命令，根据它的 exit code 来判断容器是否正常工作：

    livenessProbe:
          exec:
            command:
            - cat
            - /tmp/health
          initialDelaySeconds: 15
          timeoutSeconds: 1

另外一种就是 HTTP 的方式，通过向某个 url 路径发送 HTTP GET 请求，根据 response code 判断容器是否正常工作：

    livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            httpHeaders:
              - name: X-Custom-Header
                value: Awesome
          initialDelaySeconds: 15
          timeoutSeconds: 1
可以看到，你也可以自定义发送的 HTTP 请求的头部。

不管用什么方式，如果检测到容器不健康，kubelet 会删除该容器，并根据容器的重启策略进行处理（比如重启，或者什么都不做）。

### 容器监控
kubelet 还有一个重要的责任，就是监控所在节点的资源使用情况，并定时向 master 报告。知道整个集群所有节点的资源情况，对于 pod 的调度和正常运行至关重要。

kubelet 使用 [cAdvisor](https://github.com/google/cadvisor) 进行资源使用率的监控。cAdvisor 是 google 开源的分析容器资源使用和性能特性的工具，在 kubernetes 项目中被集成到 kubelet 里，无需额外配置。默认情况下，你可以在 `http://<host_ip>:4194` 地址看到 cAdvisor 的管理界面。

> Analyzes resource usage and performance characteristics of running containers.

除了系统使用的 CPU，Memory，存储和网络之外，cAdvisor 还记录了每个容器使用的上述资源情况。

## 运行和验证 kubelet

### 参数介绍

对于 kubelet 的配置，基本上都可以通过命令行启动时候的参数进行控制。因为 kubernetes 处于快速开发过程中，参数也可能会发生变化，这里给出 `1.4` 版本一些重要的参数含义：

参数        |   解释        |   默认值
------      | ---------     | -------
--address   | kubelet 服务监听的地址    | 0.0.0.0
--port  |   kubelet 服务监听的端口  |   10250
--read-only-port    | 只读端口，可以不用验证和授权机制，直接访问    |   10255
--allow-privileged  |   是否允许容器运行在 privileged 模式  | false
--api-servers   |   以逗号分割的 API Server 地址，用于和集群中数据交互  |   []
--cadvisor-port |   当前节点 cadvisor 运行的端口    |   4194
--config    |   本地 manifest 文件的路径或者目录    |   ""
--file-check-frequency  |   轮询本地 manifest 文件的时间间隔    |   20s
--container-runtime |   后端容器 runtime，支持 docker 和 rkt    | docker
--enable-server     |   是否启动 kubelet HTTP server    | true
--healthz-bind-address  |   健康检查服务绑定的地址，设置成 0.0.0.0 可以监听在所有网络接口   | 127.0.0.1
--healthz-port      |   健康检查服务的端口      |   10248
--hostname-override |   指定 hostname，如果非空会使用这个值作为节点在集群中的标识   | ""
--log-dir       |   日志文件，如果非空，会把 log 写到该文件 |   ""
--logtostderr   |   是否打印 log 到终端 | true
--max-open-files    |   允许 kubelet 打开文件的最大值   | 1000000
--max-pods  |   允许 kubelet 运行 pod 的最大值  |   110
--pod-infra-container-image | 基础镜像地址，每个 pod 最先启动的容器，会配置共享的网络   | gcr.io/google_containers/pause-amd64:3.0
--root-dir  |   kubelet 保存数据的目录  |   /var/lib/kubelet
--runonce   |   从本地 manifest 或者 URL 指定的 manifest 读取并运行结束就退出，和 `--api-servers` 、`--enable-server` 参数不兼容
--v     |   日志 level  |   0


### 启动 kubelet
在这篇教程中，我们会单独运行 kubelet，选择从本地目录中读取 pod manifest 信息去启动 pod。

```
[Unit]
Description=Kubelet Service
Documentation=http://kubernetes.com
After=network.target
Wants=network.target

[Service]
Type=simple
EnvironmentFile=-/etc/sysconfig/kubelet
ExecStartPre=/usr/bin/mkdir -p /etc/kubernetes/manifests
ExecStart=/usr/bin/kubelet \
    --allow-privileged=true \
    --config=/etc/kubernetes/manifests \
    --pod-infra-container-image=172.16.1.41:5000/google_containers/pause:0.8.0 \
    --hostname-override=192.168.8.100
TimeoutStartSec=0
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```

在这篇文章的例子中，我们会运行一个 pod，这个 pod 有两个容器：nginx 和 busybox。
nginx 不做任何修改，只是提供简单的 HTTP 服务；busybox 运行命令，在终端打印出 nginx 的访问日志。为了实现两个容器之间共享数据，我们使用 kubernetes 提供的 Volume 功能，

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-server
spec:
  containers:
  - name: nginx-server
    image: 172.16.1.41:5000/nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - mountPath: /var/log/nginx
      name: nginx-logs
  - name: log-output
    image: 172.16.1.41:5000/busybox
    command:
    - bin/sh
    args: [-c, 'tail -f /logdir/access.log']
    volumeMounts:
    - mountPath: /logdir
      name: nginx-logs
  volumes:
  - name: nginx-logs
    emptyDir: {}
```

把这个配置文件放到指定的目录，等待一段时间，我们就能 `docker ps` 看到有容器在运行了：

```
[root@localhost vagrant]# docker ps
CONTAINER ID        IMAGE                                            COMMAND                  CREATED             STATUS              PORTS               NAMES
66474b8cc336        172.16.1.41:5000/busybox                         "bin/sh -c 'tail -f /"   6 days ago          Up 6 days                               k8s_log-output.b93a0be3_nginx-server-192.168.8.100_default_dae69c6f50364bf8fcf20bc67f42a416_22c30949
90ab8ef86868        172.16.1.41:5000/nginx                           "nginx"                  6 days ago          Up 6 days                               k8s_nginx-server.b88325ae_nginx-server-192.168.8.100_default_dae69c6f50364bf8fcf20bc67f42a416_e152bb97
db27cfa9f58f        172.16.1.41:5000/google_containers/pause:0.8.0   "/pause"                 6 days ago          Up 6 days                               k8s_POD.b66e0279_nginx-server-192.168.8.100_default_dae69c6f50364bf8fcf20bc67f42a416_2236d469
```

我们可以看到三个容器：pause、nginx 和 busybox，容器的名字是 kubelet 自动生成的。通过 `docker inspect` 可以看到 nginx 和 busybox 的网络模式：

```
[root@localhost vagrant]# docker inspect -f '{{ .HostConfig.NetworkMode }}' 90ab
container:db27cfa9f58f7b1047fb230da951453ca15e0961ca36ee7999ab3ed8ba5da814

[root@localhost vagrant]# docker inspect -f '{{ .HostConfig.NetworkMode }}' 6647
container:db27cfa9f58f7b1047fb230da951453ca15e0961ca36ee7999ab3ed8ba5da814
```

使用的是 `container` 模式，对应的 id 就是 pause 容器的。通过查看各个容器的网络配置也能验证这一点，所有的容器网络都是一样的：

```
[root@localhost vagrant]# docker exec 90ab ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
9: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:2/64 scope link tentative dadfailed
       valid_lft forever preferred_lft forever
```

而 nginx 和 busybox 的配置信息中并没有 ip 地址，pause 容器中有：

```
[root@localhost vagrant]# docker inspect -f '{{ .NetworkSettings.IPAddress}}' 6647

[root@localhost vagrant]# docker inspect -f '{{ .NetworkSettings.IPAddress}}' 90ab

[root@localhost vagrant]# docker inspect -f '{{ .NetworkSettings.IPAddress}}' db27
172.17.0.2
```

好了，分析就这么多。我们看测试一下启动的容器是否正常，先通过 `curl` 访问几次 nginx：

```
[root@localhost vagrant]# curl -s http://172.17.0.2 | head -n 5
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx on Debian!</title>
<style>
```

然后通过 busybox 容器查看访问日志：

```
[root@localhost vagrant]# docker logs -f 6647
172.17.0.1 - - [08/Sep/2016:03:29:47 +0000] "GET / HTTP/1.1" 200 867 "-" "curl/7.29.0"
172.17.0.1 - - [08/Sep/2016:03:49:57 +0000] "GET / HTTP/1.1" 200 867 "-" "curl/7.29.0"
172.17.0.1 - - [08/Sep/2016:03:50:18 +0000] "GET / HTTP/1.1" 200 867 "-" "curl/7.29.0"
172.17.0.1 - - [08/Sep/2016:03:50:23 +0000] "GET / HTTP/1.1" 200 867 "-" "curl/7.29.0"
```

随着请求的发送，可以看到日志都打印出来了。

### API 接口

前面也提到了，kubelet 自身提供了几个 HTTP API 接口，供用户查看信息。
最简单的就是 10248 的健康检查：

```
[root@localhost vagrant]# curl http://127.0.0.1:10248/healthz
ok
```

而 10255 端口比较重要，提供了 pod 和机器的信息：

```
➜  ~ http http://192.168.8.100:10255/pods
HTTP/1.1 200 OK
Content-Length: 1345
Content-Type: application/json
Date: Thu, 08 Sep 2016 04:03:38 GMT

{
    "apiVersion": "v1",
    "items": [
        {
            "metadata": {
                "annotations": {
                    "kubernetes.io/config.hash": "bf570970380d322fa28f20a8fe4b61bc",
                    "kubernetes.io/config.seen": "2016-09-01T09:50:16.379538472+02:00",
                    "kubernetes.io/config.source": "file"
                },
                "creationTimestamp": null,
                "name": "nginx-server-192.168.8.100",
                "namespace": "default",
                "selfLink": "/api/v1/pods/namespaces/nginx-server-192.168.8.100/default",
                "uid": "dae69c6f50364bf8fcf20bc67f42a416"
            },
            "spec": {
                "containers": [
                    {
                        "image": "172.16.1.41:5000/nginx",
                        "imagePullPolicy": "Always",
                        "name": "nginx-server",
                        "ports": [
                            {
                                "containerPort": 80,
                                "protocol": "TCP"
                            }
                        ],
                        "resources": {},
                        "terminationMessagePath": "/dev/termination-log",
                        "volumeMounts": [
                            {
                                "mountPath": "/var/log/nginx",
                                "name": "nginx-logs"
                            }
                        ]
                    },
                    {
                        "args": [
                            "-c",
                            "tail -f /logdir/access.log"
                        ],
                        "command": [
                            "bin/sh"
                        ],
                        "image": "172.16.1.41:5000/busybox",
                        "imagePullPolicy": "Always",
                        "name": "log-output",
                        "resources": {},
                        "terminationMessagePath": "/dev/termination-log",
                        "volumeMounts": [
                            {
                                "mountPath": "/logdir",
                                "name": "nginx-logs"
                            }
                        ]
                    }
                ],
                "dnsPolicy": "ClusterFirst",
                "nodeName": "192.168.8.100",
                "restartPolicy": "Always",
                "securityContext": {},
                "terminationGracePeriodSeconds": 30,
                "volumes": [
                    {
                        "emptyDir": {},
                        "name": "nginx-logs"
                    }
                ]
            },
            "status": {
                "conditions": [
                    {
                        "lastProbeTime": null,
                        "lastTransitionTime": "2016-09-01T07:52:36Z",
                        "status": "True",
                        "type": "PodScheduled"
                    }
                ],
                "phase": "Pending"
            }
        }
    ],
    "kind": "PodList",
    "metadata": {}
}
```

pods 列出了节点上运行的 pod 信息，包括里面有多少容器、pod 的状态、一些 metadata 等。 `/spec/` 路径返回节点的信息：

```
➜  ~ http http://192.168.8.100:10255/spec/
HTTP/1.1 200 OK
Content-Type: application/json
Date: Thu, 08 Sep 2016 04:06:53 GMT
Transfer-Encoding: chunked

{
    "boot_id": "48955926-11dd-4ad3-8bb0-2585b1c9215d",
    "cloud_provider": "Unknown",
    "cpu_frequency_khz": 2194976,
    "disk_map": {
        "253:0": {
            "major": 253,
            "minor": 0,
            "name": "dm-0",
            "scheduler": "none",
            "size": 1107296256
        },
       ...
        "8:0": {
            "major": 8,
            "minor": 0,
            "name": "sda",
            "scheduler": "cfq",
            "size": 42949672960
        },
        "8:16": {
            "major": 8,
            "minor": 16,
            "name": "sdb",
            "scheduler": "cfq",
            "size": 21474836480
        },
     ...
    },
    "filesystems": [
        {
            "capacity": 41293721600,
            "device": "/dev/mapper/centos-root",
            "inodes": 40345600,
            "type": "vfs"
        },
        {
            "capacity": 520794112,
            "device": "/dev/sda1",
            "inodes": 512000,
            "type": "vfs"
        },
        {
            "capacity": 21464350720,
            "device": "/dev/sdb",
            "inodes": 20971520,
            "type": "vfs"
        },
       ...
    ],
    "instance_id": "None",
    "instance_type": "Unknown",
    "machine_id": "b9597c4ae5f24494833d35e806e00b29",
    "memory_capacity": 514215936,
    "network_devices": [
        {
            "mac_address": "08:00:27:c0:9e:5b",
            "mtu": 1500,
            "name": "enp0s3",
            "speed": 1000
        },
        {
            "mac_address": "08:00:27:9b:5c:b5",
            "mtu": 1500,
            "name": "enp0s8",
            "speed": 1000
        }
    ],
    "num_cores": 1,
    "system_uuid": "823EB67A-057E-4EFF-AE7F-A758140CD2F7",
    "topology": [
        {
            "caches": [
                {
                    "level": 3,
                    "size": 3145728,
                    "type": "Unified"
                }
            ],
            "cores": [
                {
                    "caches": [
                        {
                            "level": 1,
                            "size": 32768,
                            "type": "Data"
                        },
                        {
                            "level": 1,
                            "size": 32768,
                            "type": "Instruction"
                        },
                        {
                            "level": 2,
                            "size": 262144,
                            "type": "Unified"
                        }
                    ],
                    "core_id": 0,
                    "thread_ids": [
                        0
                    ]
                }
            ],
            "memory": 536403968,
            "node_id": 0
        }
    ]
}
```

磁盘、网络、CPU、内存等信息，这些东西是 kubernetes 集群调度的重要依据，在集群模式中会更新到 master 节点。

### 清理工作

如果删除 `manifests/` 路径下的文件，等一段时间就会发现，kubelet 已经清空了所有的容器。

```
$ curl -s http://localhost:10255/pods
{"kind":"PodList","apiVersion":"v1","metadata":{},"items":null}
```

## 参考链接

- [Deploy Kubernetes Master Node(s)](https://coreos.com/kubernetes/docs/latest/deploy-master.html)
- [What even is a kubelet?](http://kamalmarhubi.com/blog/2015/08/27/what-even-is-a-kubelet/)
- [Kubernetes Node Documentation](http://kubernetes.io/docs/admin/node/)
- [Overview of a Pod](https://coreos.com/kubernetes/docs/latest/pods.html)
- [Checking Pod Health](http://kubernetes.io/docs/user-guide/liveness/)
