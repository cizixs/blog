---
layout: post
title: "kubernetes 权限管理"
excerpt: "kubernetes 对于访问 API 来说提供了两个步骤的安全措施：认证和授权。认证解决用户是谁的问题，授权解决用户能做什么的问题。通过合理的权限管理，能够保证系统的安全可靠。"
categories: blog
tags: [kubernetes, https, container, ABAC, RBAC, security]
comments: true
share: true
---

kubernetes 主要通过 APIServer 对外提供服务，对于这样的系统集群来说，请求访问的安全性是非常重要的考虑因素。如果不对请求加以限制，那么会导致请求被滥用，甚至被黑客攻击。

kubernetes 对于访问 API 来说提供了两个步骤的安全措施：认证和授权。认证解决用户是谁的问题，授权解决用户能做什么的问题。通过合理的权限管理，能够保证系统的安全可靠。

下图是 API 访问要经过的三个步骤，前面两个是认证和授权，第三个是 Admission Control，它也能在一定程度上提高安全性，不过更多是资源管理方面的作用，在这篇文章不会介绍。

![](https://kubernetes.io/images/docs/admin/access-control-overview.svg)

**NOTE**： 只有通过 HTTPS 访问的时候才会通过认证和授权，HTTP 不需要。

## 一. 认证（Authentication）：我是谁

认证关注的是谁发送的请求，也就是说客户端必须用某种方式揭示自己的身份信息。我们常见的认证手段是用户名和密码的方法，比如几乎所有的社交网站；现在也有很多手机应用采用指纹的方式进行认证。不管怎么说，认证的功能只有一个，提供用户的身份信息。

kubernetes 并没有完整的用户系统，因此目前认证的方式并不是统一的，而是提供了很多可以配置的认证方式供用户选择。这些认证方式包括：

### 客户端证书认证

我们知道，一般在访问服务端的时候为了安全考虑会采用 HTTPS 的方式。但其实 HTTPS 连接可以是双向的，也就是说客户端也可以提供证书给服务端。kubernetes 可以使用客户端证书来作为认证，X509 HTTPS 的证书中会包含证书所有者的身份信息，就是证书中的 `Common Name` 字段。apiserver 启动的时候通过参数 `--client-ca-file=SOMEFILE` 来配置签发客户端证书的 CA，当客户端发送证书过来的时候，apiserver 会使用 CA 进行验证，如果证书合法，就提取其中的 `Common Name` 字段作为用户名。

NOTE：关于 HTTPS、CA、证书的内容的解释超过了这篇文章的范围，感兴趣的读者可以自行搜索对应的资料了解。

### 静态密码文件认证

静态密码的方式是提前在某个文件中保存了用户名和密码的信息，然后在 apiserver 启动的时候通过参数 `--basic-auth-file=SOMEFILE` 指定文件的路径。apiserver 一旦启动，加载的用户名和密码信息就不会发生改变，任何对源文件的修改必须重启 apiserver 才能生效。

静态密码文件是 CSV 格式的文件，每行对应一个用户的信息，前面三列密码、用户名、用户 ID 是必须的，第四列是可选的组名（如果有多个组，必须用双引号）：

```
password,user,uid,"group1,group2,group3"
```

客户端在发送请求的时候需要在请求头部添加上 `Authorization` 字段，对应的值是 `Basic BASE64ENCODED(USER:PASSWORD)`。apiserver 解析出客户端提供的用户名和密码，如果和文件中的某一行匹配，就认为认证成功。

**NOTE**：这种方式很不灵活，也不安全，可以说名存实亡，不推荐使用。

### 静态 Token 文件认证

静态 Token 文件的方式很简单，事先在一个文件中写上用户的认证信息（用户名、用户 id、token、用户所在的组名），apiserver 启动通过参数 `--token-auth-file=SOMEFILE` 指定这个文件，apiserver 把这些信息加载起来。token 文件是 CSV 格式的文件，每行代表一个用户的信息，至少包含 token、用户名和用户 ID 三列，最后一列组名列表是可选的（如果有多个组必须用双引号括起来），比如：

```
token,user,uid,"group1,group2,group3"
```

token 有点像令牌，客户端不需要证明自己和 token 的关系，只要客户端提供了令牌，就认为客户端身份是合法的。这有点像古装剧中拿着皇上颁发的某个令牌行事，不管谁拿着令牌都有对应的权力。

客户端只要在请求的头部加上 `Authorization` 字段就能完成认证，对应的值是 `Bearer TOKEN`。

这种方式下，apiserver 一旦启动就不会再根据文件的内容调整内存中的数据，想要让修改的文件生效，只能重启 apiserver。**和静态密码一样，这种方法也是名存实亡，不推荐使用。**

如果配置了多个认证方式，kubernetes 会以此遍历它们（并不保证它们的先后顺序），一旦请求能通过某个认证，就算成功。

### Service Account Tokens 认证

有些情况下，我们希望在 pod 内部访问 apiserver，获取集群的信息，甚至对集群进行改动。针对这种情况，kubernetes 提供了一种特殊的认证方式：Service Account。

Service Account 是面向 namespace 的，每个 namespace 创建的时候，kubernetes 会自动在这个 namespace 下面创建一个默认的 Service Account；并且这个 Service Account 只能访问该 namespace 的资源。Service Account 和 pod、service、deployment 一样是 kubernetes 集群中的一种资源，用户也可以创建自己的 serviceaccount。

ServiceAccount 主要包含了三个内容：namespace、Token 和 CA。namespace 指定了 pod 所在的 namespace，CA 用于验证 apiserver 的证书，token 用作身份验证。它们都通过 mount 的方式保存在 pod 的文件系统中，其中 `token` 保存的路径是 `/var/run/secrets/kubernetes.io/serviceaccount/token`，是 apiserver 通过私钥签发 token 的 base64 编码后的结果；`CA` 保存的路径是 `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`，namespace 保存的路径是 `/var/run/secrets/kubernetes.io/serviceaccount/namespace`，也是用 base64 编码。

如果 token 能够通过认证，那么请求的用户名将被设置为 `system:serviceaccount:(NAMESPACE):(SERVICEACCOUNT)`，而请求的组名有两个：`system:serviceaccounts` 和 `system:serviceaccounts:(NAMESPACE)`。

关于 Service Account 的配置可以参考官方的 [Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) 文档。

### OpenID 认证

这种认证方式是通过 OAuth2 协议进行的，也就是第三方登录常用的方式。因为没有使用过，略过不表，感兴趣可以参考 OAuth2 的原理。

### Webhook Token 认证

Webhook Token 认证方式可以让用户使用自己的认证方式，用户只需要按照约定的请求格式和应答格式提供 HTTPS 服务，当用户把 Bearer Token 放到请求的头部，kubernetes 会把 token 发送给事先配置的地址进行认证，如果认证结果成功，则认为请求用户合法。

这种方式下有两个参数可以配置：

- `--authentication-token-webhook-config-file`：kubeconfig 文件说明如果访问认证服务器
- `--authentication-token-webhook-cache-ttl`：认证结果要缓存多久，默认是两分钟

这种方式下，自定义认证的请求和应答都有一定的格式，具体的规范请参考[官方文档的说明](https://kubernetes.io/docs/admin/authentication/#webhook-token-authentication)。

### Keystone 认证

[Keystone](http://docs.openstack.org/developer/keystone/) 是 openstack 提供的认证和授权组件，这个方法对于已经使用 openstack 来搭建 Iaas 平台的公司比较适用，直接使用 keystone 可以保证 Iaas 和 Caas 平台保持一致的用户体系。

kubernetes 目前对 keystone 的支持还是实验阶段，不推荐在生产系统中使用。

### 匿名请求

如果请求没有通过以上任何方式的认证，正常情况下应该是直接返回 401 错误。但是 kubernetes 还提供另外一种选择，给没有通过认证的请求一个特殊的用户名 `system:anonymous` 和组名 `system:unauthenticated`。

这样的话，可以跟下面要讲的授权结合起来，为匿名请求设置一些特殊的权限，比如只能读取当前 namespace 的 pod 信息，方便用户访问。

## 二. 授权（Authorization）：我能做什么

授权发生在认证之后，通过认证的请求就能知道 username，而授权判断这个用户是否有权限对访问的资源执行特定的动作。

还是拿社交软件做例子，当用户成功登录之后，他能操作的资源范围是固定的：查看和自己有关系的用户的数据，但是只能修改自己的数据。不难想象，你可以随意地删除或者修改其他用户的信息是一件多么恐怖的事情。

在系统软件中，用户还分成不同的角色，最常见的比如管理员。如果普通用户只能看到自己视角内的内容，那么管理员的权限则大得多，它一般能查看整个系统的数据，并且能对大部分内容做修改（包括删除）。

**控制不同用户能操作哪些内容就是授权要做的事情**。

在启动 apiserver 的时候，可以通过 `--authorization_mode` 参数来指定授权模式，目前支持的模式有：

- `AlwaysDeny`：阻止所有的请求访问，只用于测试环境
- `AlwaysAllow`：允许所有的请求访问，注意这个模式下没有对请求进行限制，只有你能确保没有恶意请求的时候使用
- `ABAC`：基于属性的访问控制
- `RBAC`：基于角色的访问控制，这是 1.6 版本之后开始推荐的授权方式
- `WebHook`：

kubernetes apiserver 根据事先定义的授权策略 来决定用户是否有权限访问。每个请求都带上了用户和资源的信息：比如发送请求的用户、请求的路径、请求的动作等，授权就是根据这些信息和授权策略进行比较，如果符合策略，则认为授权通过，否则会返回 `403 Unauthorized` 错误。

和认证一样，管理员可以配置多个授权方式，一种任何一种方式通过，就认为授权成功。所以如果配置了 `AlwaysAllow`，不管还配置了什么，请求都能直接通过授权。

kubernetes 把请求分成了两种：资源请求和非资源请求。资源请求是对 kubernetes 封装的资源实体的请求，比如 pods、nodes、services 等；非资源请求相反，是对诸如 `/api`、`/metrics`、`healthz` 等和资源无关的请求。它们两者的授权也有区别。

`AlwaysAllow` 和 `AlwaysDeny` 比较简单，我们就不介绍了，而是看看其他三种方法。

### ABAC（Attribute-Based Access Control）

ABAC 根据请求的属性来决定某个请求是否有权限，授权策略是写到文件中的，因此如果配置了该方法，apiserver 在启动的时候还要通过参数 `--authorization-policy-file=SOME_FILENAME` 告诉策略文件的位置。

这个文件每行定义了一个策略，每个策略都是 JSON 格式的内容。这个 JSON 可以包含如下的内容：

- `apiVersion`：版本号，目前是 `abac.authorization.kubernetes.io/v1beta1`
- `kind`：资源类型，必须是 `Policy`
- `spec`：具体的策略配置，这是个字典，包括这些字段：
    - `user`：用户名
    - `group`：组名。`system:authenticated` 匹配所有通过认证的请求，`system:unauthenticated` 匹配所有没有通过认证的请求
    - `apiGroup`：资源的 API group，比如 `extensions`，`*` 表示匹配所有的 API group
    - `namespace`：kubernetes 中的 namespace，比如 `kube-system`，`*` 匹配所有的 namespace
    - `resource`：资源类型，比如 `pods`，`*` 匹配所有的资源类型
    - `nonResourcePath`：非资源请求的路径，比如 `/version`或者 `/apis` 等，`*` 匹配所有的非资源路径，`/foo/*` 匹配所有 `/foo` 的子路径
    - `readonly`：布尔型，是否只读，默认为 false。如果设置为 true，则用户只有 GET、LIST 和 WATCH 权限

每个请求都有对应的属性，授权的时候会一次匹配所有的规则，如果有某个规则匹配，则认为请求有对应的权限（Read/Write）。可以看到，通配符 `*` 表示匹配任意的值，比如可以让某个用户可以访问任意资源。

来看几个例子：

#### 1. Alice 可以做任何事情
```
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "alice", "namespace": "*", "resource": "*", "apiGroup": "*"}}
```

#### 2. kubelet 可以读取所有 pods 的信息
```
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "kubelet", "namespace": "*", "resource": "pods", "readonly": true}}
```

#### 3. kubelet 可以对 events 进行任何的读写操作
```
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "kubelet", "namespace": "*", "resource": "events"}}
```

#### 4. Bob 可以读取 `projectCaribou` namespace 的所有 pods 信息

```
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "bob", "namespace": "projectCaribou", "resource": "pods", "readonly": true}}
```

#### 5. 所有人都能对所有路径发送只读请求

```
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group": "system:authenticated", "readonly": true, "nonResourcePath": "*"}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group": "system:unauthenticated", "readonly": true, "nonResourcePath": "*"}}
```

**NOTE**：ABAC 对权限的定义比较少，只有可读、可读写两种。并且策略修改后只有重启 apiserver 才能生效，因此用起来不是很方便。
    
### RBAC（Role-Based Access Control）

RBAC 是官方才 1.6 版本之后推荐使用的授权方式，因为它比较灵活，而且能够很好地实现资源隔离效果。

这种方法引入了一个重要的概念：Role，翻译成角色。所有的权限都是围绕角色进行的，也就是说角色本身会包含一系列的权限规则，表明某个角色能做哪些事情。比如管理员可以操作所有的资源，某个 namespace 的用户只能修改该 namespace的内容，或者有些角色只允许读取资源。角色和 pods、services 这些资源一样，可以通过 API 创建和删除，因此用户可以非常灵活地根据需求创建角色。

RBAC 定义了 `Role` 和 `ClusterRole`，分别对应单个 namespace 的权限和整个集群的权限，来对两者进行区分。

权限就是对某个资源可以执行什么样的操作，操作可以是 `get`、`list`、`watch`、`create`、`update`、`patch` 和 `delete`，因此粒度要比 ABAC 更细，资源就对应了 kubernetes API 管理的资源概念，比如：

```
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch", "extensions"]
  resources: ["jobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

上面定义了两个权限，分别是读取 pods 和读写 jobs 资源。

有了权限之后，最终还是要对应到用户上面，用户和角色之间可以进行绑定，绑定后的用户就能拥有对应角色的所有权限。比如我们可以添加一个管理员角色，然后把公司的某几个人设置（绑定）成管理员。一个用户可以和多个角色进行绑定，同时应用这些角色的权限。

和 Role 一样，RBAC 也有两种资源：`RoleBinding` 和 `ClusterRoleBinding`，分别对应单个 namespace 和整个集群范围。

我们来看个例子：

```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

这个 RoleBinding 就是把用户 `jane` 绑定到 `pod-reader` 这个角色。

以上就是 RBAC 最核心的概念，还有什么细节没有讲，比如怎么具体去指定某个资源/子资源、怎么做授权、ClusterRole 的一些边缘特性、kubernetes 预先创建的一些默认角色和绑定等等。RBAC 相对 ABAC 概念更多，也更复杂，想看具体的例子和更多 RBAC 的内容，可以参考[官方文档](https://kubernetes.io/docs/admin/authorization/rbac/)。

### Web Hook

这里的 webhook 和验证差不多，就是用户在外部提供 HTTPS 服务，然后配置 apiserver 调用该服务去进行授权，略过不提。

### 自定义

除了上面两个方式之外，用户也可以自己直接修改 kubernetes 代码，编写授权方式。因为 kubernetes 做了很好的封装，用户只需要实现下面的接口即可：

```
type Authorizer interface {
  Authorize(a Attributes) error
}
```

把自己编写的代码放在 `pkg/auth/authorizer/$MODULENAME` 路径就行。

## 参考资料

- [Controlling Access to the Kubernetes API：和 kubernetes API 通信的流程](https://kubernetes.io/docs/admin/accessing-the-api/)
- [Authenticating：kubernetes 认证机制](https://kubernetes.io/docs/admin/authentication/)
- [Authorization：kubernetes 授权机制](https://kubernetes.io/docs/admin/authorization/)
- [Using Admission Controllers](https://kubernetes.io/docs/admin/admission-controllers/)
- [容器编排之Kubernetes认证与授权](https://zhuanlan.zhihu.com/p/26220963)
- [Understanding Kubernetes Authentication and Authorization](http://cloudgeekz.com/1045/kubernetes-authentication-and-authorization.html)
- [Accessing the API from a Pod](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/#accessing-the-api-from-a-pod)
- [在Kubernetes Pod中使用Service Account访问API Server](http://tonybai.com/2017/03/03/access-api-server-from-a-pod-through-serviceaccount/)
- [Kubernetes集群的安全配置](http://tonybai.com/2016/11/25/the-security-settings-for-kubernetes-cluster/)
- [4S: SERVICES ACCOUNT, SECRET, SECURITY CONTEXT AND SECURITY IN KUBERNETES](http://www.sel.zju.edu.cn/?p=588)