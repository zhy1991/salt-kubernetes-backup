1.创建一个测试用的deployment
```
[root@k8s-master1-130 ~]# kubectl run net-test --image=alpine --replicas=2 sleep 360000
```

2.查看获取IP情况
```
root@k8s-master1-130:~# kubectl get pods -o wide
NAME                        READY     STATUS              RESTARTS   AGE       IP        NODE
net-test-5767cb94df-pnt7c   0/1       ContainerCreating   0          8s        <none>    192.168.0.135
net-test-5767cb94df-vm6gk   0/1       ContainerCreating   0          8s        <none>    192.168.0.133

root@k8s-master1-130:~/yaml# kubectl get pods -o wide
NAME                                READY     STATUS              RESTARTS   AGE       IP          NODE
net-test-5767cb94df-pnt7c           1/1       Running             0          5m        10.2.31.2   192.168.0.135
net-test-5767cb94df-vm6gk           1/1       Running             0          5m        10.2.97.2   192.168.0.133
```

3.测试联通性
```
root@k8s-master1-130:~/yaml# ping 10.2.31.2
PING 10.2.31.2 (10.2.31.2) 56(84) bytes of data.
64 bytes from 10.2.31.2: icmp_seq=1 ttl=63 time=2.89 ms
64 bytes from 10.2.31.2: icmp_seq=2 ttl=63 time=0.521 ms

root@k8s-master1-130:~/yaml# ping 10.2.97.2
PING 10.2.97.2 (10.2.97.2) 56(84) bytes of data.
64 bytes from 10.2.97.2: icmp_seq=1 ttl=63 time=1.84 ms
64 bytes from 10.2.97.2: icmp_seq=2 ttl=63 time=0.571 ms
```
4.创建一个nginx的deployment
```
# vim vim nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.10.3
        ports:
        - containerPort: 80
# kubectl create -f nginx-deploy.yaml
deployment.apps "nginx-deployment" created
```
### 查看创建deployment
```
root@k8s-master1-130:~/yaml# kubectl get deploy
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
net-test           2         2         2            2           8m
nginx-deployment   3         3         3            0           4m
```
### 查看创建的pod
```
root@k8s-master1-130:~/yaml# kubectl get pods
NAME                                READY     STATUS              RESTARTS   AGE
net-test-5767cb94df-pnt7c           1/1       Running             0          8m
net-test-5767cb94df-vm6gk           1/1       Running             0          8m
nginx-deployment-75d56bb955-4sdzn   0/1       ContainerCreating   0          4m
nginx-deployment-75d56bb955-gdzpw   0/1       ContainerCreating   0          4m
nginx-deployment-75d56bb955-hcq4j   0/1       ContainerCreating   0          4m
```
### 查看pod的创建详情中的event信息，正在拉去镜像。
```
# root@k8s-master1-130:~/yaml# kubectl describe  pod/nginx-deployment-75d56bb955-4sdzn
Events:
  Type    Reason                 Age   From                    Message
  ----    ------                 ----  ----                    -------
  Normal  Scheduled              5m    default-scheduler       Successfully assigned nginx-deployment-75d56bb955-4sdzn to 192.168.0.131
  Normal  SuccessfulMountVolume  5m    kubelet, 192.168.0.131  MountVolume.SetUp succeeded for volume "default-token-2ljf5"
  Normal  Pulling                5m    kubelet, 192.168.0.131  pulling image "nginx:1.10.3"
已全部运行
# root@k8s-master1-130:~/yaml# kubectl get pods -o wide
NAME                                READY     STATUS    RESTARTS   AGE       IP          NODE
net-test-5767cb94df-pnt7c           1/1       Running   0          23m       10.2.31.2   192.168.0.135
net-test-5767cb94df-vm6gk           1/1       Running   0          23m       10.2.97.2   192.168.0.133
nginx-deployment-75d56bb955-4sdzn   1/1       Running   0          18m       10.2.12.2   192.168.0.131
nginx-deployment-75d56bb955-gdzpw   1/1       Running   0          18m       10.2.97.3   192.168.0.133
nginx-deployment-75d56bb955-hcq4j   1/1       Running   0          18m       10.2.68.2   192.168.0.134
```
### 测试联通性，先ping POD IP 在curl POD IP访问应用。
```
# root@k8s-master1-130:~/yaml# ping 10.2.12.2
PING 10.2.12.2 (10.2.12.2) 56(84) bytes of data.
64 bytes from 10.2.12.2: icmp_seq=1 ttl=63 time=13.3 ms
64 bytes from 10.2.12.2: icmp_seq=2 ttl=63 time=0.592 ms

# root@k8s-master1-130:~/yaml# ping 10.2.97.3
PING 10.2.97.3 (10.2.97.3) 56(84) bytes of data.
64 bytes from 10.2.97.3: icmp_seq=1 ttl=63 time=1.60 ms
64 bytes from 10.2.97.3: icmp_seq=2 ttl=63 time=0.672 ms

# root@k8s-master1-130:~/yaml# ping 10.2.68.2
PING 10.2.68.2 (10.2.68.2) 56(84) bytes of data.
64 bytes from 10.2.68.2: icmp_seq=1 ttl=63 time=6.14 ms
64 bytes from 10.2.68.2: icmp_seq=2 ttl=63 time=0.986 ms

# curl 10.2.12.2
<!DOCTYPE html>   ### 分别可以返回nginx欢迎页信息
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

# curl --head http://10.2.97.3    ### 正常返回200说明正常，且nginx版本为1.10.3。
HTTP/1.1 200 OK
Server: nginx/1.10.3
Date: Sun, 27 May 2018 15:28:56 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 31 Jan 2017 15:01:11 GMT
Connection: keep-alive
ETag: "5890a6b7-264"
Accept-Ranges: bytes
```
### 更新deployment，加set参数可以直接修改deployment的yaml文件内容。
```
# kubectl set image deployment/nginx-deployment nginx=nginx:1.12.2 --record    ### 加上--record参数会使用replace set记录升级日志，方便回滚。
deployment.apps "nginx-deployment" image updated

### 查看状态正在创建
root@k8s-master1-130:~/yaml# kubectl get pods
NAME                                READY     STATUS              RESTARTS   AGE
net-test-5767cb94df-pnt7c           1/1       Running             0          31m
net-test-5767cb94df-vm6gk           1/1       Running             0          31m
nginx-deployment-7498dc98f8-9599p   0/1       ContainerCreating   0          15s
nginx-deployment-75d56bb955-4sdzn   1/1       Running             0          26m
nginx-deployment-75d56bb955-gdzpw   1/1       Running             0          26m
nginx-deployment-75d56bb955-hcq4j   1/1       Running             0          26m

### 查看deployment会发现，期望3个、当前4个、有一个在创建。也就是说老的还没有删除，保证可用再升级新的。
root@k8s-master1-130:~/yaml# kubectl get deploy
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
net-test           2         2         2            2           31m
nginx-deployment   3         4         1            3           27m

root@k8s-master1-130:~/yaml# kubectl get deploy -o wide
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS   IMAGES         SELECTOR
net-test           2         2         2            2           36m       net-test     alpine         run=net-test
nginx-deployment   3         4         1            3           31m       nginx        nginx:1.12.2   app=nginx

### 有一个已经升级完了，在升级第二个。
root@k8s-master1-130:~/yaml# kubectl get pods -o wide
NAME                                READY     STATUS              RESTARTS   AGE       IP          NODE
net-test-5767cb94df-pnt7c           1/1       Running             0          40m       10.2.31.2   192.168.0.135
net-test-5767cb94df-vm6gk           1/1       Running             0          40m       10.2.97.2   192.168.0.133
nginx-deployment-7498dc98f8-9599p   1/1       Running             0          9m        10.2.31.3   192.168.0.135
nginx-deployment-7498dc98f8-trs49   0/1       ContainerCreating   0          1m        <none>      192.168.0.132
nginx-deployment-75d56bb955-gdzpw   1/1       Running             0          36m       10.2.97.3   192.168.0.133
nginx-deployment-75d56bb955-hcq4j   1/1       Running             0          36m       10.2.68.2   192.168.0.134
root@k8s-master1-130:~/yaml# kubectl get deploy
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
net-test           2         2         2            2           40m
nginx-deployment   3         4         2            3           36m

### 访问以下升级完的nginx
# curl --head http://10.2.31.3
root@k8s-master1-130:~/yaml# curl --head http://10.2.31.3
HTTP/1.1 200 OK
Server: nginx/1.12.2       ### 可以看到升级完的nginx的版本是新的
Date: Sun, 27 May 2018 15:43:38 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 11 Jul 2017 13:29:18 GMT
Connection: keep-alive
ETag: "5964d2ae-264"
Accept-Ranges: bytes

### 全部升级完成
root@k8s-master1-130:~/yaml# kubectl get pods -o wide
NAME                                READY     STATUS    RESTARTS   AGE       IP          NODE
net-test-5767cb94df-pnt7c           1/1       Running   0          55m       10.2.31.2   192.168.0.135
net-test-5767cb94df-vm6gk           1/1       Running   0          55m       10.2.97.2   192.168.0.133
nginx-deployment-7498dc98f8-9599p   1/1       Running   0          24m       10.2.31.3   192.168.0.135
nginx-deployment-7498dc98f8-sfx57   1/1       Running   0          10m       10.2.97.4   192.168.0.133
nginx-deployment-7498dc98f8-trs49   1/1       Running   0          16m       10.2.85.2   192.168.0.132
root@k8s-master1-130:~/yaml# kubectl get deploy
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
net-test           2         2         2            2           55m
nginx-deployment   3         3         3            3           51m
```
### 查看更新历史
```
root@k8s-master1-130:~/yaml# kubectl rollout history deployment/nginx-deployment
deployments "nginx-deployment"
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deployment/nginx-deployment nginx=nginx:1.12.2 --record=true
```
### 查看具体某一个版本的升级历史，可以看到都做了什么。
```
root@k8s-master1-130:~/yaml# kubectl rollout history deployment/nginx-deployment --revision=1
deployments "nginx-deployment" with revision #1
Pod Template:
  Labels:	app=nginx
	pod-template-hash=3181266511
  Containers:
   nginx:
    Image:	nginx:1.10.3
    Port:	80/TCP
    Host Port:	0/TCP
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
```
### 快速回滚到上一个版本
```
# kubectl rollout undo deployment/nginx-deployment
### 可以发现POD的IP地址在不停的发生着变化，这就是Service要解决的问题。
```
5.为nginx的deployment创建一个Service
```
# vim nginx-deploy-svc.yaml
kind: Service
apiVersion: v1
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80

# kubectl create -f nginx-deploy-svc.yaml 
service "nginx-service" created

root@k8s-master1-130:~/yaml# kubectl get service
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes      ClusterIP   10.1.0.1       <none>        443/TCP   3h
nginx-service   ClusterIP   10.1.241.107   <none>        80/TCP    2m

### 查看lvs负载情况
root@k8s-master1-130:~/yaml# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.1.0.1:443 rr persistent 10800
  -> 192.168.0.130:6443           Masq    1      0          0         
TCP  10.1.241.107:80 rr      ### 可以看到当访问service IP为10.1.241.107的80端口是会被转发到后端的三个POD实例上。
  -> 10.2.31.3:80                 Masq    1      0          0         
  -> 10.2.85.2:80                 Masq    1      0          0         
  -> 10.2.97.4:80                 Masq    1      0          0 

### 访问测试lvs负载策略
root@k8s-node1-131:~# curl --head http://10.1.241.107  ### 在node节点访问，如果想在master节点访问service IP需要安装kubelet和kube-proxy服务。
HTTP/1.1 200 OK
Server: nginx/1.12.2
Date: Sun, 27 May 2018 16:00:02 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 11 Jul 2017 13:29:18 GMT
Connection: keep-alive
ETag: "5964d2ae-264"
Accept-Ranges: bytes

### 查看lvs上的转发记录
root@k8s-master1-130:~/yaml# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.1.0.1:443 rr persistent 10800
  -> 192.168.0.130:6443           Masq    1      0          0         
TCP  10.1.241.107:80 rr
  -> 10.2.31.3:80                 Masq    1      0          0         
  -> 10.2.85.2:80                 Masq    1      0          0         
  -> 10.2.97.4:80                 Masq    1      0          1

### 快速扩容至5个POD实例
```
# kubectl scale deployment/nginx-deployment --replicas 5


```

