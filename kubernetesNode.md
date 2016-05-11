Kubernetes node deploy manual
===============

# 拷贝文件
从解压缩的kubernetes 包server/bin/目录中拷贝kube-proxy和kubelet至/usr/bin目录下

~~~
/usr/bin/kube-proxy
/usr/bin/kubelet
~~~

**如果这个node与master不是在同一台机器上, 还需要把master上面生成的/etc/kubernetes/certs/ca.crt拷贝至本机的相同目录**

# 1. Config files for kubelet

##  1.1. /etc/kubernetes/kubelet Kubelet配置文件
此文件主要内容为kube-apiserver各启动参数  
创建文件, 修改以下内容  
1. 修改apiserver地址`KUBELET_API_SERVER="--api-servers=https://master1.d.xpaas.lenovo.com:443"`  
2. 修改当前节点域名 `KUBELET_HOSTNAME="--hostname-override=node1.d.xpaas.lenovo.com"`, 此项也可以留空, 空代表直接使用本机域名

```bash
###
# kubernetes kubelet (node) config

# The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=0.0.0.0"

# The port for the info server to serve on
# KUBELET_PORT="--port=10250"

# You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=node1.d.xpaas.lenovo.com"

# location of the api-server
KUBELET_API_SERVER="--api-servers=https://master1.d.xpaas.lenovo.com:443"

KUBELET_ARGS="--kubeconfig=/etc/kubernetes/kubelet.kubeconfig --config=/etc/kubernetes/manifests --cluster-dns=10.254.0.10 --cluster-domain=cluster.local"

```

## 1.2. /etc/kubernetes/kubelet.kubeconfig Kubelet安全及认证配置
Kubelet的kubeconfig配置文件, 管理着kubelet与master连接时所使用的证书, 用户验证token信息  
1. token要根据master上生成的token信息进行修改  
2. server地址要根据master的域名修改

```yaml
apiVersion: v1
kind: Config
current-context: kubelet-to-cluster.local
preferences: {}
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/certs/ca.crt
    server: https://master1.d.xpaas.lenovo.com:443
  name: cluster.local
contexts:
- context:
    cluster: cluster.local
    user: kubelet
  name: kubelet-to-cluster.local
users:
- name: kubelet
  user:
    token: GR3EqHIPqfsYQbxKuY8hjDnM3eaXDhn5
```

## 1.3. /etc/kubernetes/config Kubernetes组件通用配置文件
需要修改`KUBE_MASTER="--master=https://master1.d.xpaas.lenovo.com:443"`, 改为master的域名及端口

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

## 1.4. 创建工作目录/var/lib/kubelet
```
mkdir -p /var/lib/kubelet
```

## 1.5. /usr/lib/systemd/system/kubelet.service Kubelet服务文件
Kubelet的service文件,用于配置kubelet的服务  
创建文件, 内容如下,无需修改

```bash
[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/bin/kubelet \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBELET_API_SERVER \
            $KUBELET_ADDRESS \
            $KUBELET_PORT \
            $KUBELET_HOSTNAME \
            $KUBE_ALLOW_PRIV \
            $KUBELET_ARGS
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

## 1.6 启动kubelet

```bash
systemctl enable kubelet
systemctl start kubelet
```

# 2. Kubeproxy config files

## 2.1. /etc/kubernetes/proxy Kubeproxy配置文件
此文件主要内容为kube-apiserver各启动参数 
无需修改

~~~bash
###
# kubernetes proxy config

# default config should be adequate

# Add your own!
KUBE_PROXY_ARGS="--kubeconfig=/etc/kubernetes/proxy.kubeconfig"
~~~

## 2.2. /etc/kubernetes/proxy.kubeconfig Kubeproxy安全及认证文件
Kubeproxy的kubeconfig配置文件, 管理着kubeproxy与master连接时所使用的证书, 用户验证token信息
1. token要根据master上生成的token信息进行修改
2. server地址要根据master的域名修改

~~~yaml
apiVersion: v1
kind: Config
current-context: proxy-to-cluster.local
preferences: {}
contexts:
- context:
    cluster: cluster.local
    user: proxy
  name: proxy-to-cluster.local
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/certs/ca.crt
    server: https://master1.d.xpaas.lenovo.com:443
  name: cluster.local
users:
- name: proxy
  user:
    token: d2cJQhO5a7XWw1TMvb3Jv74bGhH6wO4G
~~~


## 2.3. /usr/lib/systemd/system/kube-proxy.service Kubeproxy服务文件
Kubeproxy的service文件,用于配置kubeproxy的服务
创建文件, 内容如下,无需修改

~~~bash
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/proxy
ExecStart=/usr/bin/kube-proxy \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_MASTER \
            $KUBE_PROXY_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
~~~

## 2.4. 启动kube-proxy

```bash
systemctl enable kube-proxy
systemctl start kube-proxy
```
