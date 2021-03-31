(Git对md的格式支持比较差，大家可以下载到本地或者在git上选择原始文件方式进行阅读)
# Kubernetes新版本更新
当前Kubernetes的默认版本更新到了1.19+，整理安装方式、各种API、兼容性问题归纳如下  

# Kubernetes环境准备和离线部署方式
整个环境采用三台机器，一台控制节点，两台工作节点，分别安装最新版的Docker和Kubernetes软件。
首先，需要在三台机器上，分别完成Docker的安装准备。直接从Docker官网下载安装：  
```
# curl -fsSL https://get.docker.com -o get-docker.sh 
# sh get-docker.sh
```
如果服务器无法连接外网，可以将运行脚本命令修改如下，指定通过阿里云镜像进行下载安装：  
```
# sh get-docker.sh --mirror Aliyun
```
如果操作系统不是完全安装，可能会提示缺少某些依赖包，可以通过安装光盘或者Docker官网下载对应依赖包。  
安装完成后启动Docker，并将其添加到自动启动：  
```
# systemctl restart docker
# systemctl enable docker
```
下一步，在三个节点上，参照方文档，分别安装Kubeadm、kubelet、kubectl等工具：  
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/  
以CentOS/RedHat/Fedora为例：
```
# cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# sudo setenforce 0
# sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

# sudo systemctl enable --now kubelet
```
如果服务器无法连接到Google镜像仓库，可以采用阿里云镜像仓库，安装步骤修改为：  
```
# cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# sudo setenforce 0
# sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# sudo yum install -y kubelet kubeadm kubectl
# sudo systemctl enable --now kubelet
```
下一步，是在控制节点上初始化Kuberentes集群。如果该服务器无法连接到Google镜像仓库，需要提前将软件包下载、重命名，以Kubernetes V1.19.2为例，命令为：  
```
docker pull kubeimage/kube-apiserver-amd64:v1.19.2
docker pull kubeimage/kube-controller-manager-amd64:v1.19.2
docker pull kubeimage/kube-scheduler-amd64:v1.19.2
docker pull kubeimage/kube-proxy-amd64:v1.19.2
docker pull kubeimage/pause-amd64:3.2
docker pull kubeimage/etcd-amd64:3.3.10-0
docker pull coredns/coredns:1.7.0
docker tag kubeimage/kube-apiserver-amd64:v1.19.2 k8s.gcr.io/kube-apiserver:v1.19.2
docker tag kubeimage/kube-controller-manager-amd64:v1.19.2 k8s.gcr.io/kube-controller-manager:v1.19.2
docker tag kubeimage/kube-scheduler-amd64:v1.19.2 k8s.gcr.io/kube-scheduler:v1.19.2
docker tag kubeimage/kube-proxy-amd64:v1.19.2 k8s.gcr.io/kube-proxy:v1.19.2
docker tag kubeimage/pause-amd64:3.2 k8s.gcr.io/pause:3.2
docker tag kubeimage/etcd-amd64:3.3.10-0 k8s.gcr.io/etcd:3.4.13-0
docker tag coredns/coredns:1.7.0 k8s.gcr.io/coredns:1.7.0
docker rmi kubeimage/kube-apiserver-amd64:v1.19.2
docker rmi kubeimage/kube-controller-manager-amd64:v1.19.2
docker rmi kubeimage/kube-scheduler-amd64:v1.19.2
docker rmi kubeimage/kube-proxy-amd64:v1.19.2
docker rmi kubeimage/pause-amd64:3.2
docker rmi kubeimage/etcd-amd64:3.3.10-0
docker rmi coredns/coredns:1.7.0
```
整个Kubernetes集群初始化过程非常方便。在控制节点上运行kubeadm命令，创建Kubernetes集群，将172.20.230.77替换为控制节点的IP地址：  
```
# kubeadm init --apiserver-advertise-address=172.20.230.77 --pod-network-cidr=10.244.0.0/16
```
按照命令提示完成剩余操作：  
```
# mkdir -p $HOME/.kube
# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
# sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
然后，参照官网选择网络技术栈，连通Kubernetes集群:  
https://kubernetes.io/docs/concepts/cluster-administration/addons/  
这里以Flannel网络为例：  
```
# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
控制节点上的任务已经完成。最后，是让两台工作节点加入Kubernete集群。  
如果该服务器无法连接到Google镜像仓库，需要提前将软件包下载、重命名，以Kubernetes V1.19.2为例，命令为：  
```
docker pull kubeimage/kube-proxy-amd64:v1.19.2
docker pull kubeimage/pause-amd64:3.2
docker tag kubeimage/kube-proxy-amd64:v1.19.2 k8s.gcr.io/kube-proxy:v1.19.2
docker tag kubeimage/pause-amd64:3.2 k8s.gcr.io/pause:3.2
docker rmi kubeimage/kube-proxy-amd64:v1.19.2
docker rmi kubeimage/pause-amd64:3.2
```
根据之前控制节点安装过程的提示的IP地址和token、hash值，在两台工作节点上分别运行命令：  
```
# kubeadm join 172.20.230.77:6443 --token q5yw61.yix5or5y4z0dvbpn \
    --discovery-token-ca-cert-hash sha256:6d199317942ef966eb76ddbe8a546e8ffd76d1644941fbb846ffe18c0dee0e5c
```
几分钟后，工作节点加入完毕，在控制节点上可以查看到集群状态变为：  
```
# kubectl get nodes
NAME     STATUS   ROLES    AGE     VERSION
master   Ready    master   20m     v1.19.2
work1    Ready    <none>   10m     v1.19.2
work2    Ready    <none>   5m      v1.19.2
```

# Deployment API变化
注：新的Deployment API变为apps/v1，并且需要添加matchLabels。举例如下：  
```
[apiVersion: apps/v1
kind: Deployment
metadata:
  name: productpage-v1
  labels:
    app: productpage
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: productpage
      version: v1
  template:
    metadata:
      labels:
        app: productpage
        version: v1
    spec:
      serviceAccountName: bookinfo-productpage
      containers:
      - name: productpage
        image: docker.io/istio/examples-bookinfo-productpage-v1:1.16.2
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}
```

# Canal部署的注意事项
K8S的1.20.X版本和Canal存在冲突。如果尝试Canal网络策略验证，发现Nodeport兼容性问题，建议采用1.19.0的Kubernetes版本和3.17的Canal版本重新部署。  

# Helm更新
Helm从2.0更新3.0，取消了复杂的Helm客户端加Tiller服务端的架构，采用Helm单个组件完成整体部署。  
Helm V3的安装更简单：  
```
# curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
# chmod +x get_helm.sh 
# ./get_helm.sh 
```
Helm repo的操作方式更灵活：  
```
# helm repo add stable https://kubernetes-charts.storage.googleapis.com/
# helm repo add aliyuncs https://apphub.aliyuncs.com
# helm search repo prometheus-operator
# helm install prometheus-release stable/prometheus-operator
```
