1.为Flannel生成证书
```
[root@k8s-master1-130:/usr/local/src/ssl]# vim flanneld-csr.json
{
  "CN": "flanneld",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

2.生成证书
```
[root@k8s-master1-130:/usr/local/src/ssl]# cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
   -ca-key=/opt/kubernetes/ssl/ca-key.pem \
   -config=/opt/kubernetes/ssl/ca-config.json \
   -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld
```
3.分发证书
```
[root@k8s-master1-130:/usr/local/src/ssl]# cp flanneld*.pem /opt/kubernetes/ssl/
[root@k8s-master1-130:/usr/local/src/ssl]# scp flanneld*.pem 192.168.0.131:/opt/kubernetes/ssl/
[root@k8s-master1-130:/usr/local/src/ssl]# scp flanneld*.pem 192.168.0.132:/opt/kubernetes/ssl/
[root@k8s-master1-130:/usr/local/src/ssl]# scp flanneld*.pem 192.168.0.133:/opt/kubernetes/ssl/
[root@k8s-master1-130:/usr/local/src/ssl]# scp flanneld*.pem 192.168.0.134:/opt/kubernetes/ssl/
[root@k8s-master1-130:/usr/local/src/ssl]# scp flanneld*.pem 192.168.0.135:/opt/kubernetes/ssl/
```

4.下载Flannel软件包
```
[root@k8s-master1-130 ~]# cd /usr/local/src
# wget
 https://github.com/coreos/flannel/releases/download/v0.10.0/flannel-v0.10.0-linux-amd64.tar.gz
[root@k8s-master1-130:/usr/local/src]# tar zxf flannel-v0.10.0-linux-amd64.tar.gz
[root@k8s-master1-130:/usr/local/src]# cp flanneld mk-docker-opts.sh /opt/kubernetes/bin/
复制到linux-node2节点
[root@k8s-master1-130:/usr/local/src]# scp flanneld mk-docker-opts.sh 192.168.0.131:/opt/kubernetes/bin/
[root@k8s-master1-130:/usr/local/src]# scp flanneld mk-docker-opts.sh 192.168.0.132:/opt/kubernetes/bin/
[root@k8s-master1-130:/usr/local/src]# scp flanneld mk-docker-opts.sh 192.168.0.133:/opt/kubernetes/bin/
[root@k8s-master1-130:/usr/local/src]# scp flanneld mk-docker-opts.sh 192.168.0.134:/opt/kubernetes/bin/
[root@k8s-master1-130:/usr/local/src]# scp flanneld mk-docker-opts.sh 192.168.0.135:/opt/kubernetes/bin/
复制对应脚本到/opt/kubernetes/bin目录下。
[root@k8s-master1-130 ~]# cd /usr/local/src/kubernetes/cluster/centos/node/bin/
[root@k8s-master1-130 bin]# cp remove-docker0.sh /opt/kubernetes/bin/
[root@k8s-master1-130 bin]# scp remove-docker0.sh 192.168.0.131:/opt/kubernetes/bin/
[root@k8s-master1-130 bin]# scp remove-docker0.sh 192.168.0.132:/opt/kubernetes/bin/
[root@k8s-master1-130 bin]# scp remove-docker0.sh 192.168.0.133:/opt/kubernetes/bin/
[root@k8s-master1-130 bin]# scp remove-docker0.sh 192.168.0.134:/opt/kubernetes/bin/
[root@k8s-master1-130 bin]# scp remove-docker0.sh 192.168.0.135:/opt/kubernetes/bin/
```

5.配置Flannel
```
[root@k8s-master1-130 ~]# vim /opt/kubernetes/cfg/flannel
FLANNEL_ETCD="-etcd-endpoints=https://192.168.0.130:2379,https://192.168.0.131:2379,https://192.168.0.132:2379"
FLANNEL_ETCD_KEY="-etcd-prefix=/kubernetes/network"
FLANNEL_ETCD_CAFILE="--etcd-cafile=/opt/kubernetes/ssl/ca.pem"
FLANNEL_ETCD_CERTFILE="--etcd-certfile=/opt/kubernetes/ssl/flanneld.pem"
FLANNEL_ETCD_KEYFILE="--etcd-keyfile=/opt/kubernetes/ssl/flanneld-key.pem"
复制配置到其它节点上
[root@k8s-master1-130 ~]# scp /opt/kubernetes/cfg/flannel 192.168.0.131:/opt/kubernetes/cfg/
[root@k8s-master1-130 ~]# scp /opt/kubernetes/cfg/flannel 192.168.0.132:/opt/kubernetes/cfg/
[root@k8s-master1-130 ~]# scp /opt/kubernetes/cfg/flannel 192.168.0.133:/opt/kubernetes/cfg/
[root@k8s-master1-130 ~]# scp /opt/kubernetes/cfg/flannel 192.168.0.134:/opt/kubernetes/cfg/
[root@k8s-master1-130 ~]# scp /opt/kubernetes/cfg/flannel 192.168.0.135:/opt/kubernetes/cfg/
```

6.设置Flannel系统服务
```
[root@k8s-master1-130 ~]# vim /etc/systemd/system/flannel.service
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
Before=docker.service

[Service]
EnvironmentFile=-/opt/kubernetes/cfg/flannel
ExecStartPre=/opt/kubernetes/bin/remove-docker0.sh
ExecStart=/opt/kubernetes/bin/flanneld ${FLANNEL_ETCD} ${FLANNEL_ETCD_KEY} ${FLANNEL_ETCD_CAFILE} ${FLANNEL_ETCD_CERTFILE} ${FLANNEL_ETCD_KEYFILE}
ExecStartPost=/opt/kubernetes/bin/mk-docker-opts.sh -d /run/flannel/docker

Type=notify

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
复制系统服务脚本到其它节点上
# scp /etc/systemd/system/flannel.service 192.168.0.131:/etc/systemd/system/
# scp /etc/systemd/system/flannel.service 192.168.0.132:/etc/systemd/system/
# scp /etc/systemd/system/flannel.service 192.168.0.133:/etc/systemd/system/
# scp /etc/systemd/system/flannel.service 192.168.0.134:/etc/systemd/system/
# scp /etc/systemd/system/flannel.service 192.168.0.135:/etc/systemd/system/
```

## Flannel CNI集成
下载CNI插件
```
https://github.com/containernetworking/plugins/releases
wget https://github.com/containernetworking/plugins/releases/download/v0.7.1/cni-plugins-amd64-v0.7.1.tgz
[root@k8s-master1-130 ~]# mkdir /opt/kubernetes/bin/cni (所有节点都创建)
[root@k8s-master1-130 src]# tar zxf cni-plugins-amd64-v0.7.1.tgz -C /opt/kubernetes/bin/cni
# scp -r /opt/kubernetes/bin/cni/* 192.168.0.131:/opt/kubernetes/bin/cni/
# scp -r /opt/kubernetes/bin/cni/* 192.168.0.132:/opt/kubernetes/bin/cni/
# scp -r /opt/kubernetes/bin/cni/* 192.168.0.133:/opt/kubernetes/bin/cni/
# scp -r /opt/kubernetes/bin/cni/* 192.168.0.134:/opt/kubernetes/bin/cni/
# scp -r /opt/kubernetes/bin/cni/* 192.168.0.135:/opt/kubernetes/bin/cni/
```
创建Etcd的key
### 向etcd中创建POD的网段。
```
/opt/kubernetes/bin/etcdctl --ca-file /opt/kubernetes/ssl/ca.pem --cert-file /opt/kubernetes/ssl/flanneld.pem --key-file /opt/kubernetes/ssl/flanneld-key.pem \
      --no-sync -C https://192.168.0.130:2379,https://192.168.0.131:2379,https://192.168.0.132:2379 \
mk /kubernetes/network/config '{ "Network": "10.2.0.0/16", "Backend": { "Type": "vxlan", "VNI": 1 }}' >/dev/null 2>&1
```
启动flannel
```
[root@k8s-master1-130 ~]# systemctl daemon-reload
[root@k8s-master1-130 ~]# systemctl enable flannel
[root@k8s-master1-130 ~]# chmod +x /opt/kubernetes/bin/*
[root@k8s-master1-130 ~]# systemctl start flannel         ### 启动时就会运行删除docker0的脚本。
```
查看服务状态
```
[root@k8s-master1-130 ~]# systemctl status flannel
[root@k8s-master1-130 ~]# ifconfig   ### 查看每个机器的网络flannel1.1的网络是否正常。
```

## 配置Docker使用Flannel
```
[root@k8s-master1-130 ~]# vim /etc/systemd/system/docker.service
[Unit] #在Unit下面修改After和增加Requires
After=network-online.target firewalld.service flannel.service
Wants=network-online.target
Requires=flannel.service

[Service] #增加EnvironmentFile=-/run/flannel/docker
Type=notify
EnvironmentFile=-/run/flannel/docker
ExecStart=/usr/bin/dockerd $DOCKER_OPTS
```
将配置复制到另外两个阶段
```
# scp /etc/systemd/system/docker.service 192.168.0.131:/etc/systemd/system/
# scp /etc/systemd/system/docker.service 192.168.0.132:/etc/systemd/system/
# scp /etc/systemd/system/docker.service 192.168.0.133:/etc/systemd/system/
# scp /etc/systemd/system/docker.service 192.168.0.134:/etc/systemd/system/
# scp /etc/systemd/system/docker.service 192.168.0.135:/etc/systemd/system/
```
重启Docker
```
[root@k8s-master1-130 ~]# systemctl daemon-reload
[root@k8s-master1-130 ~]# systemctl restart docker
```
查看docker0地址是否和flanel1.1在同网段
```
root@k8s-master1-130:/usr/local/src# ifconfig 
docker0   Link encap:Ethernet  HWaddr 02:42:1f:23:ae:78  
          inet addr:10.2.52.1  Bcast:10.2.52.255  Mask:255.255.255.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

ens33     Link encap:Ethernet  HWaddr 00:0c:29:b4:a3:92  
          inet addr:192.168.0.130  Bcast:192.168.0.255  Mask:255.255.255.0
          inet6 addr: fe80::20c:29ff:feb4:a392/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:1880413 errors:0 dropped:0 overruns:0 frame:0
          TX packets:2213644 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:1418629648 (1.4 GB)  TX bytes:3518682363 (3.5 GB)

flannel.1 Link encap:Ethernet  HWaddr ae:c2:97:b6:27:bd  
          inet addr:10.2.52.0  Bcast:0.0.0.0  Mask:255.255.255.255
          inet6 addr: fe80::acc2:97ff:feb6:27bd/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:8 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```
每个节点的docker0都要和flanel1.1在同网段才对。flanel1.1作为网桥，docker0作为网关。
