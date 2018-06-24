# Kubernetes Dashboard

## 创建CoreDNS
```
# vim coredns.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
  labels:
      kubernetes.io/cluster-service: "true"
      addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: Reconcile
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: EnsureExists
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
  labels:
      addonmanager.kubernetes.io/mode: EnsureExists
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local. in-addr.arpa ip6.arpa {
            pods insecure
            upstream
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        proxy . /etc/resolv.conf
        cache 30
    }
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: coredns
  template:
    metadata:
      labels:
        k8s-app: coredns
    spec:
      serviceAccountName: coredns
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      containers:
      - name: coredns
        image: coredns/coredns:1.0.6
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: coredns
  clusterIP: 10.1.0.2   ### Service IP的网段，如果不一样需要修改。客户端kubelet启动参数中也需要修改指定的--cluster-dns=10.1.0.2参数。
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
    
[root@k8s-master1-130 ~]# kubectl create -f coredns.yaml 
serviceaccount "coredns" created
clusterrole.rbac.authorization.k8s.io "system:coredns" created
clusterrolebinding.rbac.authorization.k8s.io "system:coredns" created
configmap "coredns" created
deployment.extensions "coredns" created
service "coredns" created

[root@k8s-master1-130 ~]# kubectl get pod -n kube-system -o wide
NAME                       READY     STATUS              RESTARTS   AGE       IP        NODE
coredns-77c989547b-2prlc   0/1       ContainerCreating   0          52s       <none>    192.168.0.131
coredns-77c989547b-bbf8l   0/1       ContainerCreating   0          52s       <none>    192.168.0.135

root@k8s-master1-130:~/salt-kubernetes/addons/dashboard# kubectl get pods -o wide -n kube-system
NAME                                    READY     STATUS              RESTARTS   AGE       IP          NODE
coredns-77c989547b-2prlc                1/1       Running             0          11m       10.2.12.4   192.168.0.131
coredns-77c989547b-bbf8l                1/1       Running             0          11m       10.2.31.4   192.168.0.135
```
### 测试DNS服务
```
# kubectl run dns-test --rm -it --image=alpine /bin/sh  ### 创建一个临时容器
/ # ping www.baidu.com
PING www.baidu.com (220.181.112.244): 56 data bytes
64 bytes from 220.181.112.244: seq=0 ttl=55 time=110.484 ms
64 bytes from 220.181.112.244: seq=1 ttl=55 time=6.113 ms

/ # nslookup www.baidu.com
nslookup: can't resolve '(null)': Name does not resolve

Name:      www.baidu.com
Address 1: 220.181.112.244
Address 2: 220.181.111.188
```


### 创建Dashboard
```
[root@k8s-master1-130 ~]# kubectl create -f dashboard/
serviceaccount "admin-user" created
clusterrolebinding.rbac.authorization.k8s.io "admin-user" created
secret "kubernetes-dashboard-certs" created
serviceaccount "kubernetes-dashboard" created
role.rbac.authorization.k8s.io "kubernetes-dashboard-minimal" created
rolebinding.rbac.authorization.k8s.io "kubernetes-dashboard-minimal" created
deployment.apps "kubernetes-dashboard" created
service "kubernetes-dashboard" created
clusterrole.rbac.authorization.k8s.io "ui-admin" created
rolebinding.rbac.authorization.k8s.io "ui-admin-binding" created
clusterrole.rbac.authorization.k8s.io "ui-read" created
rolebinding.rbac.authorization.k8s.io "ui-read-binding" created
```
### 通过node节点ip访问dashboard，下面是获取登录token。
```
root@k8s-master1-130:~# kubectl get svc -n  kube-system
NAME                   TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
coredns                ClusterIP   10.1.0.2      <none>        53/UDP,53/TCP   24m
kubernetes-dashboard   NodePort    10.1.73.199   <none>        443:31701/TCP   21m

root@k8s-master1-130:~# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.0.130:31701 rr
  -> 10.2.97.5:8443               Masq    1      0          0         
TCP  10.1.0.1:443 rr persistent 10800
  -> 192.168.0.130:6443           Masq    1      0          0         
TCP  10.1.0.2:53 rr
  -> 10.2.12.4:53                 Masq    1      0          0         
  -> 10.2.31.4:53                 Masq    1      0          0         
TCP  10.1.73.199:443 rr
  -> 10.2.97.5:8443               Masq    1      0          0         
TCP  10.1.241.107:80 rr
  -> 10.2.12.3:80                 Masq    1      0          0         
  -> 10.2.31.3:80                 Masq    1      0          0         
  -> 10.2.68.3:80                 Masq    1      0          0         
  -> 10.2.85.2:80                 Masq    1      0          0         
  -> 10.2.97.4:80                 Masq    1      0          0         
TCP  10.2.52.0:31701 rr
  -> 10.2.97.5:8443               Masq    1      0          0         
TCP  10.2.52.1:31701 rr
  -> 10.2.97.5:8443               Masq    1      0          0         
TCP  127.0.0.1:31701 rr
  -> 10.2.97.5:8443               Masq    1      0          0         
UDP  10.1.0.2:53 rr
  -> 10.2.12.4:53                 Masq    1      0          0         
  -> 10.2.31.4:53                 Masq    1      0          0

通过node IP:NodePort端口31701访问
```

查询token
```
# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
或者
# kubectl -n kube-system describe secret `kubectl -n kube-system get secret|grep admin-token|cut -d " " -f1`|grep "token:"|tr -s " "|cut -d " " -f2 | head -n1
```
