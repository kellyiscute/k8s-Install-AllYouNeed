# k8s-Install-AllYouNeed
在墙内如何顺利地安装k8s  
以Deb系为例，使用的版本为1.19

##  `0x00` 准备就绪
CLONE THIS REPO
clone下这个仓库  
`git clone https://github.com/guo40020/k8s-Install-AllYouNeed`  
关闭swap  
`sudo swapoff -a`  
安装docker  
`sudo apt install docker docker.io containerd`  
**如果你在使用Ubuntu 19.04/Debian 10或更高版本**则需要把iptables改为legacy模式  
```bash
update-alternatives --set iptables /usr/sbin/iptables-legacy
update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
update-alternatives --set arptables /usr/sbin/arptables-legacy
update-alternatives --set ebtables /usr/sbin/ebtables-legacy
```

## `0x01` 寻求国内镜像的庇护
在这里，我们将使用**清华TUNA协会**的镜像，which is [镜像站地址](https://mirrors.tuna.tsinghua.edu.cn/)  
1. 首先导入apt key  
`cd k8s-Install-AllYouNeed`  
`sudo apt-key add k8s-key.gpg`
2. 添加k8s镜像源  
```bash
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.tuna.tsinghua.edu.cn/kubernetes/apt/ kubernetes-xenial main
EOF
```
3. 安装 `kubeadm` `kubelet` `kubectl`
首先更新apt软件列表
`sudo apt update`
安装  
`sudo apt install -y kubeadm kubelet kubectl`  
由于让apt自动升级会导致无法预期的问题，我们需要保持版本  
`sudo apt hold kubeadm kubelet kubectl`  

## `0x02` 解决k8s镜像问题
k8s需要从[google-containers](k8s.gcr.io)也就是`k8s.gcr.io`拉取几个必要镜像  
这些镜像可以通过命令`kubeadm config images list`查看  
**但是！** 在国内根本无法访问google-containers！  
那么，有两种方法可以让我们成功装上这些镜像  

### 方法 01 `如果你有代理的话...`
如果有代理的话，那就方便了不少，  
首先我们需要让docker连上代理干活  
我们需要找到`docker.service`  
这个文件在`/lib/systemd/system/docker.service`  
那么用你喜欢的编辑器 (nano? vim?) 编辑它（**记得sudo**）  
打开之后，找到`[Service]`行  
在下面添加如下内容  
```bash
Environment="HTTP_PROXY=http://<proxy-ip-addr>:<port>"
Environment="HTTPS_PROXY=http://<proxy-ip-addr>:<port>"
Environment="NO_PROXY=127.0.0.1,localhost"
```
然后保存  
接下来就可以愉快的拉镜像了  
`sudo kubeadm config images pull`  
根据你的网络体质，可能需要花点时间 **记住，没有消息就是最好的消息**  

### 方法 02 `当服务器无法连接代理...`
如果你的服务器无法连接上代理的话...  
麻烦就来了  
你依然需要求助于代理  
但是可以通过一些操作把镜像在有代理的电脑上拉下来再传上去  
比如说 **使用docker save**  
docker允许用户把镜像导出为tar包，那么我们就可以让一台电脑作为不赚差价的中间商，  
把代理先拉下来，然后导出  
接着，使用sftp / scp等工具推上服务器，然后使用 `docker load --input <filename>` 恢复成docker镜像
  
当然，还有其他的骚操作可以完成这个任务，  
比如说……  
用一台国外的服务器fake一个 k8s.gcr.io，  
通过修改hosts文件，让 k8s.gcr.io 指向这台服务器，然后从这台服务器拉取镜像...  
只是过程繁琐，这就就不再赘述  

## `0x03` 大功告成
至此，困难的部分已经结束。  
但是还有一部分我们没有做完： `kubeadm init`
**但是！** 这里需要注意一个地方……  
因为k8s需要一个网络插件才能完全初始化集群，  
那么就需要去找一个支持CNI的网络插件  
这里我们使用[Calico](https://www.projectcalico.org/)  
首先，初始化集群master  
`sudo kubeadm init --pod-network-cidr=192.168.0.0/16`  
上面的`--pod-network-cidr`网段如果已经被占用，则需要选择别的网段  
在kubeadm完成初始化任务后，在最后的输出里面，包含了一下几个命令  
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```  
逐个运行（当然，不运行也是可以的...）
接下来，安装Calico网络插件  
```bash
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
```  
接下来监视k8s的pod运行状态  
```bash
watch kubectl get pods -A
```  
当所有pod都变为Running时，运行以下命令来允许在控制面上调度pod  
`kubectl taint nodes --all node-role.kubernetes.io/master-`  

## END
那么，现在一个单节点的k8s就已经安装完成了  
ENJOY LEARNING~~
