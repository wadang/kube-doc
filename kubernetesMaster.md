Kubernetes Master Deploy Manual
==================
# 有关systemd
如果需要向systemd中添加service, 编写.service文件,并保存在/usr/lib/systemd/system下. 保存后, 执行命令 `systemd daemon-reload` 可以让systemd从文件系统中读取保存的service配置, 之后就可以用`systemctl start service名` `systemctl stop service名`命令启停应用, 也可以用`journalctl -u {service名} -f`来查看service的日志


# 下载官方包,解压缩,拷贝:  
解压缩下载的官方kubernetes包, 进入server目录, 其中kubernetes-server-linux-amd64.tar.gz是64位版kubernetes的文件, 将其解压缩  
解压缩后, 进入server/bin, 这里是kubernetes的可执行文件, 将master所需的三个文件和kubectl命令行工具拷贝至`/usr/bin`下 

```
/usr/bin/kube-apiserver
/usr/bin/kube-controller-manager
/usr/bin/kube-scheduler
/usr/bin/kubectl
``` 


# 1. Config files for kube-apiserver

## 1.1. /etc/kubernetes/apiserver Apiserver配置文件
此文件主要内容为kube-apiserver各启动参数  
创建文件, 修改以下内容  
1. `KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"` 这一项是kubernetes生成的service网段  
2. `KUBE_ETCD_SERVERS="--etcd-servers=http://master1.d.xpaas.lenovo.com:2379"`etcd所在的域名和端口

~~~bash
###
# kubernetes system config
#
# The following values are used to configure the kube-apiserver
#

# The address on the local server to listen to.
KUBE_API_ADDRESS="--insecure-bind-address=127.0.0.1"

# The port on the local server to listen on.
KUBE_API_PORT="--secure-port=443"

# Port nodes listen on
# KUBELET_PORT="--kubelet-port=10250"

# Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"

# Location of the etcd cluster
KUBE_ETCD_SERVERS="--etcd-servers=http://master1.d.xpaas.lenovo.com:2379"

# default admission control policies
KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota"

# Add your own!
KUBE_API_ARGS=" --tls-cert-file=/etc/kubernetes/certs/server.crt --tls-private-key-file=/etc/kubernetes/certs/server.key --client-ca-file=/etc/kubernetes/certs/ca.crt --token-auth-file=/etc/kubernetes/tokens/known_tokens.csv --service-account-key-file=/etc/kubernetes/certs/server.crt"
~~~  
其中`KUBE_API_ARGS`项中, 有几个crt和key文件, 这些文件是集群所用的证书, 生成证书可以使用`https://github.com/kubernetes/contrib/blob/master/ansible/roles/kubernetes/files/make-ca-cert.sh` 下载的脚本去生成.  接下来段落介绍证书生成方式
## 1.2. 生成证书

**下载脚本:**

```bash
curl https://raw.githubusercontent.com/kubernetes/contrib/master/ansible/roles/kubernetes/files/make-ca-cert.sh > make-ca-cert.sh
```

**修改脚本:**  
1. `master_name="${MASTER_NAME:="master1.d.xpaas.lenovo.com"}"`项是master的dns名字  
2. 将`service_range="${SERVICE_CLUSTER_IP_RANGE:="10.254.0.0/16"}"` 部分修改为前面`/etc/kubernetes/apiserver`文件中`KUBE_SERVICE_ADDRESSES `一致. 这一项是用来配置未来kubernetes生成的service的ip范围.    
3. `dns_domain="${DNS_DOMAIN:="d.xpaas.lenovo.com"}"`是集群的dns域  
4. 将`cert_dir="${CERT_DIR:-"/srv/kubernetes"}"`中的证书存储路径修改为`/etc/kubernetes/certs`    
5. `cert_group="${CERT_GROUP:="root"}"` 是证书文件的所属组
 
```bash
cert_ip="${MASTER_IP:="${1}"}"
master_name="${MASTER_NAME:="master1.d.xpaas.lenovo.com"}"
service_range="${SERVICE_CLUSTER_IP_RANGE:="10.254.0.0/16"}"
dns_domain="${DNS_DOMAIN:="d.xpaas.lenovo.com"}"
cert_dir="${CERT_DIR:-"/etc/kubernetes/certs"}"
cert_group="${CERT_GROUP:="root"}"
```
修改后, 执行脚本,脚本的参数为master的ip  

```bash
sh make-ca-cert.sh 10.99.202.204
```
证书将会生成至脚本指定的目录

## 1.3. /etc/kubernetes/config Kubernetes组件通用配置文件
需要修改`KUBE_MASTER="--master=https://master1.d.xpaas.lenovo.com:443"`, 改为master的域名  

```bash
###
# kubernetes system config
#
# The following values are used to configure various aspects of all
# kubernetes services, including
#
#   kube-apiserver.service
#   kube-controller-manager.service
#   kube-scheduler.service
#   kubelet.service
#   kube-proxy.service

# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=true"

# How the replication controller, scheduler, and proxy
KUBE_MASTER="--master=https://master1.d.xpaas.lenovo.com:443"
```

## 1.4. /usr/lib/systemd/system/kube-apiserver.service Apiserver服务文件
kube-apiserver的service文件,用于配置kube-apiserver的服务  
创建文件, 内容如下,无需修改

~~~bash
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/apiserver
ExecStart=/usr/bin/kube-apiserver \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_ETCD_SERVERS \
            $KUBE_API_ADDRESS \
            $KUBE_API_PORT \
            $KUBELET_PORT \
            $KUBE_ALLOW_PRIV \
            $KUBE_SERVICE_ADDRESSES \
            $KUBE_ADMISSION_CONTROL \
            $KUBE_API_ARGS
Restart=on-failure
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
~~~

## 1.5. /etc/kubernetes/tokens/known_tokens.csv Token信息配置文件
用于存储各组件和用户的token信息， 当前版本修改此文件后还需要重启kube-apiserver才能生效  
组件的token存储在相应组件的.kubeconfig文件（yaml格式）的token中  
关于token，可以使用下面的脚本来生成   
```bash
TOKEN=$(dd if=/dev/urandom bs=128 count=1 2>/dev/null | base64 | tr -d "=+/" | dd bs=32 count=1 2>/dev/null)
```   

以下为示例, 当apiserver第一次运行时, 会自动写入一行kubectl的token, 之后如有需要, 按照示例的方式添加即可  
文档地址[Admin:Authenticating](http://kubernetes.io/docs/admin/authentication/)

**注意, 在当前版本的kubernetes中, 修改token文件之后需要重启api server才能生效**

~~~
5MgGhwiDmT2OF4FNSDtTNvyfxgwWJrlL,system:controller_manager-kubemaster1.t.xpaas.lenovo.com,system:controller_manager-kubemaster1.t.xpaas.lenovo.com
LV6ItCSv1QDrei73GOu2vQmjX95tmzQG,system:scheduler-kubemaster1.t.xpaas.lenovo.com,system:scheduler-kubemaster1.t.xpaas.lenovo.com
G7kW7Fk5IMaU7IQsYYiWeXVjWCEmy1oF,system:kubectl-kubemaster1.t.xpaas.lenovo.com,system:kubectl-kubemaster1.t.xpaas.lenovo.com
987XVk1EJT0SAbEzy3QXcy8x1aBK3Iwa,system:kubelet-kubenode1.t.xpaas.lenovo.com,system:kubelet-kubenode1.t.xpaas.lenovo.com
Bsff9MgNeQeMcx2Lawd73I0ItuROjvhP,system:kubelet-kubenode2.t.xpaas.lenovo.com,system:kubelet-kubenode2.t.xpaas.lenovo.com
ThwW6kRk7ptzUlV3XnmsjgYJIZsi5MGq,system:kubelet-kubenode3.t.xpaas.lenovo.com,system:kubelet-kubenode3.t.xpaas.lenovo.com
Lxa8vtdmPpLX47AnLhDMjVngT01EQASU,uikubenode1.t.xpaas.lenovo.com,system:proxy-kubenode1.t.xpaas.lenovo.com
ehQLOcTcpfipGsWhaPBpDMTjKCYdcJra,system:proxy-kubenode2.t.xpaas.lenovo.com,system:proxy-kubenode2.t.xpaas.lenovo.com
hSOXgBKJ3Fs6Nh63RdBcfqK5z2NieFKV,system:proxy-kubenode3.t.xpaas.lenovo.com,system:proxy-kubenode3.t.xpaas.lenovo.com
7B0fyxYpJnBAQgFdLGwyeRY7ddI74Goz,system:dns,system:dns
QgD3Tf0PPW2tDnw4wJQhgTlzOVM5lFhy,system:monitoring,system:monitoring
N3C9CYQ6TDNdQJVLBYGkAWbHrJAseLYZ,system:logging,system:logging
GR3EqHIPqfsYQbxKuY8hjDnM3eaXDhn5,system:kubelet-kubemaster1.t.xpaas.lenovo.com,system:kubelet-kubemaster1.t.xpaas.lenovo.com
d2cJQhO5a7XWw1TMvb3Jv74bGhH6wO4G,system:proxy-kubemaster1.t.xpaas.lenovo.com,system:proxy-kubemaster1.t.xpaas.lenovo.com
~~~

## 1.6. 启动kube-apiserver

```bash
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
```


# 2. Config file for controller manager
## 2.1. /etc/kubernetes/controller-manager Controller Manager配置文件
此文件主要为controller master的启动参数配置

~~~bash
###
# The following values are used to configure the kubernetes controller-manager

# defaults from config and apiserver should be adequate

# Add your own!
KUBE_CONTROLLER_MANAGER_ARGS="--kubeconfig=/etc/kubernetes/controller-manager.kubeconfig --service-account-private-key-file=/etc/kubernetes/certs/server.key --root-ca-file=/etc/kubernetes/certs/ca.crt"
~~~

## 2.2. /usr/lib/systemd/system/kube-controller-manager.service Controller Manager 服务文件
此文件为controller-manager的service配置

~~~bash
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/controller-manager
ExecStart=/usr/bin/kube-controller-manager \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_MASTER \
            $KUBE_CONTROLLER_MANAGER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
~~~

## 2.3. /etc/kubernetes/controller-manager.kubeconfig Controller Manager认证配置
controller-manager的kubeconfig配置文件, 管理着controller manager与master连接时所使用的证书, 用户验证token信息  
1. token要根据文档1.5.章节生成的信息进行修改  
2. server地址要根据master的域名修改

~~~yaml
apiVersion: v1
kind: Config
current-context: controller-manager-to-cluster.local
preferences: {}
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/certs/ca.crt
    server: https://master1.d.xpaas.lenovo.com:443
  name: cluster.local
contexts:
- context:
    cluster: cluster.local
    user: controller-manager
  name: controller-manager-to-cluster.local
users:
- name: controller-manager
  user:
    token: 5MgGhwiDmT2OF4FNSDtTNvyfxgwWJrlL
~~~

##  2.4. 启动kube-controller-manager

```bash
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
```

# 3. Config file for kube scheduler

## 3.1. /etc/kubernetes/scheduler  Scheduler配置文件
此文件用于配置kube scheduler的启动参数

~~~bash
###
# kubernetes scheduler config

# default config should be adequate

# Add your own!
KUBE_SCHEDULER_ARGS="--kubeconfig=/etc/kubernetes/scheduler.kubeconfig"
~~~

## 3.2. /usr/lib/systemd/system/kube-scheduler.service Scheduler服务文件
此文件用于配置scheduler的服务

~~~bash
[Unit]
Description=Kubernetes Scheduler Plugin
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/scheduler
ExecStart=/usr/bin/kube-scheduler \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_MASTER \
            $KUBE_SCHEDULER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
~~~


## 3.3. /etc/kubernetes/scheduler.kubeconfig Scheduler认证配置
scheduler的的kubeconfig配置文件, 管理着scheduler与master连接时所使用的证书, 用户验证token信息
1. token要根据1.5.章节生成的信息进行修改  
2. server地址要根据master的域名修改

~~~yaml
apiVersion: v1
kind: Config
current-context: scheduler-to-cluster.local
preferences: {}
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/certs/ca.crt
    server: https://maseter1.d.xpaas.lenovo.com:443
  name: cluster.local
contexts:
- context:
    cluster: cluster.local
    user: scheduler
  name: scheduler-to-cluster.local
users:
- name: scheduler
  user:
    token: LV6ItCSv1QDrei73GOu2vQmjX95tmzQG
~~~

## 3.4. 启动kube-scheduler

```bash
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
```