
### 机器准备
Master节点：4核CPU，8G内存，20G硬盘，内网ip 192.168.80.128
Node1节点：1核CPU，2G内存，20G硬盘，内网ip 192.168.80.129
Node2节点：2核CPU，2G内存，20G硬盘，内网ip 192.168.80.130

### 安装前命令
1. 关闭防火墙
```
systemctl disable firewalld
systemctl stop firewalld
```
2. 修改/etc/sysconfig/selinux，设置SELINUX=disabled，或者临时关闭（setenforce 0）
3. 关闭swap
```
swapoff -a
free -m # 查看swap分区是否为0
```
4. 修改/etc/hosts文件，加入以下内容（ip需要对应）：
```
192.168.80.128 k8s-master
192.168.80.129 k8s-node1
192.168.80.130 k8s-node2
```
5. 将桥接的IPv4流量传递到iptables的链，修改/etc/sysctl.d/k8s.conf，加入以下内容：
```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```
6. 配置k8s yum源，修改/etc/yum.repos.d/kubernetes.repo
```
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
```
7. 时间同步
```
yum install ntpdate -y
ntpdate time.windows.com
```
8. 使配置生效
```
sysctl --system
```
### 安装Docker
1. 卸载原docker：
```
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine \
                  docker-ce
```
2. 安装yum工具：
```
yum install -y yum-utils \
           device-mapper-persistent-data \
           lvm2 --skip-broken

```
3. 更新本地镜像源：
```
# 设置docker镜像源
yum-config-manager \
    --add-repo \
    https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
    
sed -i 's/download.docker.com/mirrors.aliyun.com\/docker-ce/g' /etc/yum.repos.d/docker-ce.repo

yum makecache fast
```
4. 安装docker：
```
yum install -y docker-ce
```
5. 启动docker
```
systemctl start docker  # 启动docker服务

systemctl stop docker  # 停止docker服务

systemctl restart docker  # 重启docker服务

systemctl enable docker # docker自启动

```
### 安装kubeadm、kubelet和kubectl
```
yum install -y kubelet-1.23.6 kubeadm-1.23.6 kubectl-1.23.6 --disableexcludes=kubernetes
```
自启动kubelet：
```
systemctl enable kubelet
```
### 初始化Master节点
**该命令只在Master节点运行**：
* apiserver地址写Master节点的内网地址。
* image-repository为阿里云地址
* kubernetes-version填写对应安装版本
```
kubeadm init \
	--apiserver-advertise-address=192.168.80.128 \
	--image-repository registry.aliyuncs.com/google_containers \
	--kubernetes-version v1.23.6 \
	--service-cidr=10.96.0.0/12 \
	--pod-network-cidr=10.244.0.0/16
```
![[Pasted image 20240713184438.png]]
初始化成功后，需要按照指引执行以下命令：
```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

```
之后，调用kubectl get nodes即可查看是否安装成功：
![[Pasted image 20240713185831.png]]
### Node节点加入Master
Master init结束后，终端上会输出这样一段内容：
![[Pasted image 20240713191214.png]]
也就是说，我们在node节点执行下面的命令即可将node加入master对应的k8s集群：
```
kubeadm join 192.168.80.128:6443 --token 014i3y.m2zw1md9nxqji9ir \
	--discovery-token-ca-cert-hash sha256:69d1de7686cd68e0c26095480dd0908d75d7caee5c9109a6059c9d38b35d47da
```
但是node节点执行加入操作时，会有和master init时同样的问题，需要修改Docker引擎，具体参考报错问题中的<docker引擎问题>。
加入成功后，显示以下内容：
![[Pasted image 20240713192000.png]]
同时在master节点执行kubectl get nodes命令时，可以显示node节点：
![[Pasted image 20240713192032.png]]
### 安装CNI网络插件
此时K8S节点均为NOT READY状态，调用get pods状态查看，主要原因是coredns为pending状态：
![[Pasted image 20240713203305.png]]
解决该问题的方法是：**在master节点上安装calico**（一个CNI网络插件）
1. 在/opt文件夹下新建calico下载calico配置文件：
```
curl https://docs.projectcalico.org/manifests/calico.yaml -O

# 如果该文件内部进行了redirect，那么下载redirect的内容
curl https://calico-v3-25.netlify.app/archive/v3.25/manifests/calico.yaml > calico.yaml
```
2. 修改calico.yaml文件的CALICO_IPV4POOL_CIDR配置，对应master init时的配置：
![[Pasted image 20240713205315.png]]
![[Pasted image 20240713205747.png]]
![[Pasted image 20240713212431.png]]
3. 查看calico安装所需要的镜像：
```
grep image calico.yaml
```
![[Pasted image 20240713210156.png]]
4. 删除docker.io前缀（不然下载会很慢），并使用docker pull下载镜像：
```
sed -i 's#docker.io/##g' calico.yaml
docker pull calico/cni:v3.25.0
docker pull calico/node:v3.25.0
docker pull calico/kube-controllers:v3.25.0
```
5. 创建calico：
```
kubectl apply -f calico.yaml
```
6. 查看所有pod信息（此时没有default命名空间，需要指定查看kube-system命名空间），如果所有pod均为Running则证明执行成功，如果有fail的pod可以使用describe命令查看原因（大概率是镜像没拉下来）。
```
kubectl get pods -n kube-system
kubectl describe pod <podName> -n kube-system
```
7. 如果出现镜像拉不下来(Pull Image Fail)问题可参考<拉取镜像失败>，由于创建calico会新增<节点数量>个pod，**因此需要在所有节点执行该操作**。（比如node1对应的pod失败，原因是docker在pull时会解析到node1对应的ip地址，这点在describe时可以看出来）
8.  当镜像成功加载到docker时，pod会自动重新加载，如果希望手动重新加载，可以使用delete命令：
```
kubectl delete pod <podName> -n <namespace>
```
9. 最终pod状态如下图所示：
![[Pasted image 20240714212747.png]]
### node节点调用kubectl
当执行完上述步骤后，k8s集群的整体搭建就已经完成了，但是此时node节点是不能执行kubectl命令的：
![[Pasted image 20240714215430.png]]
我们需要执行如下操作：
1. 将master节点中/etc/kubernetes/admin.conf拷贝到需要运行服务器的/etc/kubernetes目录下：
```
scp /etc/kubernetes/admin.conf root@192.168.80.129:/etc/kubernetes
scp /etc/kubernetes/admin.conf root@192.168.80.130:/etc/kubernetes
```
2. 在对应服务器上配置环境变量：
```
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source ~/.bash_profile
```
之后就可以正常执行kubectl命令了。
### 报错问题及修改方法
#### Cannot find a valid baseurl for repo: base/7/x86_64
![[Pasted image 20240713171212.png]]
原因：配置的yum源在国外，无法访问。
解决方法：修改yum源的配置文件即可，改成阿里云的地址（注意不要改成清华的了，那个已经不再维护，依然会报错）
```
1.安装wget
yum install -y wget
2.备份服务器原有的yum源文件
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
3.下载阿里云镜像文件
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
4.清理缓存
yum clean all
5.生成缓存
yum makecache
6.更新最新源设置
yum update -y
```
#### the kubelet version is higher than the control plane version.
![[Pasted image 20240713183326.png]]
一般出现这个问题，基本上是kubelet和kubeadm版本不对应，可以使用下列命令查看版本：
```
kubelet --version
kubeadm verison
```
如果出现版本不一致的情况，则使用 yum -y remove <服务>，重新下载即可：
```
yum -y remove kubelet
yum install -y kubelet-1.23.6 kubeadm-1.23.6 kubectl-1.23.6 --disableexcludes=kubernete

```

#### Docker引擎问题
![[Pasted image 20240713184004.png]]
还有一些其他的报错，基本上原因都是docker Driver为cgroupfs导致的，可以使用 docker info | grep Driver命令查看：
![[Pasted image 20240713185418.png]]
修改docker Driver为systemd，命令如下：
```
vim /etc/docker/daemon.json 
# 写入命令如下
{
    "exec-opts": ["native.cgroupdriver=systemd"]
}


systemctl daemon-reload
systemctl restart docker
system restart kubelet
kubeadm reset # 一定要执行这一步
```

#### 找不到Master的token该怎么办
kubeadm token list：列举当前token
kubeadm token create：获取新token
根据token获取sha256加密值：
```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```
![[Pasted image 20240713201246.png]]之后根据token和sha256:<sha256值>拼接加入请求即可。
#### 拉取镜像失败
自己在搭建k8s集群的时候，calico相关的镜像一直拉取不下来，即使配置了阿里云proxy也没用。
这时候可以使用“在自己电脑上手动下载镜像包”的方法，具体如下：
1. 找到calico的GitHub官方地址，并找到对应版本（v3.25.0），在Assets中下载release包。该包为tgz后缀：https://github.com/projectcalico/calico/releases?page=2
![[Pasted image 20240714211111.png]]
2. 下载成功后，使用scp命令，将该文件上传到虚拟机的/usr/etc文件夹下（注意一定要使用root账号，不然没有访问权限）：
```
scp C:\Users\shangyan\Downloads\release-v3.25.0.tgz root@192.168.80.128:/usr/et
```
![[Pasted image 20240714211356.png]]
3. 解压tgz文件：
```
tar zxvf release-v3.25.0.tgz
```
4. 进入/usr/etc/release-v3.25.0/images文件夹，里面就是calico的所有镜像包对应的tar格式。
![[Pasted image 20240714211701.png]]
5. 执行 docker load -i <tar包名> 命令，将镜像包加载到docker中：
```
docker load -i calico-cni.tar
docker load -i calico-node.tar
docker load -i calico-kube-controllers.tar
```
6. 调用 docker images命令，查看镜像是否加载成功：
![[Pasted image 20240714212052.png]]
