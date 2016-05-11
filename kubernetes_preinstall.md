kubernetes 安装前准备
===================
# 安装docker
为每台服务器安装docker
增加docker官方yum源, 创建文件`/etc/yum.repos.d/docker.repo`  内容如下:

```bash
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
```

之后, 安装并启动docker

```
yum install -y docker-engine
systemctl start docker
systemctl enable docker
```
# 安装nfs-utils
为了之后支持nfs格式的presistence volume,为每一台服务器都要安装

```bash
yum install nfs-utils -y
```


# 准备安装包
## kubernets 解压缩
```
tar -zxvf kubernetes1.2.2.tar.gz
```


# 配置启动etcd
etcd官方建议使用kubernetes包所提供的版本, 所以进入解压缩好的kubernetes目录中执行`make`, make会自动制作etcd的docker镜像. 所以在执行make之前记得检察一下docker服务是否启动

```
cd kubernetes/cluster/images/etcd/
make
```
执行之后可以通过`docker images` 命令看到生成的etcd镜像

```
[root@master1 etcd]# docker images
REPOSITORY                            TAG                 IMAGE ID            CREATED             SIZE
gcr.io/google_containers/etcd-amd64   2.2.1               f02037440786        9 minutes ago       28.19 MB
gcr.io/google_containers/etcd         2.2.1               f02037440786        9 minutes ago       28.19 MB
busybox                               latest              47bcc53f74dc        6 weeks ago         1.113 MB
```

接下来,运行docker命令启动etcd

```
docker run -d  \
-p 4001:4001 \
-p 2380:2380 \
-p 2379:2379  \
--name etcd \
--restart always \
-v /data/servers/etcd:/etcd:rw \
gcr.io/google_containers/etcd-amd64:2.2.1  \
/usr/local/bin/etcd  \
-data-dir /etcd/ \
-name etcd0  \
-advertise-client-urls http://10.99.202.204:2379,http://10.99.202.204:4001  \
-listen-client-urls http://0.0.0.0:2379,http://0.0.0.0:4001  \
-initial-advertise-peer-urls http://10.99.202.204:2380  \
-listen-peer-urls http://0.0.0.0:2380  \
-initial-cluster-token etcd-cluster-dev  \
-initial-cluster etcd0=http://10.99.202.204:2380  \
-initial-cluster-state new
```

将etcdctl拷贝至`/usr/bin/etcdctl`

# 配置启动flannel
## 将flannel添加至systemd服务
### 创建文件/etc/sysconfig/flanneld


```bash
# Flanneld configuration options

# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD="http://10.99.202.204:2379"

# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
#FLANNEL_ETCD_KEY="/cluster.local/network"

# Any additional options that you want to pass
# By default, we just add a good guess for the network interface on Vbox.  Otherwise, Flannel will probably make the right guess.
FLANNEL_OPTIONS=""
```	

### 拷贝文件
解压缩flannel包,将`flanneld`文件拷贝至`/usr/bin`, 将`mk-docker-opts.sh`文件拷贝至`/usr/libexec/flannel/`

```
tar zxvf flannel-0.5.5-linux-amd64.tar.gz
cp flannel-0.5.5/flanneld /usr/bin
mkdir -p /usr/libexec/flannel/
cp mk-docker-opts.sh /usr/libexec/flannel/
```

### 创建systemd配置文件/usr/lib/systemd/system/flanneld.service

路径`/usr/lib/systemd/system/`是systemd管理的各个服务的配置文件所在目录

```
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/flanneld
ExecStart=/usr/bin/flanneld -etcd-endpoints=${FLANNEL_ETCD}  $FLANNEL_OPTIONS
ExecStartPost=/usr/libexec/flannel/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker

[Install]
RequiredBy=docker.service
                           
```
 
在起动之前先将flanneld所需的配置写入etcdctl, 在整个flannel集群创建之前执行一次即可, 比如安装配置node上的flannel时就不必执行这句了

```
etcdctl set /coreos.com/network/config '{ "Network": "172.16.0.0/12", "SubnetLen": 24, "Backend": { "Type": "vxlan" } }'
```

在创建好文件后, 需要让systemd的daemon重新load目录中的所有服务配置文件, 这样新的服务才能生效

```
systemctl daemon-reload
systemctl start flanneld.service
systemctl enable flanneld
```
在配置好每个节点的flannel之后,需要docker的网络使用flannel,所以需要修改docker的service文件`/usr/lib/systemd/system/docker.service`  
1. 在[Service]段落增加一行`EnvironmentFile=-/run/flannel/docker`
2. 将ExecStart修改为下面这样

```bash
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target docker.socket
Requires=docker.socket

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
EnvironmentFile=-/run/flannel/docker
ExecStart=/usr/bin/docker daemon -H fd:// \
          $OPTIONS \
          $DOCKER_STORAGE_OPTIONS \
          $DOCKER_NETWORK_OPTIONS \
          $INSECURE_REGISTRY
MountFlags=slave
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes

[Install]
WantedBy=multi-user.target
~                            
``` 

修改后重启docker

```
systemctl daemon-reload
systemctl restart docker
```
再检查`ifconfig`结果, 就能看到docker使用了flannel的网络
