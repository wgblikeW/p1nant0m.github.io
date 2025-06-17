---
title: "[✨Exploration Series] Kubernetes Authentications & Authorization Mechanismds"
subtitle:   "[✨探索系列] kubernetes 中的认证与授权机制"
date:       2023-08-30 12:00:00
author:     "p1nant0m"
lightgallery: true
tags:
    - Kubernetes
    - Auth & Authz
---

最近在学习**云原生最佳安全实践**，其中提到了 k8s 中的认证与授权模块，为了加深理解以及有对 k8s 源码阅读的计划，便诞生了这个\[✨探索系列]。我们将从需求出发看看为什么这个系统需要设计成这个样子，这个模块以及在源码层面是何如实现的，这样的实现有什么亮点以及工程考量，不仅如此，在这个过程中我们还会着重关注其中与安全相关的细节，关注系统的全貌，重现思考安全系统的构建。

#### 🌠 我是谁？我能做什么？—— 认证与授权

认证和授权是计算机网络安全中的两个老生常谈的基本问题，它们都旨在确保只有授权用户才能访问受保护的资源。为了实现认证与授权机制，我们需要解决抛出两个问题让大家来思考一下：1）如何证明你（主体）的身份，2）如何确保你（主体）正在访问（行为）的资源（客体）是**合法**的？

对于问题（1）如何证明你（主体）的身份，经典的解决思路有：a) 提供只有你知道的秘密（口令），b) 让一个Identity Provider第三方协助你证明你的身份（OAuth），对于问题（2）如何确保你正在访问的资源是合法的，解决思路是建立访问控制模型，典型的访问控制模型有，ACL(Access-Control-List), RBAC(Role-Based-Access-Control), ABAC(Attribute-Based-Access-Control)以及NGAC(Next-Generation-Access-Control)。

对于认证而言，为了达成一个优良的用户体验（不需要用户在进行操作时反复进行认证），在实践中往往需要实现一个可以在一个时间周期内向服务器证明自身并维持可信状态的机制，这种机制需要数据信息的支撑，根据数据信息存储位置的不同，我们可以将其分为服务器侧维护以及用户侧维护。

用户侧存储凭证维持机制：Basic Token（Base64编码username:password，在服务器侧校验凭证的合法性），Bearer Token(携带用户凭证作为数据载荷，通过签名算法实现防篡改），Client Certificate（客户侧证书，由可信目标CA签发，主体拥有私钥证明自身的合法性）。

解析后的JWT示例（Bearer Token）

```json
HEADER:ALGORITHM & TOKEN TYPE
{
  "alg": "RS256",
  "kid": "VsVyOVilN85xFdtqJZMMTlkqpQA_WZgLAfg51igPox8"
}

PAYLOAD:DATA
{
  "aud": [
    "https://kubernetes.default.svc.cluster.local"
  ],
  "exp": 1667530848,
  "iat": 1667527248,
  "iss": "https://kubernetes.default.svc.cluster.local",
  "kubernetes.io": {
    "namespace": "default",
    "serviceaccount": {
      "name": "cilium",
      "uid": "302b0155-6608-4b70-be4b-f7fb2bb73beb"
    }
  },
  "nbf": 1667527248,
  "sub": "system:serviceaccount:default:cilium"
}

```

解析后的X509 Client Certificate示例，

```纯文本
Version:          3 (0x02)
Serial number:    1041774366814885730 (0x0e7520445e1bd762)
Algorithm ID:     SHA256withRSA
Validity
  Not Before:     22/08/2023 03:12:59 (dd-mm-yyyy hh:mm:ss) (230822031259Z)
  Not After:      21/08/2024 03:18:00 (dd-mm-yyyy hh:mm:ss) (240821031800Z)
Issuer
  CN = kubernetes
Subject
  O  = system:masters
  CN = kubernetes-admin
Public Key
  Algorithm:      RSA
  Length:         2048 bits
  Modulus:        bf:a4:cf:72:34:07:a7:0c:4b:db:19:2a:f7:60:10:68:
                  8b:52:db:e9:3f:24:d5:05:c5:4b:c7:f5:04:1e:01:d4:
                  2a:e5:a5:59:4e:66:6f:25:eb:9f:1b:a5:91:81:28:18:
                  7f:a3:fd:36:c5:45:e5:ea:ef:18:a0:f8:61:1a:bc:6b:
                  ee:d5:18:5e:3a:08:c0:11:d4:62:3c:40:18:6f:e6:56:
                  49:fe:01:5b:c5:56:f2:ab:4e:db:9f:96:4e:2f:e3:26:
                  56:39:4b:e8:08:7d:e3:9c:15:01:be:86:8b:ce:b4:8e:
                  7e:b7:1f:f9:98:4d:fc:13:7a:0a:f5:d1:bf:11:c7:4d:
                  11:41:d5:14:92:4a:3e:ef:c9:03:07:58:5a:49:31:df:
                  c6:21:64:b6:5e:2e:1a:f8:da:4e:1a:7e:1b:7b:48:10:
                  9c:93:05:bd:a1:9e:f8:2c:75:0a:1a:4b:c6:1c:11:95:
                  de:9a:7f:dd:12:9e:94:7a:68:07:4d:19:ed:fb:63:26:
                  aa:a4:0a:71:81:83:a0:ad:94:e8:1e:7b:cc:dc:39:3c:
                  ca:ae:bf:ba:50:57:37:87:a4:e9:b7:b1:1c:27:0a:18:
                  18:51:5c:55:28:87:0a:18:12:35:99:14:c4:10:53:37:
                  26:2d:fa:69:7e:b2:d8:8d:f1:dc:76:83:01:a7:81:e5
  Exponent:       65537 (0x10001)
Certificate Signature
  Algorithm:      SHA256withRSA
  Signature:      92:d7:99:22:91:2f:f3:ef:38:25:b4:ad:ea:7d:2c:b7:
                  a1:bb:cf:a4:eb:a0:f8:86:b9:ea:2b:e8:39:0e:0d:3c:
                  27:72:19:33:a1:69:e6:72:6b:f0:82:9e:f1:44:e6:0a:
                  1d:9b:55:58:e6:79:75:84:af:65:d2:5c:30:8d:1f:0e:
                  25:18:62:5a:9f:7e:0a:20:79:da:29:be:bc:16:3d:d2:
                  4e:fa:0f:13:c2:9a:5a:bb:98:32:55:ae:a3:bf:36:db:
                  00:fc:db:6a:ef:7e:09:f2:8a:aa:22:ca:90:db:b8:15:
                  70:5d:b2:1b:b6:b9:37:93:52:8c:d9:92:0f:38:97:bb:
                  93:c7:90:19:e7:c1:41:29:2f:7f:b8:02:03:07:43:47:
                  0a:6d:bb:f5:1f:e5:42:e2:22:19:6e:dc:e1:8e:fb:49:
                  c2:43:69:1a:85:1e:9e:99:9e:e3:85:83:91:15:e3:9c:
                  b8:17:7d:83:65:db:c8:03:ee:4a:1e:75:98:66:2f:e7:
                  86:40:ea:71:17:b8:34:bc:a4:b9:48:4b:1f:a3:42:c1:
                  a2:e7:70:b7:0d:25:7d:af:16:40:63:49:fd:d1:85:67:
                  1f:2b:5a:4e:25:cc:b9:a6:99:4b:5c:03:c0:ad:3c:e8:
                  97:d1:6b:29:74:5e:5b:a0:99:f1:09:69:f9:d8:59:aa

Extensions
  keyUsage CRITICAL:
    digitalSignature,keyEncipherment
  extKeyUsage :
    clientAuth
  basicConstraints CRITICAL:
    {}
  authorityKeyIdentifier :
    kid=d2003d36621e663f31e8b164d33f6dbf9433c371
```

服务器侧存储凭证维持机制：SessionID（不携带用户凭证，每一个ID绑定一个用户凭证并由服务器在后台进行管理）。

两者实现方法各有千秋，比如，使用Session ID需要在服务器端维护会话状态，而Bearer Token可以直接将与用户凭证相关的信息存储在客户端上，不需要服务器端进行会话管理，只需要在进行相关操作时对Token进行签名的校验，确认其合法性即可。

在本节初我们提到了**合法**这个词，这个词带有一点儿权威第三方下定义的定义的意味，我们姑且把这个权威第三方称作管理者。**授权便是由管理者决定主体的哪些行为是合法的过程**。

管理者提供对主体（用户）进行身份认证的机制并为其所管理的客体（资源）制定访问规则，确保只有满足规则之下的主体才能够对这些客体（资源）进行相关的行为操作。这样的规则一般由一个三元组组成，它们分别是主体、客体以及行为，下面展示了一些根据不同AC模型制定的规则。

```纯文本
|      Subject      |       Object                      |         Action
        Alice              /api/v1/*                                GET
        Admin                 *                                      *
        Guest            /api/v1/hello                              GET    
        
     [Bod, Danny]    [/api/v1/stock/*, /api/v1/db/*]           GET, POST, DELETE
        Watcher           [/api/v1/stock/*]                         GET
        Dannis              Watcher
        
      tag:age:>22         /api/v1/secrets/*                         GET              
```

正确施行的授权机制保障了：1）个人数据和隐私不被滥用或泄露，2）确保系统和数据的安全，以防止未经授权的访问或攻击，3）确保每个用户或应用程序只能访问其所需的资源或数据，而不是超出其权限的内容，4）确保授权人获得适当的许可证或权限，以使用受限制的资源或数据。

k8s中的认证与鉴权机制也万变不离其宗，我们在下一章节中将看到k8s上的认证与鉴权。

#### 🌄 Overviews In Kubernetes Authenticating Model

实验环境：

Kind 搭建 k8s 集群，HTTPie 作为 HTTP Client，k9s 可视化命令行工具管理 k8s 集群。下面给出的是本实验所使用到的 config.yaml，用于配置 kind 创建 k8s 集群。

```yaml
# config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: k8s-exp
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
```

k8s 中的认证策略 （[Authentication Strategies](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)）

-   **X509 Client Certs**
-   Static Token File
-   Bootstrap Tokens
-   **Service Account Tokens**
-   OpenID Connect Tokens
-   Webhook Token Authentication
-   Authenticating Proxy

> It is assumed that a cluster-independent service manages normal users in the following ways:
>
> -   an administrator distributing private keys
> -   a user store like Keystone or Google Accounts
> -   a file with a list of usernames and passwords
>
> \*\*In this regard, \*\****Kubernetes does not have objects which represent normal user accounts.***

- 使用 **kubelet**(credential in `.kube` directory) 的身份与 **apiserver** 进行通信 （Client Certs | mTLS）
- 使用 system:serviceaccount:default:cilium 身份与 **apiserver** 进行通信 （Bearer Token）

1.  Create ClusterRole \[Cross Namespaces]

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: secret-reader
rules:
- apiGroups: [""]
  #
  # at the HTTP level, the name of the resource for accessing Secret
  # objects is "secrets"
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

2. Create ServiceAccount

```bash
kubectl create serviceaccount cilium
```

3. Create ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
# This role binding allows "dave" to read secrets in the "development" namespace.
# You need to already have a ClusterRole named "secret-reader".
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: ServiceAccount
  name: cilium # Name is case sensitive
  namespace: "default"
  apiGroup: ""
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

4. Create token for *system:serviceaccount:default:cilium*

```bash
kubectl create token cilium
```

> ServiceAccounts Using certificate to communicate with apiserver? Basic Steps (Guessing):
>
> &#x20; 1.Create Private Key and generate certificate.
>
> &#x20; 2.Create X509 certificates for client \[/CN='system:serviceaccount:default:cilium']
>
> &#x20; 3.Generate csr and request CA to sign the certificate
>
> &#x20; 4.Building mTLS communication to the apiserver with the certificate&#x20;

{{< image src="https://image.p1nant0m.xyz/20250617223917418.png" caption="Relationship Between kubernetes Componets related to RBAC" src_s="https://image.p1nant0m.xyz/20250617223917418.png" src_l="https://image.p1nant0m.xyz/20250617223917418.png" >}} 

通过上述操作，位于 \[default] 命名空间下的 \[cilium] **ServiceAccount** 便可以对集群环境中的所有 **Secrets** 进行 get、watch、list 操作了。

```bash
# request
$http -A bearer -a eyJhbGciOiJSUzI1NiIsImtpZCI6IlZzVnlPVmlsTjg1eEZkdHFKWk1NVGxrcXBRQV9XWmdMQWZnNTFpZ1BveDgifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjY3NTMwODQ4LCJpYXQiOjE2Njc1MjcyNDgsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0Iiwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImNpbGl1bSIsInVpZCI6IjMwMmIwMTU1LTY2MDgtNGI3MC1iZTRiLWY3ZmIyYmI3M2JlYiJ9fSwibmJmIjoxNjY3NTI3MjQ4LCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpjaWxpdW0ifQ.T8Hekf2GGRBfBpKeKZsgyTYJvCEsRbG-cIydmi9lQR7YEzT-_2zhyIto8cDphnbhGSAnyjz8Ia2LruW2FAHawKMZdkTUbigCsnE0adUcri1-lHbV4SYI-PSXGvktnDsEpaauqDVa3AmlTn_o35UnqQDMNed5-5DIFWm3AcqWwZthriSuEH9j8HiaMRtV8-4YuwcVZyQ6bANUGkOsYd_IOHnt1ZT9fJ2mJnWvOjjO7xjcUjcD38DtClwRdTD7pY_BDONw1TashsjXcYNIeXk7n5fBHkENgUktS_3HyRP-whkX2Td-SghZTTYYd7cLhMVYKZL1-kBdi0Q9R4LhEjalvA https://127.0.0.1:44257/api/v1/namespaces/default/pods\?limit\=500
```

```bash
# response
{
    "apiVersion": "v1",
    "items": [],
    "kind": "SecretList",
    "metadata": {
        "resourceVersion": "72253"
    }
}
```

我们使用 `kubectl` 命令来观察向 **apiserver** 请求创建一个 **secret** 时，所构造的 HTTP 报文格式，

```bash
kubectl create secret generic top-secret --from-file=topsecret.txt --namespace=default -v=8

{"kind":"Secret","apiVersion":"v1","metadata":{"name":"top-secret","namespace":"default","creationTimestamp":null},"data":{"topsecret.txt":"W1RPUCBTRUNSRVRdCg=="}}

POST https://127.0.0.1:44257/api/v1/namespaces/default/secrets
```

相关的 RESTful API [细节可以查看](<https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/secret-v1/#create-create-a-secret>)：

我们使用 *system:serviceaccount:default:cilium* 身份向 **apiserver** 发起请求，

```bash
echo -n "{"kind":"Secret","apiVersion":"v1","metadata":{"name":"top-secret","namespace":"default","creationTimestamp":null},"data":{"topsecret.txt":"W1RPUCBTRUNSRVRdCg=="}}" | http -A bearer -a eyJhbGciOiJSUzI1NiIsImtpZCI6IlZzVnlPVmlsTjg1eEZkdHFKWk1NVGxrcXBRQV9XWmdMQWZnNTFpZ1BveDgifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjY3NTMwODQ4LCJpYXQiOjE2Njc1MjcyNDgsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0Iiwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImNpbGl1bSIsInVpZCI6IjMwMmIwMTU1LTY2MDgtNGI3MC1iZTRiLWY3ZmIyYmI3M2JlYiJ9fSwibmJmIjoxNjY3NTI3MjQ4LCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpjaWxpdW0ifQ.T8Hekf2GGRBfBpKeKZsgyTYJvCEsRbG-cIydmi9lQR7YEzT-_2zhyIto8cDphnbhGSAnyjz8Ia2LruW2FAHawKMZdkTUbigCsnE0adUcri1-lHbV4SYI-PSXGvktnDsEpaauqDVa3AmlTn_o35UnqQDMNed5-5DIFWm3AcqWwZthriSuEH9j8HiaMRtV8-4YuwcVZyQ6bANUGkOsYd_IOHnt1ZT9fJ2mJnWvOjjO7xjcUjcD38DtClwRdTD7pY_BDONw1TashsjXcYNIeXk7n5fBHkENgUktS_3HyRP-whkX2Td-SghZTTYYd7cLhMVYKZL1-kBdi0Q9R4LhEjalvA  POST https://127.0.0.1:44257/api/v1/namespaces/default/secrets
```

不过注意到我们并没有 `create secret` 的权限，我们若尝试 `create secret` 则会提示 401 错误码【未授权的操作】，

```bash
xxxxxxxxxx HTTP/1.1 401 UnauthorizedAudit-Id: bdb8f7a5-84be-42b9-8d9b-8445c3bd3b85Cache-Control: no-cache, privateContent-Length: 129Content-Type: application/jsonDate: Fri, 04 Nov 2022 11:09:15 GMT​{    "apiVersion": "v1",    "code": 401,    "kind": "Status",    "message": "Unauthorized",    "metadata": {},    "reason": "Unauthorized",    "status": "Failure"}
```

**kubectl** 使用 `/path/to/home/.kube/config` 文件中的用户私钥与证书来与 **apiserver** 进行通信，在使用 kind 创建集群时，kind-control-plane 中的 `/etc/kubernetes` 目录下有 `admin.conf` 文件，其中的私钥与证书正是在 **kubectl** 中使用到的，kind 为我们准备好了一切！

-   k9s 下键入 groups 命令，可以看到名为 system:master 的 **ClusterRoleBinding** 资源对象，其将 cluster-admin【ClusterRole】绑定到了 system:masters【Group】上。
-   在 `/path/to/home/.kube/config` 中的数据经过了 Base64 编码，解码后可以得到 X509 制式的证书。

> Kubernetes determines the username from the common name field in the 'subject' of the cert (e.g., "/CN=bob")

综上，我们可以使用 Openssl 解析相关证书，查看其中的 **Common Name Field**，也就是我们在 k8s 语义下所说的用户名（注意：在 k8s 中，用户所属的组在证书中用 **Organization Field** 表示）。在 `config` 文件所在目录下执行，

```bash
$cat config | grep -oP "(?<=client-certificate-data: ).*" | base64 -d >> kubernetes-admin.crt && openssl x509 -in kubernetes-admin.crt -text && rm kubernetes-admin.crt
```

截取证书中的相关内容，我们可以看到，

```text
 Subject: O = system:masters, CN = kubernetes-admin
```

其中，该证书代表的“用户名”为 kubernetes-admin，所属的"用户组"为 system:masters，根据我们上述的分析可以知道 **system:masters Group** 与 **cluster-admin ClusterRole** 绑定，**system:masters** 中的成员具有操作集群中所有资源的权限。

我们知道，k8s 中并没有专门用于维护用户账户的组件，其获取用户账户的途径是在通信的过程中通过客户端证书以及 token 解析后对应的账户字段信息表示当前跟 **apiserver** 进行通信的对象，其信任通信对等方是因为其提供的认证凭据（客户端证书由 **apiserver** 启动时提供的 CA 所签发、token 校验合法）合法。就拿 `system:serviceaccount:default:cilium` 的 token (JWT) 来看，其在解析后具有如下信息，

```json
{
  "aud": [
    "https://kubernetes.default.svc.cluster.local"
  ],
  "exp": 1667530848,
  "iat": 1667527248,
  "iss": "https://kubernetes.default.svc.cluster.local",
  "kubernetes.io": {
    "namespace": "default",
    "serviceaccount": {
      "name": "cilium",
      "uid": "302b0155-6608-4b70-be4b-f7fb2bb73beb"
    }
  },
  "nbf": 1667527248,
  "sub": "system:serviceaccount:default:cilium"
}
```

**"sub"** 字段便标识了带有该 token 请求的主体身份，**apiserver** 根据该身份以及其请求的相应资源经过授权模块以及准入控制器处理后向请求主体返回响应结果。

> 🌟 **Tips:**
> 关于用户身份主体的命名规则："**system:serviceaccount**" 代表k8s中的资源对象，"**default**" 代表命名空间(namespace)，"**cilium**" 代表serviceaccount的命名

ServiceAccount-RoleBinding-ClusterRole 操作仍然会限制在 **ServiceAccount** 所处的 Namespace 下，最小权限交集集合。

ServiceAccount-ClusterRoleBinding-ClusterRole 则能够在整个集群范围中执行 Role 所授权的操作。

Role & ClusterRole || RoleBinding & ClusterRoleBinding 【subject:(User, Group, ServiceAccount) object(roleRef):(Role)】

使用 kubectl 与 apiServer 进行交互执行命令时，发出请求所使用的身份是 User 实体。在我的实验环境中，则是用户 kind-kind 其在 Cluster 中的 **Role** 是 **cluster-amdin**，被赋予了能够对所有资源执行任意操作的权限。kubectl 加载的配置文件默认位于 `/path/to/home/.kube` 下，其中存有 kind-kind 用户的证书以及私钥信息，用于证明其身份，下面展示的是用户kind-kind的证书，其中Subject-O字段代表其所在的用户组，Subject-CN代表其在k8s维护资源中的用户名。

```纯文本
Version:          3 (0x02)
Serial number:    1041774366814885730 (0x0e7520445e1bd762)
Algorithm ID:     SHA256withRSA
Validity
  Not Before:     22/08/2023 03:12:59 (dd-mm-yyyy hh:mm:ss) (230822031259Z)
  Not After:      21/08/2024 03:18:00 (dd-mm-yyyy hh:mm:ss) (240821031800Z)
Issuer
  CN = kubernetes
Subject
  O  = system:masters
  CN = kubernetes-admin
Public Key
  Algorithm:      RSA
  Length:         2048 bits
  Modulus:        bf:a4:cf:72:34:07:a7:0c:4b:db:19:2a:f7:60:10:68:
                  8b:52:db:e9:3f:24:d5:05:c5:4b:c7:f5:04:1e:01:d4:
                  2a:e5:a5:59:4e:66:6f:25:eb:9f:1b:a5:91:81:28:18:
                  7f:a3:fd:36:c5:45:e5:ea:ef:18:a0:f8:61:1a:bc:6b:
                  ee:d5:18:5e:3a:08:c0:11:d4:62:3c:40:18:6f:e6:56:
                  49:fe:01:5b:c5:56:f2:ab:4e:db:9f:96:4e:2f:e3:26:
                  56:39:4b:e8:08:7d:e3:9c:15:01:be:86:8b:ce:b4:8e:
                  7e:b7:1f:f9:98:4d:fc:13:7a:0a:f5:d1:bf:11:c7:4d:
                  11:41:d5:14:92:4a:3e:ef:c9:03:07:58:5a:49:31:df:
                  c6:21:64:b6:5e:2e:1a:f8:da:4e:1a:7e:1b:7b:48:10:
                  9c:93:05:bd:a1:9e:f8:2c:75:0a:1a:4b:c6:1c:11:95:
                  de:9a:7f:dd:12:9e:94:7a:68:07:4d:19:ed:fb:63:26:
                  aa:a4:0a:71:81:83:a0:ad:94:e8:1e:7b:cc:dc:39:3c:
                  ca:ae:bf:ba:50:57:37:87:a4:e9:b7:b1:1c:27:0a:18:
                  18:51:5c:55:28:87:0a:18:12:35:99:14:c4:10:53:37:
                  26:2d:fa:69:7e:b2:d8:8d:f1:dc:76:83:01:a7:81:e5
  Exponent:       65537 (0x10001)
Certificate Signature
  Algorithm:      SHA256withRSA
  Signature:      92:d7:99:22:91:2f:f3:ef:38:25:b4:ad:ea:7d:2c:b7:
                  a1:bb:cf:a4:eb:a0:f8:86:b9:ea:2b:e8:39:0e:0d:3c:
                  27:72:19:33:a1:69:e6:72:6b:f0:82:9e:f1:44:e6:0a:
                  1d:9b:55:58:e6:79:75:84:af:65:d2:5c:30:8d:1f:0e:
                  25:18:62:5a:9f:7e:0a:20:79:da:29:be:bc:16:3d:d2:
                  4e:fa:0f:13:c2:9a:5a:bb:98:32:55:ae:a3:bf:36:db:
                  00:fc:db:6a:ef:7e:09:f2:8a:aa:22:ca:90:db:b8:15:
                  70:5d:b2:1b:b6:b9:37:93:52:8c:d9:92:0f:38:97:bb:
                  93:c7:90:19:e7:c1:41:29:2f:7f:b8:02:03:07:43:47:
                  0a:6d:bb:f5:1f:e5:42:e2:22:19:6e:dc:e1:8e:fb:49:
                  c2:43:69:1a:85:1e:9e:99:9e:e3:85:83:91:15:e3:9c:
                  b8:17:7d:83:65:db:c8:03:ee:4a:1e:75:98:66:2f:e7:
                  86:40:ea:71:17:b8:34:bc:a4:b9:48:4b:1f:a3:42:c1:
                  a2:e7:70:b7:0d:25:7d:af:16:40:63:49:fd:d1:85:67:
                  1f:2b:5a:4e:25:cc:b9:a6:99:4b:5c:03:c0:ad:3c:e8:
                  97:d1:6b:29:74:5e:5b:a0:99:f1:09:69:f9:d8:59:aa

Extensions
  keyUsage CRITICAL:
    digitalSignature,keyEncipherment
  extKeyUsage :
    clientAuth
  basicConstraints CRITICAL:
    {}
  authorityKeyIdentifier :
    kid=d2003d36621e663f31e8b164d33f6dbf9433c371

```

#### 🌠 Security Issues Related to Super-Privilege Serviceaccount

[1] [GKE-Autopilot Vulnerabilities](<https://unit42.paloaltonetworks.com/gke-autopilot-vulnerabilities/>)

[2] [Kubernetes Privilege Escalation Excessive Permissions in Popular Platforms](<https://www.paloaltonetworks.com/apps/pan/public/downloadResource?pagePath=/content/pan/en_US/resources/whitepapers/kubernetes-privilege-escalation-excessive-permissions-in-popular-platforms>)
