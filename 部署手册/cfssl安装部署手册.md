# CFSSL安装部署手册

说明：该文档中所有用到的安装文件均在：

> https://github.com/pcp3/k8s/tree/master/安装介质/cfssl/

## 1 组件说明

CFSSL是CloudFlare开源的一款PKI/TLS工具。 CFSSL 包含一个命令行工具和一个用于签名、验证并且捆绑TLS证书的 HTTP API 服务。 使用Go语言编写。该工具用來建立 TLS Certificates。只安装到一个节点即可。

根据认证对象可以将证书分成三类：服务器证书server cert，客户端证书client cert，对等证书peer cert(表示既是server cert又是client cert)。

- client certificate： 用于服务端认证客户端,例如etcdctl、etcd proxy客户端。
- server certificate: 服务端使用，客户端以此验证服务端身份。
- peer certificate: 双向证书，用于etcd集群成员间通信。

> 附：在kubernetes 集群中需要的证书种类如下：
>
> etcd 节点需要标识自己服务的server cert，也需要client cert与etcd集群其他节点交互，当然可以分别指定2个证书，也可以使用一个对等证书
>
> master 节点需要标识 apiserver服务的server cert，也需要client cert连接etcd集群，这里也使用一个对等证书
>
> kubectl calico kube-proxy 只需要client cert，因此证书请求中 hosts 字段可以为空。
>
> kubelet证书比较特殊，不是手动生成，它由node节点TLS BootStrap向apiserver请求，由master节点的controller-manager 自动签发，包含一个client cert 和一个server cert。
>

## 2 安装步骤

### 2.1 制作服务器端证书

**Step1：拷贝cfssl、cfssljson共2个文件到/usr/local/bin/目录下。**

**Step2：权限设置。执行：**

```
chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson
```

**Step3：在ssl安装节点创建/etc/ssl/文件夹，执行：**		

```
mkdir -p /etc/ssl && cd /etc/ssl
```

**Step4：拷贝ca-config.json与ca-csr.json共2个文件至/etc/ssl/文件夹下。**

**Step5：生成CA证书。执行：**

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

### 2.2 制作客户端证书

以下步骤以制作etcd客户端证书为例。其它类型客户端证书制作方式类似。

**Step1：拷贝etcd-csr.json至/etc/ssl/文件夹下。将该文件中IP地址改为etcd集群中各节点的IP地址：**

> *{*
>
>   *"CN": "etcd",*
>
>   *"hosts": [*
>
> ​    *"127.0.0.1",*
>
> ​    *"10.110.30.41",*
>
> ​    *"10.110.30.42",*
>
> ​    *"10.110.30.43"*
>
>   *],*
>
>   *"key": {*
>
> ​    *"algo": "rsa",*
>
> ​    *"size": 2048*
>
>   *},*
>
>   *"names": [*
>
> ​    *{*
>
> ​      *"C": "CN",*
>
> ​      *"ST": "jinan",*
>
> ​      *"L": "jinan",*
>
> ​      *"O": "etcd",*
>
> ​      *"OU": "System"*
>
> ​    *}*
>
>   *]*
>
> *}*

**Step2：生成etcd证书和私钥。执行：**

```
cfssl gencert \

  -ca=ca.pem \

  -ca-key=ca-key.pem \

  -config=ca-config.json \

  -profile=etcd \

  etcd-csr.json | cfssljson -bare etcd
```

**Step3：删除不必要的json文件。执行：**

```
rm -rf *.json
```

### 2.3 检查及颁发证书

**Step1：确认/etc/ssl/有一下6个文件即成功。**

```
ls /etc/ssl
```

> etcd-ca.csr、etcd-ca-key.pem、etcd-ca.pem、etcd.csr、etcd-key.pem、etcd.pem

**Step2：将上述6个文件分别拷贝至其它节点。**

```
scp -r /etc/etcd* root@10.110.30.41:/etc/
scp -r /etc/etcd* root@10.110.30.42:/etc/
scp -r /etc/etcd* root@10.110.30.43:/etc/
```

