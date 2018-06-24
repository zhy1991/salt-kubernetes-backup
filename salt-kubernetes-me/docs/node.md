## 部署kubelet

1.二进制包准备
将软件包从k8s-master1-130复制到k8s-node-131-135中去。
```
[root@k8s-master1-130 ~]# cd /usr/local/src/kubernetes/server/bin/
[root@k8s-master1-130 bin]# cp kubelet kube-proxy /opt/kubernetes/bin/
[root@k8s-master1-130 bin]# scp kubelet kube-proxy 192.168.0.131:/opt/kubernetes/bin/
[root@k8s-master1-130 bin]# scp kubelet kube-proxy 192.168.0.132:/opt/kubernetes/bin/
[root@k8s-master1-130 bin]# scp kubelet kube-proxy 192.168.0.133:/opt/kubernetes/bin/
[root@k8s-master1-130 bin]# scp kubelet kube-proxy 192.168.0.134:/opt/kubernetes/bin/
[root@k8s-master1-130 bin]# scp kubelet kube-proxy 192.168.0.135:/opt/kubernetes/bin/
```

2.创建角色绑定
```
[root@k8s-master1-130:/usr/local/src/ssl]# kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
clusterrolebinding "kubelet-bootstrap" created
### kubelet启动时会向api-server发送TSL bootstrap请求，所以要将bootstrap token设置成对应的角色，这样kubectl才有权限创建这个请求。kubelet启动时会向api-server发送请求动态获取证书。
```

3.创建 kubelet bootstrapping kubeconfig 文件
设置集群参数
```
[root@k8s-master1-130:/usr/local/src/ssl]# kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://192.168.0.130:6443 \
   --kubeconfig=bootstrap.kubeconfig
Cluster "kubernetes" set.
```

设置客户端认证参数
```
[root@k8s-master1-130:/usr/local/src/ssl]# kubectl config set-credentials kubelet-bootstrap \
   --token=ad6d5bb607a186796d8861557df0d17f \
   --kubeconfig=bootstrap.kubeconfig   
User "kubelet-bootstrap" set.
### 注意token要和api-server配置文件中启动参数中的--token-auth-file=/opt/kubernetes/ssl/bootstrap-token.csv文件中的token一致。
```

设置上下文参数
```
[root@k8s-master1-130:/usr/local/src/ssl]# kubectl config set-context default \
   --cluster=kubernetes \
   --user=kubelet-bootstrap \
   --kubeconfig=bootstrap.kubeconfig
Context "default" created.
```

选择默认上下文
```
[root@k8s-master1-130:/usr/local/src/ssl]# kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
Switched to context "default".
[root@k8s-master1-130:/usr/local/src/ssl]# cat bootstrap.kubeconfig   ### 上面几步操作所生成的文件
[root@k8s-master1-130:/usr/local/src/ssl]# cp bootstrap.kubeconfig /opt/kubernetes/cfg
[root@k8s-master1-130:/usr/local/src/ssl]# scp bootstrap.kubeconfig 192.168.0.131:/opt/kubernetes/cfg
[root@k8s-master1-130:/usr/local/src/ssl]# scp bootstrap.kubeconfig 192.168.0.132:/opt/kubernetes/cfg
[root@k8s-master1-130:/usr/local/src/ssl]# scp bootstrap.kubeconfig 192.168.0.133:/opt/kubernetes/cfg
[root@k8s-master1-130:/usr/local/src/ssl]# scp bootstrap.kubeconfig 192.168.0.134:/opt/kubernetes/cfg
[root@k8s-master1-130:/usr/local/src/ssl]# scp bootstrap.kubeconfig 192.168.0.135:/opt/kubernetes/cfg
```

部署kubelet（所有节点，master节点可以不配置。因为也不给master安装kubelet服务）
1.设置CNI支持，k8s网络接口的插件。
```
[root@k8s-master1-130 ~]# mkdir -p /etc/cni/net.d
[root@k8s-master1-130 ~]# vim /etc/cni/net.d/10-default.conf
{
        "name": "flannel",
        "type": "flannel",
        "delegate": {
            "bridge": "docker0",
            "isDefaultGateway": true,
            "mtu": 1400
        }
}


```

2.创建kubelet目录，用于存放kubelet的数据。
```
[root@k8s-master1-130 ~]# mkdir /var/lib/kubelet
```

3.创建kubelet服务配置
```
[root@k8s-master1-130 ~]# vim /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/opt/kubernetes/bin/kubelet \
  --address=192.168.0.130 \
  --hostname-override=192.168.0.130 \
  --pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.0 \
  --experimental-bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \
  --kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \
  --cert-dir=/opt/kubernetes/ssl \
  --network-plugin=cni \
  --cni-conf-dir=/etc/cni/net.d \
  --cni-bin-dir=/opt/kubernetes/bin/cni \
  --cluster-dns=10.1.0.2 \
  --cluster-domain=cluster.local. \
  --hairpin-mode hairpin-veth \
  --allow-privileged=true \
  --fail-swap-on=false \
  --logtostderr=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log
Restart=on-failure
RestartSec=5
复制都所有node节点，然后修改配置文件中的IP地址为每个节点本机的地址。
[root@k8s-master1-130  ~]# scp /etc/systemd/system/kubelet.service 192.168.0.131:/etc/systemd/system/kubelet.service
[root@k8s-master1-130  ~]# scp /etc/systemd/system/kubelet.service 192.168.0.132:/etc/systemd/system/kubelet.service
[root@k8s-master1-130  ~]# scp /etc/systemd/system/kubelet.service 192.168.0.133:/etc/systemd/system/kubelet.service
[root@k8s-master1-130  ~]# scp /etc/systemd/system/kubelet.service 192.168.0.134:/etc/systemd/system/kubelet.service
[root@k8s-master1-130  ~]# scp /etc/systemd/system/kubelet.service 192.168.0.135:/etc/systemd/system/kubelet.service
```

4.启动Kubelet
```
[root@k8s-master1-130 ~]# systemctl daemon-reload   ### master节点的kubelet不启动。
[root@k8s-master1-130 ~]# systemctl enable kubelet
[root@k8s-master1-130 ~]# systemctl start kubelet
```

5.查看服务状态
```
[root@k8s-master1-130 kubernetes]# systemctl status kubelet
```

6.查看csr请求
注意是在master上执行。
```
[root@k8s-master1-130 ~]# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-0_w5F1FM_la_SeGiu3Y5xELRpYUjjT2icIFk9gO9KOU   1m        kubelet-bootstrap   Pending
如果忘记修改就启动了服务，kubelet会上报到master，可以先停掉kubelt服务然后使用下面的命令删除上报信息。在重新启动node节点的kubelet服务。
[root@linux-node1 ~]#kubectl delete csr $(kubectl get csr | awk -F" " '{print $1}')
```

7.批准kubelet 的 TLS 证书请求
```
[root@k8s-master1-130 ~]# kubectl get csr|grep 'Pending' | awk 'NR>0{print $1}'| xargs kubectl certificate approve
批准后在node节点可以查看到kubelet在api-server那里请求来的证书文件kubelet-client.crt。
root@k8s-node1-131:~# ll /opt/kubernetes/ssl/
total 56
drwxr-xr-x 2 root root 4096 May 27 06:40 ./
drwxr-xr-x 6 root root 4096 May 27 03:20 ../
-rw-r--r-- 1 root root  290 May 27 03:58 ca-config.json
-rw-r--r-- 1 root root 1001 May 27 03:58 ca.csr
-rw------- 1 root root 1679 May 27 03:58 ca-key.pem
-rw-r--r-- 1 root root 1359 May 27 03:58 ca.pem
-rw------- 1 root root 1679 May 27 04:38 etcd-key.pem
-rw-r--r-- 1 root root 1436 May 27 04:38 etcd.pem
-rw-r--r-- 1 root root 1046 May 27 06:40 kubelet-client.crt
-rw------- 1 root root  227 May 27 06:35 kubelet-client.key
-rw-r--r-- 1 root root 2185 May 27 06:27 kubelet.crt
-rw------- 1 root root 1675 May 27 06:27 kubelet.key
-rw------- 1 root root 1675 May 27 05:01 kubernetes-key.pem
-rw-r--r-- 1 root root 1610 May 27 05:01 kubernetes.pem
```
执行完毕后，查看节点状态已经是Ready的状态了
[root@k8s-master1-130 ssl]#  kubectl get node
NAME            STATUS    ROLES     AGE       VERSION
192.168.0.131   Ready     <none>    26s       v1.10.1
192.168.0.132   Ready     <none>    26s       v1.10.1
192.168.0.133   Ready     <none>    26s       v1.10.1
192.168.0.134   Ready     <none>    25s       v1.10.1
192.168.0.135   Ready     <none>    26s       v1.10.1

## 部署Kubernetes Proxy（所有node节点，如果想在master节点也能ping通pod地址和curl通service IP的话需要在master节点也部署kube-proxy服务。）
1.配置kube-proxy使用LVS（老版的k8s仅支持iptables做转发，新版支持LVS做转发。）
```
[root@k8s-master1-130 ~]# apt-get install -y ipvsadm ipset conntrack
```

2.创建 kube-proxy 证书请求
```
[root@k8s-master1-130 ~]# cd /usr/local/src/ssl/
[root@k8s-master1-130:/usr/local/src/ssl]# vim kube-proxy-csr.json
{
  "CN": "system:kube-proxy",
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
   
3.生成证书
```
[root@k8s-master1-130:/usr/local/src/ssl]# cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
   -ca-key=/opt/kubernetes/ssl/ca-key.pem \
   -config=/opt/kubernetes/ssl/ca-config.json \
   -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
```
4.分发证书到所有Node节点
```
[root@k8s-master1-130:/usr/local/src/ssl]# cp kube-proxy*.pem /opt/kubernetes/ssl/
[root@k8s-master1-130:/usr/local/src/ssl]# scp kube-proxy*.pem 192.168.0.131:/opt/kubernetes/ssl/
[root@k8s-master1-130:/usr/local/src/ssl]# scp kube-proxy*.pem 192.168.0.132:/opt/kubernetes/ssl/
[root@k8s-master1-130:/usr/local/src/ssl]# scp kube-proxy*.pem 192.168.0.133:/opt/kubernetes/ssl/
[root@k8s-master1-130:/usr/local/src/ssl]# scp kube-proxy*.pem 192.168.0.134:/opt/kubernetes/ssl/
[root@k8s-master1-130:/usr/local/src/ssl]# scp kube-proxy*.pem 192.168.0.135:/opt/kubernetes/ssl/
```

5.创建kube-proxy配置文件
```
[root@k8s-master1-130:/usr/local/src/ssl]# kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://192.168.0.130:6443 \
   --kubeconfig=kube-proxy.kubeconfig
Cluster "kubernetes" set.
### 生成一个名为kube-proxy.kubeconfig的文件。

[root@k8s-master1-130:/usr/local/src/ssl]# kubectl config set-credentials kube-proxy \
   --client-certificate=/opt/kubernetes/ssl/kube-proxy.pem \
   --client-key=/opt/kubernetes/ssl/kube-proxy-key.pem \
   --embed-certs=true \
   --kubeconfig=kube-proxy.kubeconfig
User "kube-proxy" set.

[root@k8s-master1-130:/usr/local/src/ssl]# kubectl config set-context default \
   --cluster=kubernetes \
   --user=kube-proxy \
   --kubeconfig=kube-proxy.kubeconfig
Context "default" created.

[root@k8s-master1-130:/usr/local/src/ssl]# kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
Switched to context "default".
```
6.分发kubeconfig配置文件
```
[root@k8s-master1-130:/usr/local/src/ssl]# cp kube-proxy.kubeconfig /opt/kubernetes/cfg/
[root@k8s-master1-130:/usr/local/src/ssl]# scp kube-proxy.kubeconfig 192.168.0.131:/opt/kubernetes/cfg/
[root@k8s-master1-130:/usr/local/src/ssl]# scp kube-proxy.kubeconfig 192.168.0.132:/opt/kubernetes/cfg/
[root@k8s-master1-130:/usr/local/src/ssl]# scp kube-proxy.kubeconfig 192.168.0.133:/opt/kubernetes/cfg/
[root@k8s-master1-130:/usr/local/src/ssl]# scp kube-proxy.kubeconfig 192.168.0.134:/opt/kubernetes/cfg/
[root@k8s-master1-130:/usr/local/src/ssl]# scp kube-proxy.kubeconfig 192.168.0.135:/opt/kubernetes/cfg/
```

7.创建kube-proxy服务配置(所有安装kube-proxy服务的主机)
```
[root@k8s-master1-130 ~]# mkdir /var/lib/kube-proxy

[root@k8s-master1-130 ~]# vim /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/opt/kubernetes/bin/kube-proxy \
  --bind-address=192.168.0.130 \
  --hostname-override=192.168.0.130 \
  --kubeconfig=/opt/kubernetes/cfg/kube-proxy.kubeconfig \
--masquerade-all \
  --feature-gates=SupportIPVSProxyMode=true \
  --proxy-mode=ipvs \
  --ipvs-min-sync-period=5s \
  --ipvs-sync-period=5s \
  --ipvs-scheduler=rr \
  --logtostderr=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log

Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
8.拷贝至其他节点，修改配置文件中的IP地址为node本机地址。
```
[root@k8s-master1-130 ~]# scp /etc/systemd/system/kube-proxy.service 192.168.0.131:/etc/systemd/system/kube-proxy.service
[root@k8s-master1-130 ~]# scp /etc/systemd/system/kube-proxy.service 192.168.0.132:/etc/systemd/system/kube-proxy.service
[root@k8s-master1-130 ~]# scp /etc/systemd/system/kube-proxy.service 192.168.0.133:/etc/systemd/system/kube-proxy.service
[root@k8s-master1-130 ~]# scp /etc/systemd/system/kube-proxy.service 192.168.0.134:/etc/systemd/system/kube-proxy.service
[root@k8s-master1-130 ~]# scp /etc/systemd/system/kube-proxy.service 192.168.0.135:/etc/systemd/system/kube-proxy.service
[root@k8s-master1-131 ~]# vim /etc/systemd/system/kube-proxy.service
[root@k8s-master1-132 ~]# vim /etc/systemd/system/kube-proxy.service
[root@k8s-master1-133 ~]# vim /etc/systemd/system/kube-proxy.service
[root@k8s-master1-134 ~]# vim /etc/systemd/system/kube-proxy.service
[root@k8s-master1-135 ~]# vim /etc/systemd/system/kube-proxy.service
```
9.启动Kubernetes Proxy
[root@k8s-master1-130 ~]# systemctl daemon-reload
[root@k8s-master1-130 ~]# systemctl enable kube-proxy
[root@k8s-master1-130 ~]# systemctl start kube-proxy

10.查看服务状态
查看kube-proxy服务状态
```
[root@k8s-master1-130 ~]# systemctl status kube-proxy

在任意节点，检查LVS状态。
[root@k8s-master1-130 ~]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.1.0.1:443 rr persistent 10800     ### 所有到这个IP地址的转发到192.168.0.130:6443 API-Server上。
  -> 192.168.0.130:6443           Masq    1      0          0         
```
如果你在5台实验机器都安装了kubelet，6台主机安装了kube-proxy服务，使用下面的命令可以检查状态：
```
[root@k8s-master1-130 ssl]#  kubectl get node
NAME            STATUS    ROLES     AGE       VERSION
192.168.0.131   Ready     <none>    27m       v1.10.1
192.168.0.132   Ready     <none>    27m       v1.10.1
192.168.0.133   Ready     <none>    27m       v1.10.1
192.168.0.134   Ready     <none>    27m       v1.10.1
192.168.0.135   Ready     <none>    27m       v1.10.1
```
