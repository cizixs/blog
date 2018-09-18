---
layout: post
title: "salt api 配置和使用"
excerpt: "本文主要介绍了 salt API 的配置和使用，以及 API 性能的简单测试"
categories: 程序技术
tags: [salt, API]
comments: true
share: true
---



##准备工作

### 安装
salt 使用 CherryPy 来实现 restful 的 api，供外部的程序调用。

根据[官方的文档](http://docs.saltstack.com/en/latest/topics/installation/ubuntu.html)配置 salt repo：

    http://ppa.launchpad.net/saltstack/salt/ubuntu/

然后使用命令 `sudo apt-get update && sudo apt-get install salt-api -y` 安装 salt-api。

### 添加用户
salt-api 使用 [eauth 验证系统](http://docs.saltstack.com/en/latest/topics/eauth/index.html)（使用api所在机器的账户进行验证），最好单独为 salt-api 添加一个账号：

    useradd -M salt
    passwd salt

记住账户的密码，在下面的 api 登录中会用到。

### 配置文件

安全方面的考虑，官方文档建议使用 https 进行加密通信（如果想使用 http 协议，忽略这部分内容，在配置文件里加上 `disable_ssl: True`）。这时候需要[生成自签名的证书](https://www.openssl.org/docs/HOWTO/certificates.txt)（如果你有通用的证书，可以直接使用）：

    # 生成 key
    # openssl genrsa -out /etc/ssl/private/key.pem 4096
    # 生成证书
    # openssl req -new -x509 -key /etc/ssl/private/key.pem -out /etc/ssl/private/cert.pem -days 1826
     

建议为 salt-api 单独创建配置文件，`/etc/salt/master` 默认会导入 `master.d/*.conf`，所以只要 `touch /etc/salt/master.d/salt-api.conf` 就可以了。文件的内容如下：

    external_auth:
      pam:
        salt:
          - .*
      
    rest_cherrypy:
      port: 8080
      host: 0.0.0.0     
      # disable_ssl: True   ## 如果不需要 https，直接使用 http ，可以去掉本行最前面的注释
      ssl_crt: /etc/ssl/private/cert.pem
      ssl_key: /etc/ssl/private/key.pem

然后重启 salt-master 和 salt-api 服务：

    sudo service salt-master restart
    sudo service salt-api restart


## API 验证
至此，salt-api 已经可以使用了。在使用之前还有最后一步工作要做——获取 token，也就是登录。还记得上面的 salt 账号和密码吧：

    curl -ki https://127.0.0.1:8080/login -H "Accept: application/json" \
     -d username='salt' \
     -d password='salt_pass' \
     -d eauth='pam'

返回的结果：

    {
    "return": [
        {
            "eauth": "pam", 
            "expire": 1418415113.831753, 
            "perms": [
                ".*"
            ], 
            "start": 1418371913.831752, 
            "token": "f17847776f6ac228f41cd51ab545d8c1021dfa98", 
            "user": "salt"
        }
    ]
}

拿到 token 之后，有两种使用方式：

1. 基于 cookie 的 session 会话，比如浏览器或者 [requests.Session](http://docs.python-requests.org/en/latest/user/advanced/#session-objects)。
2. 在 http 请求的 header 加上 `X-Auth-Token: f17847776f6ac228f41cd51ab545d8c1021dfa98`。

## API 使用

使用上面的 token 在 minion 上执行命令非常简单：

    curl -i http://localhost:8080/ -H "Accept: application/json"  -H "X-Auth-Token: b363bf8b7a34c6a5db6719d745e32df38329a43e" -d client='local' -d tgt='minion-id1' -d fun="cmd.run" -d arg="uname -a"
    
返回结果：

    {"return": [{"ubuntu": "Linux ubuntu 3.5.0-23-generic #35~precise1-Ubuntu SMP Fri Jan 25 17:13:26 UTC 2013 x86_64 x86_64 x86_64 GNU/Linux"}]}

salt-api 除了直接调用 salt-client 之外，还提供了某些模块的快捷访问：

1. /minions：管理 minion
2. /jobs： jobs 查看，管理
3. /run：绕过 token，直接通过用户名和密码访问 salt 服务
4. /events
5. /hook
6. /keys
7. ...

请访问参考文档的 [A REST API FOR SALT](http://docs.saltstack.com/en/latest/ref/netapi/all/salt.netapi.rest_cherrypy.html)查看详细的内容。

## API 性能

1. salt 执行命令是并行的（使用 gevent 测试结果: 并行结果 `max(37s, sleeptime)` ），而 salt-api 测试下来不是并发的(尽管文档里有 thread_pool, socket_queue-size 的配置)。
2. chrerrypy [本身的性能很高](https://cherrypy.readthedocs.org/en/3.2.6/appendix/cherrypyspeed.html)，基本在 ms 级（和 ab 测试结果一致）。
3. 脚本运行时间 hp teaming: 7s， network setting： 5s

### ab 测试 cherrypy API 性能

request number | request concurrency | requests/s | time/req | mean time  |  total 
-----       | ---------     | --------- |   ------------ | --------------- | ----
1           |   1           |   263.78  |   3.79    | 3.79     |  0.004
10          |    1          |   412.52  |    2.42    | 2.42    |  0.024
10          |   2           |   379.08  |    2.638  |  5.276   |  0.026
10          |   5           |   375.45  |   2.663   | 13.317   |  0.027
50          |   1           |   431.38  |   2.318   | 2.318    |  0.138
50          |   5           |   363.45  |   2.751   | 13.757   |  0.116
100         |   10          |   364.79  |   2.741   | 27.413   |  0.274
1000        |   1           |   445.05  |   2.247   | 2.247    |  2.247 
1000        |   10          |   346.47  |   2.886   | 28.862   |  2.886 

### API 时间过长的解决方案

+ rewrite salt-API
+ trigger a salt action using API, return a job id. 




## 参考文档
+ [salt-api 官方文档](https://salt-api.readthedocs.org/en/latest/)
+ [Basic usage of Salt-API](https://coderwall.com/p/jcxcba/basic-usage-of-salt-api)
+ [Salt-API安装配置及使用](http://pengyao.org/salt-api-deploy-and-use.html)
+ [Using salt-api to integrate SaltStack with other services](http://bencane.com/2014/07/17/integrating-saltstack-with-other-services-via-salt-api/)
+ [A REST API FOR SALT](http://docs.saltstack.com/en/latest/ref/netapi/all/salt.netapi.rest_cherrypy.html)
