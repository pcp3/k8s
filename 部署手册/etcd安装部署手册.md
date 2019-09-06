# Etcd安装部署手册

说明：部署Etcd集群需至少准备3个节点，每个节点互为主备，部署步骤也完全一样。该文档中所有用到的安装介质均在：

> https://github.com/pcp3/k8s/tree/master/安装介质/etcd

# 1. 安装执行文件

**Step1：拷贝etcd、etcdctl共2个文件至/usr/bin/etcd/文件夹（请提前建好该文件夹）**

**Step2：创建etcd用户组和用户。执行：**

```
groupadd etcd && useradd -c "Etcd user" -g etcd -s /sbin/nologin -r etcd
```

**Step3：拷贝etcd.service共1个文件至/usr/lib/systemd/system/文件夹。**

# 2. 修改配置文件

**Step1：拷贝etcd.conf至/etc/etcd/文件夹。按照下述说明修改：**

> *#[Member]* *该部分为本机配置*
>
> *ETCD_NAME="**etcd_node1_pcp**" #etcd**实例名称，**每个节点均不一样*
>
> *ETCD_DATA_DIR="/var/lib/etcd/data"    #etcd**数据保存目录*
>
> *ETCD_LISTEN_PEER_URLS="http://10.110.30.41:2380" #**监听URL，用于与其他节点通讯，**每个节点均不一样*
>
> *ETCD_LISTEN_CLIENT_URLS="http://10.110.30.41:2379,http://127.0.0.1:2379" #**对外提供服务的地址，**每个节点均不一样*
>
>  
>
> *#[cluster]* *该部分为集群配置*
>
> *ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.110.30.41:2380" #**该节点同伴监听地址，这个值会告诉集群中其他节点。**每个节点均不一样*
>
> *ETCD_ADVERTISE_CLIENT_URLS="http://10.110.30.41:2379" #**对外公告的该节点客户端监听地址，这个值会告诉集群中其他节点。**每个节点均不一样*
>
> *ETCD_INITIAL_CLUSTER="etcd_node1_pcp=http://10.110.30.41:2380,etcd_node2_pcp=http://10.110.30.42:2380,etcd_node2_pcp=http://10.110.30.43:2380" #**集群中所有节点的信息，格式为 node1=http://ip1:2380,node2=http://ip2:2380,… 。注意：这里的 etcd_node1_pcp 是节点的 --name 指定的名字；后面的 ip1:2380 是 --initial-advertise-peer-urls 指定的值*
>
> *ETCD_INITIAL_CLUSTER_STATE="new" #**新建集群的时候，这个值为 new ；假如已经存在的集群，这个值为 existing*
>
> *ETCD_INITIAL_CLUSTER_TOKEN="etcd-node1-pcp" #**创建集群的 token，这个值每个集群保持唯一*



**Step2：按照上述etcd.conf中ETCD_DATA_DIR的配置，创建/var/lib/etcd/data文件夹，并授予etcd用户读写权限。执行：**

```
mkdir -p /var/lib/etcd/data && chown etcd:etcd -R /var/lib/etcd/data /etc/etcd*
```

# 3. 启动并检查

**Step1：启动服务。执行：**

```
systemctl enable etcd.service && systemctl start etcd.service
```

**Step2：检查版本。执行：**

```
cd /usr/bin/etcd && ./etcdctl --version
```

# 4. 安装其它节点

**Step1：按照上述步骤，在其它节点执行etcd安装。**

**Step2：检查成员。执行：**

```
./etcdctl member list
```