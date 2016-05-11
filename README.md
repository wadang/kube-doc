# kube-doc
Kubernetes cluster creating guide

# 概要
这是一篇我在公司搭建kubernetes集群时所记下的笔记. 根据kubernetes官方的文档, 选择了使用`systemd`来管理kubernetes的进程, 使用`etcd`来管理配置, 用`flannel`来提供vxlan overlay网络.

# 接下来
1. 有些用户希望能像官方文档中所说的将kube-apiserver, kube-controller-manager, kube-scheduller这三个master组件运行在容器内而不是使用systemd来管理, 接下来的文档将会对这一环节进行补充.(虽然我依然认为, 将组件跑在docker里不是个万全的做法)
2. 增加对多master(高可用)的支持
3. 增加etcd的加密及高可用配置
4. 编写一套ansible脚本, 让一切自动化起来,或许有个图形界面更好
