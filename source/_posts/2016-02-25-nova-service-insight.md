---
layout: post
title: "nova compute service 启动过程"
excerpt: "openstack 各个组件都是作为服务启动的，这篇文章就以 nova-compute 为例，介绍了 openstack 内部是如何实现 service 的。"
categories: blog
tags: [nova, openstack, service, compute, messaging, eventlet]
comments: true
share: true
---

+ OS 版本：Ubuntu 12.04
+ openstack 版本：icehouse

我们知道 nova-compute 服务可以通过:

    service nova-compute status/start/stop/restart

进行管理，我们可以从 `/etc/init/nova-compute.conf` 文件看到，背后调用的是 `/usr/bin/nova-compute` 可执行文件。这个文件的内容比较简单，主要调用 `nova.cmd.compute:main` 函数运行，我们就看一下这个 main 函数的内容：

    def main():
        objects.register_all()
        config.parse_args(sys.argv)
        logging.setup('nova')
        utils.monkey_patch()

        gmr.TextGuruMeditation.setup_autorun(version)

        if not CONF.conductor.use_local:
            block_db_access()
            objects_base.NovaObject.indirection_api = \
                conductor_rpcapi.ConductorAPI()

        server = service.Service.create(binary='nova-compute',
                                        topic=CONF.compute_topic,
                                        db_allowed=CONF.conductor.use_local)
        service.serve(server)
        service.wait()

上面代码最主要的功能就是：创建一个 service，然后启动它。每个 service 都有下面几个重要的参数：

+ host：服务所在机器，默认是机器的 hostname
+ binary：服务名，默认是可执行文件的 basename，比如这里的 nova-compute
+ topic：rpc 使用到的 topic，默认是 `binary` 去掉前面 `nova-` 字符的结果，比如这里是 `compute`
+ manager：每个 service 都有一个 manager，默认从 `CONF.<topic>_manager` 中获取，这里是 `nova.compute.manager.ComputeManager`
+ report_interval：每个 service 都要定时报告自己的状态，以便 controller 可以知晓。默认是 `CONF.report_interval`。

`service.serve` 会调用 `service.launch(server, workers=workers)`来执行：

    def launch(service, workers=None):
        if workers:
            launcher = ProcessLauncher()
            launcher.launch_service(service, workers=workers)
        else:
            launcher = ServiceLauncher()
            launcher.launch_service(service)
        return launcher

这里根据是否有 worker 来决定是开新的进程还是使用 eventlet。
service 启动的时候调用 `server.start` 函数，内容如下：

    def start(self):
        verstr = version.version_string_with_package()
        LOG.audit(_('Starting %(topic)s node (version %(version)s)'),
                  {'topic': self.topic, 'version': verstr})
        self.basic_config_check()
        self.manager.init_host()
        self.model_disconnected = False
        ctxt = context.get_admin_context()
        try:
            self.service_ref = self.conductor_api.service_get_by_args(ctxt,
                    self.host, self.binary)
            self.service_id = self.service_ref['id']
        except exception.NotFound:
            try:
                self.service_ref = self._create_service_ref(ctxt)
            except (exception.ServiceTopicExists,
                    exception.ServiceBinaryExists):
                # NOTE(danms): If we race to create a record with a sibling
                # worker, don't fail here.
                self.service_ref = self.conductor_api.service_get_by_args(ctxt,
                    self.host, self.binary)

        self.manager.pre_start_hook()

        if self.backdoor_port is not None:
            self.manager.backdoor_port = self.backdoor_port

        LOG.debug(_("Creating RPC server for service %s") % self.topic)

        target = messaging.Target(topic=self.topic, server=self.host)

        endpoints = [
            self.manager,
            baserpc.BaseRPCAPI(self.manager.service_name, self.backdoor_port)
        ]
        endpoints.extend(self.manager.additional_endpoints)

        serializer = objects_base.NovaObjectSerializer()

        self.rpcserver = rpc.get_server(target, endpoints, serializer)
        self.rpcserver.start()

        self.manager.post_start_hook()

        LOG.debug(_("Join ServiceGroup membership for this service %s")
                  % self.topic)
        # Add service to the ServiceGroup membership group.
        self.servicegroup_api.join(self.host, self.topic, self)

        if self.periodic_enable:
            if self.periodic_fuzzy_delay:
                initial_delay = random.randint(0, self.periodic_fuzzy_delay)
            else:
                initial_delay = None

            self.tg.add_dynamic_timer(self.periodic_tasks,
                                     initial_delay=initial_delay,
                                     periodic_interval_max=
                                        self.periodic_interval_max)

这里主要做了几件事：

1. 调用 manager 的 init_host 进行初始化
2. 创建 rpcserver 监听对应的队列
3. 把自己加入到 servicegroup 中
4. 如果有周期性任务，就初始化一个 timer，在这个服务中定期 run 这些任务

至此，nova-compute 服务就完全启动了，其他服务的启动和这个类似，主要是 Manager 类不一样。当然 nova-api 服务使用的是 WSGIService，启动的是 wsgi server。


## 参考资料

+ [OpenStack之Nova分析——Nova Scheduler服务启动](http://blog.csdn.net/qiuhan0314/article/details/43196149)
+ [nova service 启动](http://www.choudan.net/2013/08/09/Nova-Service%E5%90%AF%E5%8A%A8.html)
