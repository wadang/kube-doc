# kube-doc
Kubernetes cluster creating guide

# 概要
这是一篇我在公司搭建kubernetes集群时所记下的笔记. 根据kubernetes官方的文档, 选择了使用`systemd`来管理kubernetes的进程, 使用`etcd`来管理配置, 用`flannel`来提供vxlan overlay网络.

# 接下来
1. 有些用户希望能像官方文档中所说的将kube-apiserver, kube-controller-manager, kube-scheduler这三个master组件运行在容器内而不是使用systemd来管理, 接下来的文档将会对这一环节进行补充.(虽然我依然认为, 将组件跑在docker里不是个万全的做法)
2. 增加对多master(高可用)的支持
3. 增加etcd的加密及高可用配置
4. 编写一套ansible脚本, 让一切自动化起来,或许有个图形界面更好
5. 这一点似乎应该写在靠前一些: 为文档增加一章如何为kubernetes安装addon, 以及重要addon的安装指导
6. 这一点似乎应该写在更靠前一些: 丰富每一步操作的讲解, 讲解原理和参数配置分析, 这样读者看完文章后可以根据自己的理解和需求搭建自己的集群

# 文档怎么看
目前共有三篇文档, 先从`kubernetes_preinstall.md`开始,  里面介绍了在安装kubernetes之前所需要的操作. 接下来`kubernetesMaster.md`和`kubernetesNode.md`分别讲了master和node中的各组件配置方式.
```bash
kubernetes_preinstall.md #安装前的准备
kubernetesMaster.md #kube-apiserver kube-controller-manager kube-scheduler 配置方式
kubernetesNode.md #kubelet kube-proxy 配置方式
```

# 重要
这篇文档基于我个人对kubernetes的理解,所以肯定是有考虑不周全甚至方向的错误, 恳请看到文档的人斧正这篇文档, 非常乐意和各路高手学习交流.  
**如需转载,请联系我先**

