# 微职位：自动化运维工程师


# SaltStack自动化部署Kubernetes
- SaltStack自动化部署Kubernetes v1.10.0版本（支持TLS 双向认证、RBAC 授权、Flannel网络、ETCD集群、Kuber-Proxy使用LVS等）。
- 手动部署步骤请看最下面的文档（目前已更新至v.1.10.1）。

## 版本明细：Release-v1.0

- 测试通过系统：Ubuntu-16.04
- salt-ssh: 2017.7.4
- Kubernetes： v1.10.0
- Etcd: v3.3.1
- Docker: 18.03.1-ce
- Flannel： v0.10.0
- CNI-Plugins： v0.7.0
建议部署节点：最少三个节点，请配置好主机名解析（必备）

## 架构介绍

1. 使用Salt Grains进行角色定义，增加灵活性。
2. 使用Salt Pillar进行配置项管理，保证安全性。
3. 使用Salt SSH执行状态，不需要安装Agent，保证通用性。
4. 使用Kubernetes当前稳定版本v1.9.3，保证稳定性。

# 使用手册
## 0.系统初始化

1. 设置主机名！！！
2. 设置/etc/hosts保证主机名能够解析
3. 关闭SELinux和防火墙

## 1.设置部署节点到其它所有节点的SSH免密码登录（包括本机）
```
[root@k8s-master1-130 ~]# ssh-keygen -t rsa
[root@k8s-master1-130 ~]# ssh-copy-id linux-node1
[root@k8s-master1-130 ~]# ssh-copy-id linux-node2
[root@k8s-master1-130 ~]# ssh-copy-id linux-node3
```

## 2.安装Salt-SSH并克隆本项目代码。

2.1 安装Salt SSH
```
[root@k8s-master1-130 ~]# yum install https://repo.saltstack.com/yum/redhat/salt-repo-latest-2.el7.noarch.rpm 
[root@k8s-master1-130 ~]# yum install -y salt-ssh
```

2.2 获取本项目代码，并放置在/srv目录
```
[root@k8s-master1-130 ~]# git clone https://github.com/unixhot/salt-kubernetes.git
[root@k8s-master1-130 srv]# cd salt-kubernetes/
[root@k8s-master1-130 srv]# mv * /srv/
[root@k8s-master1-130 srv]# cd /srv/
[root@k8s-master1-130 srv]# cp roster /etc/salt/roster
[root@k8s-master1-130 srv]# cp master /etc/salt/master
```

2.4 下载二进制文件，也可以自行官方下载，为了方便国内用户访问，请在百度云盘下载。
下载完成后，将文件移动到/srv/salt/k8s/目录下，并解压
Kubernetes二进制文件下载地址： https://pan.baidu.com/s/1WaLSRMMh-qF_MzBbnEufkA

```
[root@k8s-master1-130 ~]# cd /srv/salt/k8s/
[root@k8s-master1-130 k8s]# unzip k8s-v1.9.3.zip 
[root@k8s-master1-130 k8s]# ls -l files/
total 0
drwxr-xr-x 2 root root  94 Mar 28 00:33 cfssl-1.2
drwxrwxr-x 2 root root 195 Mar 27 23:15 cni-plugins-amd64-v0.7.0
drwxr-xr-x 2 root root  33 Mar 28 00:33 etcd-v3.3.1-linux-amd64
drwxr-xr-x 2 root root  47 Mar 28 12:05 flannel-v0.10.0-linux-amd64
drwxr-xr-x 3 root root  17 Mar 28 00:47 k8s-v1.9.3
```

## 3.Salt SSH管理的机器以及角色分配

- k8s-role: 用来设置K8S的角色
- etcd-role: 用来设置etcd的角色，如果只需要部署一个etcd，只需要在一台机器上设置即可
- etcd-name: 如果对一台机器设置了etcd-role就必须设置etcd-name

```
[root@k8s-master1-130 ~]# vim /etc/salt/roster 
k8s-master1-130:
  host: 192.168.0.130
  user: root
  priv: /root/.ssh/id_rsa
  minion_opts:
    grains:
      k8s-role: master
      etcd-role: node
      etcd-name: etcd-node1

k8s-node1-131:
  host: 192.168.0.131
  user: root
  priv: /root/.ssh/id_rsa
  minion_opts:
    grains:
      k8s-role: node
      etcd-role: node
      etcd-name: etcd-node2

k8s-node1-132:
  host: 192.168.0.132
  user: root
  priv: /root/.ssh/id_rsa
  minion_opts:
    grains:
      k8s-role: node
      etcd-role: node
      etcd-name: etcd-node3
      
k8s-node1-133:
  host: 192.168.0.133
  user: root
  priv: /root/.ssh/id_rsa
  minion_opts:
    grains:
      k8s-role: node
      etcd-role: node

k8s-node1-134:
  host: 192.168.0.134
  user: root
  priv: /root/.ssh/id_rsa
  minion_opts:
    grains:
      k8s-role: node
      etcd-role: node

k8s-node1-135:
  host: 192.168.0.135
  user: root
  priv: /root/.ssh/id_rsa
  minion_opts:
    grains:
      k8s-role: node
      etcd-role: node
```

## 4.修改对应的配置参数，本项目使用Salt Pillar保存配置
```
[root@k8s-master1-130 ~]# vim /srv/pillar/k8s.sls
#设置Master的IP地址(必须修改)
MASTER_IP: "192.168.0.130"

#设置ETCD集群访问地址（必须修改）
ETCD_ENDPOINTS: "https://192.168.0.130:2379,https://192.168.0.131:2379,https://192.168.0.131:2379"

#设置ETCD集群初始化列表（必须修改）
ETCD_CLUSTER: "etcd-node1=https://192.168.0.130:2380,etcd-node2=https://192.168.0.131:2380,etcd-node3=https://192.168.0.132:2380"

#通过Grains FQDN自动获取本机IP地址，请注意保证主机名解析到本机IP地址
NODE_IP: {{ grains['fqdn_ip4'][0] }}

#设置BOOTSTARP的TOKEN，可以自己生成
BOOTSTRAP_TOKEN: "ad6d5bb607a186796d8861557df0d17f"

#配置Service IP地址段
SERVICE_CIDR: "10.1.0.0/16"

#Kubernetes服务 IP (从 SERVICE_CIDR 中预分配)
CLUSTER_KUBERNETES_SVC_IP: "10.1.0.1"

#Kubernetes DNS 服务 IP (从 SERVICE_CIDR 中预分配)
CLUSTER_DNS_SVC_IP: "10.1.0.2"

#设置Node Port的端口范围
NODE_PORT_RANGE: "20000-40000"

#设置POD的IP地址段
POD_CIDR: "10.2.0.0/16"

#设置集群的DNS域名
CLUSTER_DNS_DOMAIN: "cluster.local."

```

## 5.执行SaltStack状态
```
测试Salt SSH联通性
[root@k8s-master1-130 ~]# salt-ssh '*' test.ping

执行高级状态，会根据定义的角色再对应的机器部署对应的服务

5.1 部署Etcd，由于Etcd是基础组建，需要先部署，目标为部署etcd的节点。
[root@k8s-master1-130 ~]# salt-ssh -L 'k8s-master1-130,k8s-node1-131,k8s-node2-132,k8s-node3-133,k8s-node4-134,k8s-node5-135' state.sls k8s.etcd

5.2 部署K8S集群
[root@k8s-master1-130 ~]# salt-ssh '*' state.highstate
```
由于包比较大，这里执行时间较长，5分钟+，如果执行有失败可以再次执行即可！

## 6.测试Kubernetes安装（请新打开一个窗口，保证环境变量生效！）
```
[root@k8s-master1-130 ~]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
[root@k8s-master1-130~]# kubectl get node
NAME            STATUS    ROLES     AGE       VERSION
192.168.0.131   Ready     <none>    2h        v1.10.1
192.168.0.132   Ready     <none>    2h        v1.10.1
192.168.0.133   Ready     <none>    2h        v1.10.1
192.168.0.134   Ready     <none>    2h        v1.10.1
192.168.0.135   Ready     <none>    2h        v1.10.1
```
## 7.测试Kubernetes集群和Flannel网络
```
[root@k8s-master1-130 ~]# kubectl run net-test --image=alpine --replicas=2 sleep 360000
deployment "net-test" created
需要等待拉取镜像，可能稍有的慢，请等待。
root@k8s-master1-130:~/yaml# kubectl get pods -o wide
NAME                                READY     STATUS              RESTARTS   AGE       IP          NODE
net-test-5767cb94df-pnt7c           1/1       Running             0          1h        10.2.31.2   192.168.0.135
net-test-5767cb94df-vm6gk           1/1       Running             0          1h        10.2.97.2   192.168.0.133

测试联通性
[root@k8s-master1-130 ~]# ping -c 1 10.2.31.2
PING 10.2.31.2 (10.2.31.2) 56(84) bytes of data.
64 bytes from 10.2.31.2: icmp_seq=1 ttl=61 time=3.11 ms

[root@linux-node1 ~]# ping -c 1 10.2.97.2
PING 10.2.97.2 (10.2.97.2) 56(84) bytes of data.
64 bytes from 10.2.97.2: icmp_seq=1 ttl=61 time=1.23 ms

```
## 7.如何新增Kubernetes节点

- 1.设置SSH无密码登录
- 2.在/etc/salt/roster里面，增加对应的机器
- 3.执行SaltStack状态salt-ssh '*' state.highstate。
```
[root@k8s-master1-130 ~]# vim /etc/salt/roster 
k8s-node6-136:
  host: 192.168.0.136
  user: root
  priv: /root/.ssh/id_rsa
  minion_opts:
    grains:
      k8s-role: node
[root@k8s-master1-130 ~]# salt-ssh '*' state.highstate
```

注意：不要相信自己，要相信电脑！！！

# 手动部署

- [系统初始化](docs/init.md)
- [CA证书制作](docs/ca.md)
- [ETCD集群部署](docs/etcd-install.md)
- [Master节点部署](docs/master.md)
- [Node节点部署](docs/node.md)
- [Flannel网络部署](docs/flannel.md)
- [创建第一个K8S应用](docs/app.md)
- [CoreDNS和Dashboard部署](docs/dashboard.md)
  
