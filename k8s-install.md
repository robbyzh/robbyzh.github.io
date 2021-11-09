#### 一.KVM虚拟机安装
1. 设置静态IP地址
```
BOOTPROTO="static"
IPADDR=192.168.122.200
NETMASK=255.255.255.0
GATEWAY=192.168.122.1
DNS1=192.168.122.1
DNS2=114.114.114.114

service network restart
```

2. 设置hostname
```bash
hostnamectl set-hostname your-new-host-name

cat >> /etc/hosts << EOF
192.168.122.201 master201.k8s
192.168.122.202 host202.k8s
192.168.122.203 host203.k8s
192.168.122.204 host204.k8s
192.168.122.205 host205.k8s
EOF
```

3. yum源设置(centos docker-ce)
```bash
yum -y install wget
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

yum clean all
yum makecache
```
```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
可参考 [https://blog.51cto.com/loong576/2398136](https://blog.51cto.com/loong576/2398136)


#### 二.Docker与k8s安装
Linux docker与k8s版本选择  Docker 18.09.6   k8s 1.15.2 
版本对应关系参考:[https://blog.csdn.net/m82_a1/article/details/98872734](https://blog.csdn.net/m82_a1/article/details/98872734)
​


1. Docker
   1. 安装
```bash
yum install -y docker-ce-18.09.6 docker-ce-cli-18.09.6 containerd.io
systemctl enable docker && systemctl start docker
```

   2. docker镜像加速设置
```bash
mkdir -p /etc/docker

tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://dtd27dtr.mirror.aliyuncs.com"]
}
EOF

systemctl daemon-reload
systemctl restart docker
```

   3. 禁用swap,修改内核参数
```bash
swapoff -a;
sed -i.bak '/swap/s/^/#/' /etc/fstab;

sysctl net.bridge.bridge-nf-call-iptables=1;
sysctl net.bridge.bridge-nf-call-ip6tables=1

cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl -p /etc/sysctl.d/k8s.conf
```

2. 安装k8s
```bash
yum install -y kubelet-1.15.2 kubeadm-1.15.2 kubectl-1.15.2

systemctl enable kubelet && systemctl start kubelet
```

3. 下载镜像
```bash
#!/bin/bash
url=registry.cn-hangzhou.aliyuncs.com/google_containers
version=v1.15.12
images=(`kubeadm config images list --kubernetes-version=$version|awk -F '/' '{print $2}'`)
for imagename in ${images[@]} ; do
  docker pull $url/$imagename
  docker tag $url/$imagename k8s.gcr.io/$imagename
  docker rmi -f $url/$imagename
done

# dashboard镜像下载
docker pull anjia0532/google-containers.kubernetes-dashboard-amd64:v1.10.0
docker tag  anjia0532/google-containers.kubernetes-dashboard-amd64:v1.10.0   k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.0
docker rmi  anjia0532/google-containers.kubernetes-dashboard-amd64:v1.10.0

```


#### 三.初始化Master

1. 执行初始化
```bash
kubeadm init --apiserver-advertise-address 192.168.122.201 --pod-network-cidr=10.244.0.0/16

echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source .bash_profile

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

2. 删除Master节点的默认污点(可跳过)
#### 四.Node节点安装

1. 机器配置
1. docker安装
1. k8s安装
1. 下载镜像
1. 加入集群
```bash
# node上执行此命令加入集群
kubeadm join 192.168.122.201:6443 --token 2f1rqh.4s9psft256wx673q \
    --discovery-token-ca-cert-hash sha256:77375c3f891e9b549105e1dac590f8346985bc1c15b0217b5e6560c43bc07bd8
```
​

#### 五.Dashboard安装
```bash
https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml


sed -i 's/k8s.gcr.io/registry.cn-hangzhou.aliyuncs.com\/kuberneters/g' kubernetes-dashboard.yaml
sed -i '/targetPort:/a\ \ \ \ \ \ nodePort: 30001\n\ \ type: NodePort' kubernetes-dashboard.yaml

kubectl apply -f kubernetes-dashboard.yaml


# 网络不通时修改 /usr/lib/systemd/system/docker.service
ExecStartPost=/sbin/iptables -I FORWARD -s 0.0.0.0/0 -j ACCEPT
systemctl daemon-reload && systemctl restart docker


#要用https协议访问 dashboard的url
#登录令牌
kubectl describe secrets -n kube-system dashboard-admin

#删除已停止的容器
docker container prune
ansible k8s -a 'docker container prune -f' -u root

#错误日志问题解决:
#Nov  9 11:15:44 host202 kubelet: E1109 11:15:44.575774     660 summary_sys_containers.go:47] Failed to get system container stats for "/system.slice/docker.service": failed to get cgroup stats for "/system
#.slice/docker.service": failed to get container info for "/system.slice/docker.service": unknown container "/system.slice/docker.service"
ansible k8s -a "sed -i.bak  '/sysconfig\/kubelet/a\Environment=\"KUBELET_CGROUP_ARGS=--runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice\"' /lib/systemd/system/kubelet.service.d/10-kubeadm.conf" -u root
ansible k8s -a "systemctl daemon-reload" -u root
ansible k8s -a "systemctl restart kubelet" -u root
```
​

#### 六.主要参考

1. [https://github.com/loong576/Centos7.6-install-k8s-v1.14.2-cluster](https://github.com/loong576/Centos7.6-install-k8s-v1.14.2-cluster)



