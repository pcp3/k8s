# k8s节点初始化步骤

说明：安装部署kubernetes集群之前，必须先对操作系统进行初始化设置，包括：设置主机名、关闭防火墙、设置netfilter以及设置免密钥登录等。本文档详细介绍了对kubernetes集群内节点的初始化操作步骤。

# 1. 设置主机名

**Step1：按照节点规划，分别登录所有节点，修改主机名。**

```
hostnamectl set-hostname kmanager.k8s.pcp
hostnamectl set-hostname kmaster1.k8s.pcp
hostnamectl set-hostname kmaster2.k8s.pcp
hostnamectl set-hostname knode1.k8s.pcp
hostnamectl set-hostname knode2.k8s.pcp
```

**Step2：登录manager（如果没有manager则登录master1），修改hosts文件。**

```
vi /etc/hosts
```

将host映射关系添加到此文件中：

> 10.110.30.41         kmanager.k8s.pcp
>
> 10.110.30.42         kmaster1.k8s.pcp
>
> 10.110.30.43         kmaster2.k8s.pcp
>
> 10.110.30.49         knode1.k8s.pcp
>
> 10.110.30.51         knode2.k8s.pcp

**Step3：将上述hosts文件分发到其他节点。**

```
scp /etc/hosts root@kmaster1.k8s.pcp:/etc
scp /etc/hosts root@kmaster2.k8s.pcp:/etc
scp /etc/hosts root@knode1.k8s.pcp:/etc
scp /etc/hosts root@knode2.k8s.pcp:/etc
```

# 2. 关闭防火墙

**Step1：关闭防火墙。**

```
systemctl stop firewalld
systemctl disable firewalld
```

**Step2：关闭iptables。**

```
systemctl disable iptables
systemctl stop iptables
```

# **3. 关闭seLinux**

**Step1：关闭seLinux**

```
setenforce 0
```

**Step2：永久关闭seLinux。**

```
sed -i '/^SELINUX=/cSELINUX=disabled' /etc/sysconfig/selinux
```

# 4. 关闭swap分区

**Step1：刷新swap分区**

```
swapoff -a && swapon -a
```

**Step2：永久关闭，不用重启。**

```
sysctl -p
```

# 5. 设置netfilter

**Step1：创建k8s.conf。**

```
vi /etc/sysctl.d/k8s.conf
```

添加

> net.bridge.bridge-nf-call-ip6tables = 1
>
> net.bridge.bridge-nf-call-iptables = 1
>
> net.ipv4.ip_forward = 1

**Step2：使配置生效。**

```
modprobe br_netfilter
sysctl -p /etc/sysctl.d/k8s.conf
```

# 6. 打通ssh

**Step1：登录manager（如果没有manager则登录master1），生成公钥、私钥对。**

```
ssh-keygen
```

**Step2：一路回车执行，生成公钥。**

**Step3：将公钥复制到所有节点。注意：必须往本机拷贝一份。 **

```
ssh-copy-id root@kmanager.k8s.pcp
ssh-copy-id root@kmaster1.k8s.pcp
ssh-copy-id root@kmaster2.k8s.pcp
ssh-copy-id root@knode1.k8s.pcp
ssh-copy-id root@knode2.k8s.pcp
```

# 7. 设置时钟源

## 7.1 只Master节点

**Step1：安装ntp服务。**

```
yum -y install ntp
```

**Step2：修改ntp配置文件。**

```
vi /etc/ntp.conf
```

> restrict 10.110.30.254 mask 255.255.255.0 nomodify notrap 
>
> server 10.110.30.1 prefer
>
> server 127.127.1.0
>
> fudge 127.127.1.0 stratum 8

$$
*[参数说明]*
10.110.30.254 和255.255.255.0是集群所在网段的网关和子网掩码。
10.110.30.1是主时钟源IP，根据实际情况替换，prefer表示优先选择的时钟源。
$$

**Step3：从主时钟源同步时间：**

```
ntpdate 10.110.30.1
```

## 7.2 其它节点

**Step1：安装ntp服务。**

```
yum -y install ntp
```

**Step2：修改ntp配置文件。**

```
vi /etc/ntp.conf
```

> server 10.110.30.1 prefer
>
> server master.k8s.pcp

$$
[参数说明]*
master.k8s.pcp是master节点的IP。
10.110.30.1是主时钟源IP，根据实际情况替换，prefer表示优先选择的时钟源。
$$

**Step3：从主时钟源同步时间。**

```
ntpdate 10.110.30.1
```

## 7.3 启动及检查（所有节点）

**Step1：启动ntp服务。 **

```
systemctl start ntpd.service 
systemctl enable ntpd.service
```

**Step2：查看ntp状态及连通性。**

```
ntpq -p
ntpstat
```

> ’*’ 表示当前使用的时钟源，’+’ 表示这些源可作为NTP 源

