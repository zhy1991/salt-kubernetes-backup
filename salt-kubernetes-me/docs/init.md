
# 系统环境初始化
注：整个部署过程中除了特殊标注需要在指定节点上操作的步骤外，其他步骤均在所有节点都操作。

## 0.系统配置

第一步：配置IP地址。
```
```
第二步：设置主机名。
```
# vim /etc/hostname
k8s-master1-130
```
第三步：设置主机名解析。
```
# vim /etc/hosts
192.168.0.130 k8s-master1-130
192.168.0.131 k8s-node1-131
192.168.0.132 k8s-node1-132
192.168.0.133 k8s-node1-133
192.168.0.134 k8s-node1-134
192.168.0.135 k8s-node1-135
```
第四步：设置DNS解析。
```
# vim /etc/resolv.conf
nameserver 192.168.0.1
```
第五步：关闭SELINUX。
```
# vim /etc/selinux/conf  改成disabled
```
第六步：关闭NetworkManager和防火墙。
```
CentOS-7:
# systemctl disable NetworkManager  && systemctl disable firewalld
Ubuntu-16.04:
# systemctl disable ufw
```
第七步：安装ssh工具，并设置主机间免密码登录（在master节点统一操作）。
```
# yum install -y openssl-server（所有机器都安装）
# ssh-keygen  ### 一路回车
# ssh-copy-id 192.168.0.130  ### 设置所有节点免密码登录。
# ssh-copy-id 192.168.0.131
# ssh-copy-id 192.168.0.132
# ssh-copy-id 192.168.0.133
# ssh-copy-id 192.168.0.134
# ssh-copy-id 192.168.0.135
# ssh 192.168.0.131          ### 测试可以免密码登录，为了后续拷贝文件是方便。
```
第八步：安装EPEL源
```
# rpm -ivh http://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm
```
第九步：安装常用命令。
```
# yum install -y net-tools vim lrzsz tree screen lsof tcpdump nc mtr nmap
```

## 1.安装Docker

CentOS-7 安装docker
第一步：使用国内Docker源
```
[root@k8s-master1-130 ~]# cd /etc/yum.repos.d/
[root@k8s-master1-130 yum.repos.d]# wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
 ```
第二步：Docker安装：
```
[root@k8s-master1-130 ~]# yum install -y docker-ce
```

第三步：启动后台进程：
```
[root@k8s-master1-130 ~]# systemctl start docker
```
Ubuntu-16.04 安装docker
第一步：安装软件包以允许apt通过HTTPS使用存储库
```
# apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
```
第二步：添加Docker的官方GPG密钥
```
# curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
第三步：设置稳定的存储库
```
# sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```
第四步：更新apt源并安装docker
```
# apt-get update
# apt-get install docker-ce
# systemctl status docker   ### 安装完成后可以简单检查一下docker是否正常运行。
# docker version            ### 此安装会安装docker最新版本。
# docker ps
```
## 2.准备部署目录
```
# mkdir -p /opt/kubernetes/{cfg,bin,ssl,log}  ### 分别存放配置、二进制、证书、日志文件。
```
## 3.准备软件包
```
百度网盘下载地址：
[https://pan.baidu.com/s/1WaLSRMMh-qF_MzBbnEufkA]
```
## 4.解压软件包（放在master节点/usr/local/src/目录下并解压）
```
 # cd /usr/local/src/
 # unzip k8s-v1.10.1-manual.zip
 # cd k8s-v1.10.1-manual/k8s-v1.10.1
 # mv * /usr/local/src/
 # cd /usr/local/src/
 # tar zxf kubernetes.tar.gz           ### 以下几个tar包会统一解压至kubernetes目录中。
 # tar zxf kubernetes-server-linux-amd64.tar.gz 
 # tar zxf kubernetes-client-linux-amd64.tar.gz
 # tar zxf kubernetes-node-linux-amd64.tar.gz
```
## 5.在环境变量中为PATH变量添加/opt/kubernetes/bin目录。
CentOS-7
```
# vim .bash_profile
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/opt/kubernetes/bin"
# sources /etc/environment
```
Ubuntu 16.04
```
# vim /etc/environment
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/opt/kubernetes/bin"
# sources /etc/environment
```



