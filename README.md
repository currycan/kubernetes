# kubeadm 安装部署 kubernetes 1.14.0

## 1. 安装准备

- 系统：centos 7

- 用户： root

- 机器规划:

|  角色  | 数量  |  配置  |     物理ip     |  hostname  |
| :----: | :---: | :----: | :------------: | :--------: |
| master |   1   | 2核 2g | 192.168.43.252 | k8s-master |
| node1  |   1   | 2核 2g | 192.168.43.252 | k8s-node1  |
| node2  |   1   | 2核 2g | 192.168.43.252 | k8s-node2  |

- 硬件配置参考：CPU 2核或以上，内存2GB或以上。
- 机器最好都在同一个局域网，在三台机器上都设置好hostname
- 1个cpu的话初始化master的时候会报`[WARNING NumCPU]: the number of available CPUs 1 is less than the required 2`，部署插件或者`pod`时可能会报`warning：FailedScheduling：Insufficient cpu, Insufficient memory`

### 1.1 校准时区

```shell
# timedatectl list-timezones
timedatectl set-timezone Asia/Shanghai
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
yum install -y ntpdate
systemctl start ntpdate.service
systemctl enable ntpdate.service
echo '*/30 * * * * /usr/sbin/ntpdate cn.pool.ntp.org >/dev/null 2>&1' > /tmp/crontab2.tmp
crontab /tmp/crontab2.tmp
systemctl start ntpdate.service
```
### 1.2 优化系统参数
```shell
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf
echo "* soft nproc 65536"  >> /etc/security/limits.conf
echo "* hard nproc 65536"  >> /etc/security/limits.conf
echo "* soft  memlock  unlimited"  >> /etc/security/limits.conf
echo "* hard memlock  unlimited"  >> /etc/security/limits.conf
```

### 1.3 系统设置
```shell
# 关闭防火墙，swap，selinux
systemctl disable --now firewalld NetworkManager
setenforce 0
sed -ri '/^[^#]*SELINUX=/s#=.+$#=disabled#' /etc/selinux/config
# swapoff -a && sysctl -w vm.swappiness=0
# rm -rf $(swapon -s | awk 'NR>1{print $1}')
# sed -i 's/.*swap.*/#&/' /etc/fstab

sed -i "13i exclude=kernel*" /etc/yum.conf
yum install epel-release -y
yum install -y wget git jq psmisc socat ipvsadm ipset sysstat conntrack libseccomp
yum update -y

# 升级内核
rpm -Uvh https://elrepo.org/linux/kernel/el7/x86_64/RPMS/kernel-lt-4.4.177-1.el7.elrepo.x86_64.rpm
grub2-set-default  0 && grub2-mkconfig -o /etc/grub2.cfg
grubby --default-kernel

reboot
```
**加载内核模块**
```shell
:> /etc/modules-load.d/ipvs.conf
module=(
  br_netfilter
  ip_vs
  ip_vs_lc
  ip_vs_wlc
  ip_vs_rr
  ip_vs_wrr
  ip_vs_lblc
  ip_vs_lblcr
  ip_vs_dh
  ip_vs_sh
  ip_vs_fo
  ip_vs_nq
  ip_vs_sed
  ip_vs_ftp
  nf_conntrack
  )
for kernel_module in ${module[@]};do
    /sbin/modinfo -F filename $kernel_module |& grep -qv ERROR && echo $kernel_module >> /etc/modules-load.d/ipvs.conf || :
done
systemctl enable --now systemd-modules-load.service
nl /etc/modules-load.d/ipvs.conf
systemctl status systemd-modules-load.service -l
```
上面如果'systemctl enable'命令报错可以'systemctl status -l systemd-modules-load.service'看看哪个内核模块加载不了,在'/etc/modules-load.d/ipvs.conf'里注释掉它再enable试试

**Tips:**
因为测试主机上还运行其他服务，关闭swap可能会对其他服务产生影响，所以建议修改kubelet的启动参数
```
--fail-swap-on=false
```

 去掉这个限制。修改
```
/etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

加入：
```
# 已经安装好kubernetes环境
sed "6i Environment="KUBELET_EXTRA_ARGS=--fail-swap-on=false" -i /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
systemctl daemon-reload && systemctl restart kubelet
```

### 1.4 调整内核参数,解决路由异常
```shell
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl -p /etc/sysctl.d/k8s.conf
```

## 2 安装

### 2.1 安装docker

检查系统内核和模块是否适合运行 docker (仅适用于 linux 系统)
```
curl https://raw.githubusercontent.com/docker/docker/master/contrib/check-config.sh > check-config.sh
bash ./check-config.sh
```

```shell
# 删除残留（如果之前有安装过docker）
yum remove docker -y \
                docker-client \
                docker-client-latest \
                docker-common \
                docker-latest \
                docker-latest-logrotate \
                docker-logrotate \
                docker-selinux \
                docker-engine-selinux \
                docker-engine
# 此步骤会删除所有数据，慎做
rm -rf /var/lib/docker

# 安装docker
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum list docker-ce --showduplicates | sort -r
yum install -y docker-ce

\cp -a /usr/share/bash-completion/completions/docker /etc/bash_completion.d/
systemctl enable --now docker
systemctl status docker -l

cat << EOF > /etc/docker/daemon.json
{
    "registry-mirrors": [
        "https://registry.docker-cn.com",
        "https://8trm4p9x.mirror.aliyuncs.com",
        "http://010a79c4.m.daocloud.io",
        "https://docker.mirrors.ustc.edu.cn/"
    ],
    "insecure-registries": ["harbor.iibu.com"],
    "storage-driver": "overlay2",
    "storage-opts": ["overlay2.override_kernel_check=true"],
    "exec-opts": ["native.cgroupdriver=cgroupfs"],
    "max-concurrent-downloads": 10,
    "log-driver": "json-file",
    "log-level": "warn",
    "metrics-addr" : "0.0.0.0:9323",
    "experimental" : true,
    "log-opts": {
       "max-size": "10m",
       "max-file": "3"
    },
  "data-root": "/var/lib/docker"
}
EOF
systemctl daemon-reload && systemctl restart docker

# 设置docker代理配置，http代理(如果用代理就可以直接从gcr.io直接pull镜像)
mkdir -p /etc/systemd/system/docker.service.d
cat <<EOF > /etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=http://currycan:shachao123321@192.168.39.88:1080"
Environment="HTTPS_PROXY=http://currycan:shachao123321@192.168.39.88:1080"
Environment="NO_PROXY=localhost,127.0.0.1,127.0.0.0/8,harbor.iibu.com,registry.docker-cn.com,8trm4p9x.mirror.aliyuncs.com,registry.cn-hangzhou.aliyuncs.com,docker.mirrors.ustc.edu.cn,010a79c4.m.daocloud.io"
EOF
systemctl daemon-reload && systemctl restart docker
systemctl show --property=Environment docker
```

### 2.2 安装kubeadm, kubelet和kubectl

配置kubernetes源
```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
source <(kubectl completion bash)
source <(kubeadm completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
echo "source <(kubeadm completion bash)" >> ~/.bashrc

sed '5i Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"' -i /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
sed '6i Environment="KUBELET_EXTRA_ARGS=--fail-swap-on=false"' -i /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf

cat << EOF > /etc/sysconfig/kubelet
KUBELET_CGROUP_ARGS="--cgroup-driver=cgroupfs"
KUBELET_EXTRA_ARGS="--fail-swap-on=false"
KUBE_PROXY=MODE=ipvs
EOF

# 如果失败执行
DOCKER_CGROUPS=$(docker info | grep 'Cgroup' | cut -d' ' -f3)
cat >/etc/sysconfig/kubelet<<EOF
KUBELET_CGROUP_ARGS="--cgroup-driver=$DOCKER_CGROUPS"
KUBELET_EXTRA_ARGS="--fail-swap-on=false"
EOF
sed -e 's/^Environment="KUBELET_CGROUP_ARGS=.*/Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"/g' -i /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf

systemctl enable kubelet
systemctl daemon-reload && systemctl restart kubelet
```
注意，这里不需要启动kubelet，初始化的过程中会自动启动的，如果此时启动了会出现如下报错，忽略即可。日志在tail -f /var/log/messages
```
Apr  2 20:44:41 just-test kubelet: F0402 20:44:41.022247    5858 server.go:193] failed to load Kubelet config file /var/lib/kubelet/config.yaml, error failed to read kubelet config file "/var/lib/kubelet/config.yaml", error: open /var/lib/kubelet/config.yaml: no such file or directory
```

### 2.3 下载gcr.io镜像

kubeadm init 命令默认使用的docker镜像仓库为k8s.gcr.io，国内无法直接访问，于是需要变通一下。首先查看需要使用哪些镜像
```shell
[root@just-test ~]# kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.14.0
k8s.gcr.io/kube-controller-manager:v1.14.0
k8s.gcr.io/kube-scheduler:v1.14.0
k8s.gcr.io/kube-proxy:v1.14.0
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1
```
可以通过 docker.io/mirrorgooglecontainers 中转一下
注：coredns没包含在docker.io/mirrorgooglecontainers中，需要手工从coredns官方镜像转换下。

批量下载及转换标签，脚本如下
```shell
kubeadm config images list |sed -e 's/^/docker pull /g' -e 's#k8s.gcr.io#docker.io/mirrorgooglecontainers#g' |sh -x
docker images |grep mirrorgooglecontainers |awk '{print "docker tag ",$1":"$2,$1":"$2}' |sed -e 's#[docker.io/]*mirrorgooglecontainers#k8s.gcr.io#2' |sh -x
docker images |grep mirrorgooglecontainers |awk '{print "docker rmi ", $1":"$2}' |sh -x
docker pull coredns/coredns:1.3.1
docker tag coredns/coredns:1.3.1 k8s.gcr.io/coredns:1.3.1
docker rmi coredns/coredns:1.3.1
```
分解步骤
```shell
[root@just-test ~]# kubeadm config images list |sed -e 's/^/docker pull /g' -e 's#k8s.gcr.io#docker.io/mirrorgooglecontainers#g' |sh -x
+ docker pull docker.io/mirrorgooglecontainers/kube-apiserver:v1.14.0
v1.14.0: Pulling from mirrorgooglecontainers/kube-apiserver
1552c1d94162: Pull complete
c3f0381da60f: Pull complete
Digest: sha256:3bf082bfadff726fcb2e8acb32d640b836839cb121d29d62f50c072e7ef65be2
Status: Downloaded newer image for mirrorgooglecontainers/kube-apiserver:v1.14.0
+ docker pull docker.io/mirrorgooglecontainers/kube-controller-manager:v1.14.0
v1.14.0: Pulling from mirrorgooglecontainers/kube-controller-manager
1552c1d94162: Already exists
792f70ee1bcc: Pull complete
Digest: sha256:b6b3663d445c9f843e1a104a85d57e8853f03eb8e250c38dfbc1be6888e9eeb1
Status: Downloaded newer image for mirrorgooglecontainers/kube-controller-manager:v1.14.0
+ docker pull docker.io/mirrorgooglecontainers/kube-scheduler:v1.14.0
v1.14.0: Pulling from mirrorgooglecontainers/kube-scheduler
1552c1d94162: Already exists
19de5072ac25: Pull complete
Digest: sha256:1989038d353e1dc041e1bd219067da133619cbead44484393c3927845ce746ff
Status: Downloaded newer image for mirrorgooglecontainers/kube-scheduler:v1.14.0
+ docker pull docker.io/mirrorgooglecontainers/kube-proxy:v1.14.0
v1.14.0: Pulling from mirrorgooglecontainers/kube-proxy
1552c1d94162: Already exists
152cd3f507ac: Pull complete
ee506f620da3: Pull complete
Digest: sha256:56c56857dbab79470aa37a77cc5190419590ed03ca360883c05189391ec9ff05
Status: Downloaded newer image for mirrorgooglecontainers/kube-proxy:v1.14.0
+ docker pull docker.io/mirrorgooglecontainers/pause:3.1
3.1: Pulling from mirrorgooglecontainers/pause
67ddbfb20a22: Pull complete
Digest: sha256:59eec8837a4d942cc19a52b8c09ea75121acc38114a2c68b98983ce9356b8610
Status: Downloaded newer image for mirrorgooglecontainers/pause:3.1
+ docker pull docker.io/mirrorgooglecontainers/etcd:3.3.10
3.3.10: Pulling from mirrorgooglecontainers/etcd
860b4e629066: Pull complete
3de3fe131c22: Pull complete
12ec62a49b1f: Pull complete
Digest: sha256:8a82adeb3d0770bfd37dd56765c64d082b6e7c6ad6a6c1fd961dc6e719ea4183
Status: Downloaded newer image for mirrorgooglecontainers/etcd:3.3.10
+ docker pull docker.io/mirrorgooglecontainers/coredns:1.3.1
Error response from daemon: pull access denied for mirrorgooglecontainers/coredns, repository does not exist or may require 'docker login'
```
下载完成的镜像
```shell
[root@just-test ~]# docker images
REPOSITORY                                       TAG                 IMAGE ID            CREATED             SIZE
mirrorgooglecontainers/kube-proxy                v1.14.0             5cd54e388aba        7 days ago          82.1MB
mirrorgooglecontainers/kube-apiserver            v1.14.0             ecf910f40d6e        7 days ago          210MB
mirrorgooglecontainers/kube-controller-manager   v1.14.0             b95b1efa0436        7 days ago          158MB
mirrorgooglecontainers/kube-scheduler            v1.14.0             00638a24688b        7 days ago          81.6MB
mirrorgooglecontainers/etcd                      3.3.10              2c4adeb21b4f        4 months ago        258MB
mirrorgooglecontainers/pause                     3.1                 da86e6ba6ca1        15 months ago       742kB
```
给镜像添加tag
```shell
[root@just-test ~]# docker images |grep mirrorgooglecontainers |awk '{print "docker tag ",$1":"$2,$1":"$2}' |sed -e 's#[docker.io/]*mirrorgooglecontainers#k8s.gcr.io#2' |sh -x
+ docker tag mirrorgooglecontainers/kube-proxy:v1.14.0 k8s.gcr.io/kube-proxy:v1.14.0
+ docker tag mirrorgooglecontainers/kube-scheduler:v1.14.0 k8s.gcr.io/kube-scheduler:v1.14.0
+ docker tag mirrorgooglecontainers/kube-apiserver:v1.14.0 k8s.gcr.io/kube-apiserver:v1.14.0
+ docker tag mirrorgooglecontainers/kube-controller-manager:v1.14.0 k8s.gcr.io/kube-controller-manager:v1.14.0
+ docker tag mirrorgooglecontainers/etcd:3.3.10 k8s.gcr.io/etcd:3.3.10
+ docker tag mirrorgooglecontainers/pause:3.1 k8s.gcr.io/pause:3.1
[root@just-test ~]# docker images
REPOSITORY                                       TAG                 IMAGE ID            CREATED             SIZE
mirrorgooglecontainers/kube-proxy                v1.14.0             5cd54e388aba        7 days ago          82.1MB
k8s.gcr.io/kube-proxy                            v1.14.0             5cd54e388aba        7 days ago          82.1MB
mirrorgooglecontainers/kube-apiserver            v1.14.0             ecf910f40d6e        7 days ago          210MB
k8s.gcr.io/kube-apiserver                        v1.14.0             ecf910f40d6e        7 days ago          210MB
mirrorgooglecontainers/kube-controller-manager   v1.14.0             b95b1efa0436        7 days ago          158MB
k8s.gcr.io/kube-controller-manager               v1.14.0             b95b1efa0436        7 days ago          158MB
mirrorgooglecontainers/kube-scheduler            v1.14.0             00638a24688b        7 days ago          81.6MB
k8s.gcr.io/kube-scheduler                        v1.14.0             00638a24688b        7 days ago          81.6MB
mirrorgooglecontainers/etcd                      3.3.10              2c4adeb21b4f        4 months ago        258MB
k8s.gcr.io/etcd                                  3.3.10              2c4adeb21b4f        4 months ago        258MB
mirrorgooglecontainers/pause                     3.1                 da86e6ba6ca1        15 months ago       742kB
k8s.gcr.io/pause                                 3.1                 da86e6ba6ca1        15 months ago       742kB
```
删除旧的tag
```shell
[root@just-test ~]# docker images |grep mirrorgooglecontainers |awk '{print "docker rmi ", $1":"$2}' |sh -x
+ docker rmi mirrorgooglecontainers/kube-proxy:v1.14.0
Untagged: mirrorgooglecontainers/kube-proxy:v1.14.0
Untagged: mirrorgooglecontainers/kube-proxy@sha256:56c56857dbab79470aa37a77cc5190419590ed03ca360883c05189391ec9ff05
+ docker rmi mirrorgooglecontainers/kube-apiserver:v1.14.0
Untagged: mirrorgooglecontainers/kube-apiserver:v1.14.0
Untagged: mirrorgooglecontainers/kube-apiserver@sha256:3bf082bfadff726fcb2e8acb32d640b836839cb121d29d62f50c072e7ef65be2
+ docker rmi mirrorgooglecontainers/kube-controller-manager:v1.14.0
Untagged: mirrorgooglecontainers/kube-controller-manager:v1.14.0
Untagged: mirrorgooglecontainers/kube-controller-manager@sha256:b6b3663d445c9f843e1a104a85d57e8853f03eb8e250c38dfbc1be6888e9eeb1
+ docker rmi mirrorgooglecontainers/kube-scheduler:v1.14.0
Untagged: mirrorgooglecontainers/kube-scheduler:v1.14.0
Untagged: mirrorgooglecontainers/kube-scheduler@sha256:1989038d353e1dc041e1bd219067da133619cbead44484393c3927845ce746ff
+ docker rmi mirrorgooglecontainers/etcd:3.3.10
Untagged: mirrorgooglecontainers/etcd:3.3.10
Untagged: mirrorgooglecontainers/etcd@sha256:8a82adeb3d0770bfd37dd56765c64d082b6e7c6ad6a6c1fd961dc6e719ea4183
+ docker rmi mirrorgooglecontainers/pause:3.1
Untagged: mirrorgooglecontainers/pause:3.1
Untagged: mirrorgooglecontainers/pause@sha256:59eec8837a4d942cc19a52b8c09ea75121acc38114a2c68b98983ce9356b8610
[root@just-test ~]# docker images
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy                v1.14.0             5cd54e388aba        7 days ago          82.1MB
k8s.gcr.io/kube-controller-manager   v1.14.0             b95b1efa0436        7 days ago          158MB
k8s.gcr.io/kube-scheduler            v1.14.0             00638a24688b        7 days ago          81.6MB
k8s.gcr.io/kube-apiserver            v1.14.0             ecf910f40d6e        7 days ago          210MB
k8s.gcr.io/etcd                      3.3.10              2c4adeb21b4f        4 months ago        258MB
k8s.gcr.io/pause                     3.1                 da86e6ba6ca1        15 months ago       742kB
```
拉取coredns镜像
```shell
[root@just-test ~]# docker pull coredns/coredns:1.3.1
1.3.1: Pulling from coredns/coredns
e0daa8927b68: Pull complete
3928e47de029: Pull complete
Digest: sha256:02382353821b12c21b062c59184e227e001079bb13ebd01f9d3270ba0fcbf1e4
Status: Downloaded newer image for coredns/coredns:1.3.1
[root@just-test ~]# docker images | grep coredns
coredns/coredns                      1.3.1               eb516548c180        2 months ago        40.3MB
[root@just-test ~]# docker tag coredns/coredns:1.3.1 k8s.gcr.io/coredns:1.3.1
[root@just-test ~]# docker rmi coredns/coredns:1.3.1
Untagged: coredns/coredns:1.3.1
Untagged: coredns/coredns@sha256:02382353821b12c21b062c59184e227e001079bb13ebd01f9d3270ba0fcbf1e4
[root@just-test ~]# docker images | grep coredns
k8s.gcr.io/coredns                   1.3.1               eb516548c180        2 months ago        40.3MB
```
查看镜像列表
```shell
[root@just-test ~]# docker images
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy                v1.14.0             5cd54e388aba        7 days ago          82.1MB
k8s.gcr.io/kube-apiserver            v1.14.0             ecf910f40d6e        7 days ago          210MB
k8s.gcr.io/kube-controller-manager   v1.14.0             b95b1efa0436        7 days ago          158MB
k8s.gcr.io/kube-scheduler            v1.14.0             00638a24688b        7 days ago          81.6MB
k8s.gcr.io/coredns                   1.3.1               eb516548c180        2 months ago        40.3MB
k8s.gcr.io/etcd                      3.3.10              2c4adeb21b4f        4 months ago        258MB
k8s.gcr.io/pause                     3.1                 da86e6ba6ca1        15 months ago       742kB
```

现在已经满足要求了，可以愉快的继续kubeadm init

另外一种方法是使用kubeadm配置文件，通过在配置文件中指定docker仓库地址，便于内网快速部署。目前处于试验阶段。
生成配置文件
```shell
kubeadm config print-defaults --api-objects ClusterConfiguration >kubeadm.conf
```
将配置文件中的`imageRepository: k8s.gcr.io`改为你自己的私有`docker`仓库，比如`imageRepository: docker.io/mirrorgooglecontainers`
`kubeadm`生成的配置文件目前不够完善，需要修改`kubernetes`版本
`kubernetesVersion: v1.12.0`改为`kubernetesVersion: v1.12.2`
然后运行命令
```shell
kubeadm config images list --config kubeadm.conf
kubeadm config images pull --config kubeadm.conf
kubeadm init --config kubeadm.conf
```
更多kubeadm配置文件参数详见
```shell
kubeadm config print-defaults
```

## 3. 在master上配置

### 3.1 初始化K8S

初始化之前最好先了解一下 kubeadm init 参数
```
--apiserver-advertise-address string
API Server将要广播的监听地址。如指定为 `0.0.0.0` 将使用缺省的网卡地址。

--apiserver-bind-port int32     缺省值: 6443
API Server绑定的端口

--apiserver-cert-extra-sans stringSlice
可选的额外提供的证书主题别名（SANs）用于指定API Server的服务器证书。可以是IP地址也可以是DNS名称。

--cert-dir string     缺省值: "/etc/kubernetes/pki"
证书的存储路径。

--config string
kubeadm配置文件的路径。警告：配置文件的功能是实验性的。

--cri-socket string     缺省值: "/var/run/dockershim.sock"
指明要连接的CRI socket文件

--dry-run
不会应用任何改变；只会输出将要执行的操作。

--feature-gates string
键值对的集合，用来控制各种功能的开关。可选项有:
Auditing=true|false (当前为ALPHA状态 - 缺省值=false)
CoreDNS=true|false (缺省值=true)
DynamicKubeletConfig=true|false (当前为BETA状态 - 缺省值=false)

-h, --help
获取init命令的帮助信息

--ignore-preflight-errors stringSlice
忽视检查项错误列表，列表中的每一个检查项如发生错误将被展示输出为警告，而非错误。 例如: 'IsPrivilegedUser,Swap'. 如填写为 'all' 则将忽视所有的检查项错误。

--kubernetes-version string     缺省值: "stable-1"
为control plane选择一个特定的Kubernetes版本。

--node-name string
指定节点的名称。

--pod-network-cidr string
指明pod网络可以使用的IP地址段。 如果设置了这个参数，control plane将会为每一个节点自动分配CIDRs。

--service-cidr string     缺省值: "10.96.0.0/12"
为service的虚拟IP地址另外指定IP地址段

--service-dns-domain string     缺省值: "cluster.local"
为services另外指定域名, 例如： "myorg.internal".

--skip-token-print
不打印出由 `kubeadm init` 命令生成的默认令牌。

--token string
这个令牌用于建立主从节点间的双向受信链接。格式为 [a-z0-9]{6}\.[a-z0-9]{16} - 示例： abcdef.0123456789abcdef

--token-ttl duration     缺省值: 24h0m0s
令牌被自动删除前的可用时长 (示例： 1s, 2m, 3h). 如果设置为 '0', 令牌将永不过期。
```
**注意：**

如果要安装网络插件flannel ，这里要添加参数， --pod-network-cidr=10.244.0.0/16，10.244.0.0/16是flannel插件固定使用的ip段，它的值取决于你准备安装哪个网络插件

如果要自定义配置，先kubeadm config print init-defaults >kubeadm.conf，再修改，改完指定配置文件路径--config /root/kubeadm.conf

指定Kubenetes版本--kubernetes-version，如果不指定该参数，会从google网站下载最新的版本信息，因为它的默认值是stable-1。

测试的话，如果只分配一个cpu，需要指定参数--ignore-preflight-errors=NumCPU，如果cpu足够，不要添加这个参数.

-- apiserver-advertise-address该参数一般指定为haproxy+keepalived 的vip。
-- pod-network-cidr 主要是在搭建pod network（calico）时候需要在init时候指定。

在master上开始初始化,通过kubeadm init命令来初始化，指定一下kubernetes版本，并设置一下pod-network-cidr。此过程，需要从google cloud下载镜像，前面步骤中已做了镜像下载，完成后如下：

```shell
# kubeadm init --apiserver-advertise-address=10.177.37.90 --pod-network-cidr=10.88.0.0/16 --kubernetes-version=v1.14.2 --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers/ --ignore-preflight-errors=Swap --dry-run
[root@just-test ~]# kubeadm init --kubernetes-version=v1.14.0 --apiserver-advertise-address=192.168.43.252 --pod-network-cidr=10.244.0.0/16
[init] Using Kubernetes version: v1.14.0
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [just-test localhost] and IPs [192.168.43.252 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [just-test localhost] and IPs [192.168.43.252 127.0.0.1 ::1]
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [just-test kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.43.252]
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 18.004252 seconds
[upload-config] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.14" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --experimental-upload-certs
[mark-control-plane] Marking the node just-test as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node just-test as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: 9mm3u2.abk82donxijbreds
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.43.252:6443 --token 9mm3u2.abk82donxijbreds \
    --discovery-token-ca-cert-hash sha256:a8d910eed14dc0a9e4c6f702847bb264638b7b372390b5e35fab02eb18db2507
```

根据输出的内容，可以了解到初始化Kubernetes集群所需要的关键步骤。
```
1、[preflight] 检查系统状态。 发生错误会退出执行，除非指定了--ignore-preflight-errors=<list-of-errors>
2、[kubelet-start] 启动kubelet
3、[certs] 生成各种证书。可以通过 --cert-dir 指定自有的证书目录（缺省值为 /etc/kubernetes/pki）
4、[kubeconfig] 在/etc/kubernetes/ 目录，生成配置文件 admin.conf（kubectl） ，kubelet.conf 、controller-manager.conf 和 scheduler.conf
5、[control-plane] 为 apiserver、controller manager 和 scheduler 生成创建Pod时要用到的yaml文件。
6、[etcd]生成 本地 etcd 的Pod yaml，除非指定外部 etcd
7、[wait-control-plane] 安装master的组件 apiserver、controller manager 和 scheduler
8、[apiclient] 检查组件是否健康
9、[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
10、[kubelet] Creating a ConfigMap "kubelet-config-1.14" in namespace kube-system with the configuration for the kubelets in the cluster
11、[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "master.hanli.com" as an annotation
12、[mark-control-plane] 将master节点标记为不可调度
13、[bootstrap-token] token
14、[addons] 安装 CoreDNS和kube-proxy
```

机器上的用户要使用 kubectl来 管理集群操作集群，需要配置kubectl认证信息（Master节点操作）
```shell
# 非root用户
mkdir -p $HOME/.kubeyu
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# root用户
export KUBECONFIG=/etc/kubernetes/admin.conf
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source  ~/.bash_profile

# 单节点执行，允许master节点被调度
kubectl taint nodes --all node-role.kubernetes.io/master-
```

###3.2 将worker节点加入集群（此部分来源于网络）

在所有work节点上，使用初始化时给出的命令，将worker加入集群
```
[root@slave1] ~$ kubeadm join 192.168.43.252:6443 --token orpuz8.j7vb3y83z6qfr15w --discovery-token-ca-cert-hash sha256:9608d18cd75ad1d9675036b8801d9a550d2a1ca3c4ddf0a5cc15d22e883badb7
[preflight] Running pre-flight checks
	[WARNING SystemVerification]: this Docker version is not on the list of validated versions: 18.06. Latest validated version: 18.09
[discovery] Trying to connect to API Server "192.168.43.252:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://192.168.43.252:6443"
[discovery] Requesting info from "https://192.168.43.252:6443" again to validate TLS against the pinned public key
[discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "192.168.43.252:6443"
[discovery] Successfully established connection with API Server "192.168.43.252:6443"
[join] Reading configuration from the cluster...
[join] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet] Downloading configuration for the kubelet from the "kubelet-config-1.14" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[tlsbootstrap] Waiting for the kubelet to perform the TLS Bootstrap...
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "slave1.hanli.com" as an annotation

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
```

查看节点是否加入集群，此时NotReady 是正常的，因为还没有安装网络
```
[root@master] ~$ kubectl get node
NAME               STATUS     ROLES    AGE     VERSION
master.hanli.com   NotReady   master   4m16s   v1.14.0
slave1.hanli.com   NotReady   <none>   2m54s   v1.14.0
slave2.hanli.com   NotReady   <none>   114s    v1.14.0
slave3.hanli.com   NotReady   <none>   2m40s   v1.14.0
```

查看pod，可以看到节点还没有Ready，dns的两个pod也不正常，还需要配置网络插件。

```
[root@master] ~$ kubectl get pod -n kube-system -o wide
NAME                                       READY   STATUS              RESTARTS   AGE     IP                NODE               NOMINATED NODE   READINESS GATES
coredns-86c58d9df4-r59rv                   0/1     ContainerCreating   0          4m24s   <none>            slave1.hanli.com   <none>           <none>
coredns-86c58d9df4-rbzx5                   0/1     ContainerCreating   0          4m25s   <none>            slave1.hanli.com   <none>           <none>
etcd-master.hanli.com                      1/1     Running             0          3m50s   192.168.43.252   master.hanli.com   <none>           <none>
kube-apiserver-master.hanli.com            1/1     Running             0          3m30s   192.168.43.252   master.hanli.com   <none>           <none>
kube-controller-manager-master.hanli.com   1/1     Running             2          3m56s   192.168.43.252   master.hanli.com   <none>           <none>
kube-proxy-4wrg5                           1/1     Running             0          4m24s   192.168.43.252   master.hanli.com   <none>           <none>
kube-proxy-6rlqz                           1/1     Running             0          2m1s    192.168.255.122   slave2.hanli.com   <none>           <none>
kube-proxy-jw7cj                           1/1     Running             0          2m25s   192.168.255.121   slave1.hanli.com   <none>           <none>
kube-proxy-zq442                           1/1     Running             0          2m19s   192.168.255.123   slave3.hanli.com   <none>           <none>
kube-scheduler-master.hanli.com            1/1     Running             2          3m52s   192.168.43.252   master.hanli.com   <none>           <none>
```

若加入集群口令忘记，重新生成:
```shell
[root@just-test ~]# kubeadm token create
89fnxq.jf31eeohb1mvhgya
[root@just-test ~]# kubeadm token generate
sz6374.4kallhypakfplrp6
[root@just-test ~]# kubeadm token list
TOKEN                     TTL       EXPIRES                     USAGES                   DESCRIPTION                                                EXTRA GROUPS
89fnxq.jf31eeohb1mvhgya   23h       2019-04-04T09:28:49+08:00   authentication,signing   <none>                                                     system:bootstrappers:kubeadm:default-node-token
9mm3u2.abk82donxijbreds   23h       2019-04-04T09:13:57+08:00   authentication,signing   The default bootstrap token generated by 'kubeadm init'.   system:bootstrappers:kubeadm:default-node-token
[root@just-test ~]# openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
a8d910eed14dc0a9e4c6f702847bb264638b7b372390b5e35fab02eb18db2507
新的token:
kubeadm join 192.168.43.252:6443 --token 89fnxq.jf31eeohb1mvhgya --discovery-token-ca-cert-hash sha256:a8d910eed14dc0a9e4c6f702847bb264638b7b372390b5e35fab02eb18db2507
```
如果初始化失败，可以重置下，再初始化
```shell
kubeadm reset
```
**错误分析：**
```
[root@just-test ~]# kubectl get pods
The connection to the server localhost:8080 was refused - did you specify the right host or port?
# 是不是前面的kubectl认证没有做
# 同样的没做的话，任何操作都会报错，如：
unable to recognize "https://docs.projectcalico.org/v3.6/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml": Get http://localhost:8080/api?timeout=32s: dial tcp 127.0.0.1:8080: connect: connection refused
```


### 3.2 安装network addon

要docker之间能互相通信需要做些配置，这里用calico来实现,详细可以参看官网很详尽。[Quickstart for Calico on Kubernetes](https://docs.projectcalico.org/v3.6/getting-started/kubernetes/)
```shell
# kubectl apply -f https://docs.projectcalico.org/v3.7/manifests/calico.yaml
[root@just-test ~]# kubectl apply -f kubernetes-datastore/calico-networking/1.7/calico.yaml
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.extensions/calico-node created
serviceaccount/calico-node created
deployment.extensions/calico-kube-controllers created
serviceaccount/calico-kube-controllers created

# 运行：watch kubectl get pods --all-namespaces
Every 2.0s: kubectl get pods --all-namespaces                                                                                                                                         Wed Apr  3 09:51:55 2019

NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-5cbcccc885-qgrv9   0/1     Running   0          82s
kube-system   calico-node-zgsxl                          0/1     Running   0          82s
kube-system   coredns-fb8b8dccf-7gxfl                    0/1     Running   0          37m
kube-system   coredns-fb8b8dccf-wd746                    0/1     Running   0          37m
kube-system   etcd-just-test                             0/1     Running   0          36m
kube-system   kube-apiserver-just-test                   0/1     Running   0          36m
kube-system   kube-controller-manager-just-test          0/1     Running   0          36m
kube-system   kube-proxy-wg8hr                           0/1     Running   0          37m
kube-system   kube-scheduler-just-test                   0/1     Running   0          36m
# 下载镜像，一段时间后就完成了
[root@just-test ~]#  kubectl get pods --all-namespaces
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-5cbcccc885-qgrv9   1/1     Running   0          97s
kube-system   calico-node-zgsxl                          1/1     Running   0          97s
kube-system   coredns-fb8b8dccf-7gxfl                    1/1     Running   0          37m
kube-system   coredns-fb8b8dccf-wd746                    1/1     Running   0          37m
kube-system   etcd-just-test                             1/1     Running   0          36m
kube-system   kube-apiserver-just-test                   1/1     Running   0          36m
kube-system   kube-controller-manager-just-test          1/1     Running   0          37m
kube-system   kube-proxy-wg8hr                           1/1     Running   0          37m
kube-system   kube-scheduler-just-test                   1/1     Running   0          37m
```
另外，删除master节点上的taints ，使其能够被调度pod。
```shell
[root@just-test ~]# kubectl taint nodes --all node-role.kubernetes.io/master-
node/just-test untainted
```
查看状态
```shell
[root@just-test ~]# kubectl get po --all-namespaces
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-5cbcccc885-qgrv9   1/1     Running   0          15m
kube-system   calico-node-zgsxl                          1/1     Running   0          15m
kube-system   coredns-fb8b8dccf-7gxfl                    1/1     Running   0          52m
kube-system   coredns-fb8b8dccf-wd746                    1/1     Running   0          52m
kube-system   etcd-just-test                             1/1     Running   0          50m
kube-system   kube-apiserver-just-test                   1/1     Running   0          51m
kube-system   kube-controller-manager-just-test          1/1     Running   0          51m
kube-system   kube-proxy-wg8hr                           1/1     Running   0          52m
kube-system   kube-scheduler-just-test                   1/1     Running   0          51m
# 节点状态
[root@just-test ~]# kubectl get nodes
NAME        STATUS   ROLES    AGE   VERSION
just-test   Ready    master   52m   v1.14.0
# 组件状态
[root@just-test ~]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
# 服务账户
[root@just-test ~]# kubectl get serviceaccount
NAME      SECRETS   AGE
default   1         56m
# 集群信息
[root@just-test ~]# kubectl cluster-info
Kubernetes master is running at https://192.168.43.252:6443
KubeDNS is running at https://192.168.43.252:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```
## 4. 测试集群功能是否正常
创建一个nginx的service试一下集群是否可用。
```shell
[root@just-test ~]# kubectl run nginx --replicas=2 --labels="run=load-balancer-example" --image=nginx --port=80 kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead. deployment.apps/nginx created
pod/nginx created
[root@just-test ~]# kubectl expose deployment nginx --type=NodePort --name=example-service
service/example-service exposed
[root@just-test ~]# kubectl describe service example-service
Name:                     example-service
Namespace:                default
Labels:                   run=load-balancer-example
Annotations:              <none>
Selector:                 run=load-balancer-example
Type:                     NodePort
IP:                       10.106.110.245
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  31130/TCP
Endpoints:                192.168.152.68:80,192.168.152.69:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```
服务状态
```shell
[root@just-test ~]# kubectl get service
NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
example-service   NodePort    10.106.110.245   <none>        80:31130/TCP   74s
kubernetes        ClusterIP   10.96.0.1        <none>        443/TCP        64m
```
服务pod
```shell
[root@just-test ~]# kubectl get pods
NAME                     READY   STATUS             RESTARTS   AGE
nginx                    0/1     CrashLoopBackOff   4          2m26s
nginx-7d645df49c-cmxc5   1/1     Running            0          4m1s
nginx-7d645df49c-mfbl5   1/1     Running            0          4m1s
```
访问服务
```shell
[root@just-test ~]# curl 192.168.43.252:31130
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
访问endpoint，与访问服务ip结果相同。这些 IP 只能在 Kubernetes Cluster 中的容器和节点访问。endpoint与service 之间有映射关系。service实际上是负载均衡着后端的endpoint。其原理是通过iptables实现的，这个不是本文内容，在此不谈。


整个部署过程是这样的：

① kubectl 发送部署请求到 API Server。

② API Server 通知 Controller Manager 创建一个 deployment 资源。

③ Scheduler 执行调度任务，将两个副本 Pod 分发到 node1 和 node2。

④ node1 和 node2 上的 kubelet 在各自的节点上创建并运行 Pod。


## 5. 配置dashboard

默认是没web界面的，可以在master机器上安装一个dashboard插件，实现通过web来管理

### 5.1 下载配置文件
Images
```shell
k8s.gcr.io/kubernetes-dashboard-arm64:v1.10.1
k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
k8s.gcr.io/kubernetes-dashboard-ppc64le:v1.10.1
k8s.gcr.io/kubernetes-dashboard-arm:v1.10.1
k8s.gcr.io/kubernetes-dashboard-s390x:v1.10.1
```
Installation
```
wget https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

编辑kubernetes-dashboard.yaml文件，添加type:  NodePort 和 dashboard端口，暴露Dashboard服务。注意这里添加行type:  NodePor和32666端口即可，其他配置不用改，大概位置在末尾的Dashboard Service的spec中，162行和166行，参考如下。
```shell
# ------------------- Dashboard Service ------------------- #
kind: Service
apiVersion: v1
metadata:
    labels:
    k8s-app: kubernetes-dashboard
    name: kubernetes-dashboard
    namespace: kube-system
spec:
    type: NodePort
    ports:
    - port: 443
        targetPort: 8443
        nodePort: 32666
    selector:
    k8s-app: kubernetes-dashboard
```
### 5.2 安装Dashboard插件
```shell
[root@just-test ~]# kubectl create -f kubernetes-dashboard.yaml
secret/kubernetes-dashboard-certs created
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created
```

### 5.3 授予Dashboard账户集群管理权限

需要一个管理集群admin的权限，新建kubernetes-dashboard-admin.rbac.yaml文件，内容如下
```shell
cat <<EOF > kubernetes-dashboard-admin.rbac.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
    labels:
    k8s-app: kubernetes-dashboard
    name: kubernetes-dashboard-admin
    namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
    name: kubernetes-dashboard-admin
    labels:
    k8s-app: kubernetes-dashboard
roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: cluster-admin
subjects:
- kind: ServiceAccount
    name: kubernetes-dashboard-admin
    namespace: kube-system
EOF
```
执行命令
```shell
kubectl create -f kubernetes-dashboard-admin.rbac.yaml
```
找到kubernete-dashboard-admin的token，用户登录使用
执行命令
```shell
[root@app5-185 kubernetes]# kubectl -n kube-system get secret | grep kubernetes-dashboard-admin
kubernetes-dashboard-admin-token-sn9p2           kubernetes.io/service-account-token   3         11s
```
可以看到名称是kubernetes-dashboard-admin-token-ddskx，使用该名称执行如下命令
```shell
[root@app5-185 kubernetes]# kubectl describe -n kube-system secret/kubernetes-dashboard-admin-token-sn9p2
Name:         kubernetes-dashboard-admin-token-sn9p2
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=kubernetes-dashboard-admin
                kubernetes.io/service-account.uid=2fc12ee2-6482-11e8-888b-000c29f785ee

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi10b2tlbi1zbjlwMiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjJmYzEyZWUyLTY0ODItMTFlOC04ODhiLTAwMGMyOWY3ODVlZSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTprdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiJ9.OYCJP1349iqnxh2tOav8BGhFg11KfahWUt1jItp0s0NlKKBZfynWC3_u77OGTZNEUY5Ajf7_8HOl3epAb1X3EvV6IBDYQczVlbrmB_3D9Ul9-KaUb3w7eGnBVeYyj_p2aAJrq0FffqfY4080UPsz3TwS43x67FGn_x4Bm_SDDO7ThUh-057XU5TQIqrptyJqDMnI4I8AkDpdgEn5xBRau4msz5VyOMwYm02PoZzaJ8TUiPw7gS9k8esIL4tQmI4rlK_sbfK_7-n9oYE5fmtZdAlBVddBL44nKgHROv5S2RDkZJPnLKuygsoOQvN7RRTOEugYnn4k_DMZGcwU7R-kHQ
```
记下这串token，等下登录使用，这个token默认是永久的。

### 5.4 找出Dashboard服务所在主机和端口
```shell
[root@app5-185 kubernetes]# kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                                       READY     STATUS    RESTARTS   AGE       IP                NODE
kube-system   calico-etcd-mfb8c                          1/1       Running   0          23m       192.168.39.227    app5-185
kube-system   calico-kube-controllers-5d74847676-wtmwl   1/1       Running   0          23m       192.168.39.227    app5-185
kube-system   calico-node-x7qvj                          2/2       Running   0          22m       192.168.110.182   app2-182
kube-system   calico-node-x9fmt                          2/2       Running   0          23m       192.168.39.227    app5-185
kube-system   coredns-7997f8864c-kpmm8                   1/1       Running   0          25m       192.168.153.178   app5-185
kube-system   coredns-7997f8864c-tpb69                   1/1       Running   0          25m       192.168.153.179   app5-185
kube-system   etcd-app5-185                              1/1       Running   0          24m       192.168.39.227    app5-185
kube-system   kube-apiserver-app5-185                    1/1       Running   0          24m       192.168.39.227    app5-185
kube-system   kube-controller-manager-app5-185           1/1       Running   0          24m       192.168.39.227    app5-185
kube-system   kube-proxy-mt9lm                           1/1       Running   0          22m       192.168.110.182   app2-182
kube-system   kube-proxy-rkrvr                           1/1       Running   0          25m       192.168.39.227    app5-185
kube-system   kube-scheduler-app5-185                    1/1       Running   0          24m       192.168.39.227    app5-185
kube-system   kubernetes-dashboard-7d5dcdb6d9-4fzzg      1/1       Running   0          21m       192.168.1.129     app2-182

[root@app5-185 kubernetes]# kubectl get svc -n kube-system
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
calico-etcd            ClusterIP   10.96.232.136   <none>        6666/TCP        15h
kube-dns               ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP   15h
kubernetes-dashboard   NodePort    10.110.101.45   <none>        443:32666/TCP   1m
```
可以看到`kubernetes-dashboard`运行在`app2-182`主机上，端口32666。

打开浏览器，访问`https://192.168.43.91:32666/`，证书会提示不可信，需要添加例外信任才能访问，

选择令牌，输入刚才的token即可进入

## 6. 部署heapster插件
```shell
mkdir -p ./heapster
cd ./heapster
wget https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/grafana.yaml
wget https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml
wget https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster.yaml
wget https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb.yaml
kubectl create -f ./
```
安装完成后，重新登录web界面即可看到。

## 7. 卸载
```shell
# kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
[root@app5-185 kubernetes]# kubectl drain app5-185 --delete-local-data --force --ignore-daemonsets
node "k8s-master" cordoned
WARNING: Ignoring DaemonSet-managed pods: calico-etcd-ffwff, calico-node-jsr48, kube-proxy-xfk5z; Deleting pods not managed by ReplicationController, ReplicaSet, Job, DaemonSet or StatefulSet: etcd-k8s-master, kube-apiserver-k8s-master, kube-controller-manager-k8s-master, kube-scheduler-k8s-master; Deleting pods with local storage: kubernetes-dashboard-5bd6f767c7-qsrpf
pod "httpd-app-5fbccd7c6c-d657q" evicted
pod "kubernetes-dashboard-5bd6f767c7-qsrpf" evicted
pod "httpd-app-5fbccd7c6c-7tzz7" evicted
pod "calico-kube-controllers-559b575f97-vt69n" evicted
pod "kube-dns-6f4fd4bdf-22jk4" evicted
node "k8s-master" drained

# kubectl delete node <node name>
[root@app5-185 kubernetes]# kubectl delete node k8s-master
node "k8s-master" deleted

# kubeadm reset
[root@k8s-master ~]# kubeadm reset
[preflight] Running pre-flight checks.
[reset] Stopping the kubelet service.
[reset] Unmounting mounted directories in "/var/lib/kubelet"
[reset] Removing kubernetes-managed containers.
[reset] Deleting contents of stateful directories: [/var/lib/kubelet /etc/cni/net.d /var/lib/dockershim /var/run/kubernetes /var/lib/etcd]
[reset] Deleting contents of config directories: [/etc/kubernetes/manifests /etc/kubernetes/pki]
[reset] Deleting files: [/etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf /etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf]
```

## 8. 离线安装和保存镜像

保存镜像
```shell
docker images | grep -v 'REPOSITORY' | awk '{print "docker save -o " $1".tar " $1":"$2}' | sed -e 's#\/#_#' | sh -x

[root@just-test ~]#  docker images | grep -v 'REPOSITORY' | awk '{print "docker save -o " $1".tar " $1":"$2}' | sed -e 's#\/#_#' | sh -x
+ docker save -o calico_node.tar calico/node:v3.6.1
+ docker save -o calico_cni.tar calico/cni:v3.6.1
+ docker save -o calico_kube-controllers.tar calico/kube-controllers:v3.6.1
+ docker save -o nginx.tar nginx:latest
+ docker save -o k8s.gcr.io_kube-proxy.tar k8s.gcr.io/kube-proxy:v1.14.0
+ docker save -o k8s.gcr.io_kube-apiserver.tar k8s.gcr.io/kube-apiserver:v1.14.0
+ docker save -o k8s.gcr.io_kube-controller-manager.tar k8s.gcr.io/kube-controller-manager:v1.14.0
+ docker save -o k8s.gcr.io_kube-scheduler.tar k8s.gcr.io/kube-scheduler:v1.14.0
+ docker save -o k8s.gcr.io_coredns.tar k8s.gcr.io/coredns:1.3.1
+ docker save -o k8s.gcr.io_kubernetes-dashboard-amd64.tar k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
+ docker save -o k8s.gcr.io_etcd.tar k8s.gcr.io/etcd:3.3.10
+ docker save -o k8s.gcr.io_pause.tar k8s.gcr.io/pause:3.1
[root@just-test ~/images]# ls
calico_cni.tar               calico_node.tar         k8s.gcr.io_etcd.tar            k8s.gcr.io_kube-controller-manager.tar  k8s.gcr.io_kubernetes-dashboard-amd64.tar  k8s.gcr.io_pause.tar
calico_kube-controllers.tar  k8s.gcr.io_coredns.tar  k8s.gcr.io_kube-apiserver.tar  k8s.gcr.io_kube-proxy.tar               k8s.gcr.io_kube-scheduler.tar              nginx.tar
```
安装镜像
```shell
ls | awk '{print "docker load -i " $1}' | sh -x

[root@just-test ~/images]# ls | awk '{print "docker load -i " $1}' | sh -x
+ docker load -i calico_cni.tar
Loaded image: calico/cni:v3.6.1
+ docker load -i calico_kube-controllers.tar
Loaded image: calico/kube-controllers:v3.6.1
+ docker load -i calico_node.tar
Loaded image: calico/node:v3.6.1
+ docker load -i k8s.gcr.io_coredns.tar
Loaded image: k8s.gcr.io/coredns:1.3.1
+ docker load -i k8s.gcr.io_etcd.tar
Loaded image: k8s.gcr.io/etcd:3.3.10
+ docker load -i k8s.gcr.io_kube-apiserver.tar
Loaded image: k8s.gcr.io/kube-apiserver:v1.14.0
+ docker load -i k8s.gcr.io_kube-controller-manager.tar
Loaded image: k8s.gcr.io/kube-controller-manager:v1.14.0
+ docker load -i k8s.gcr.io_kube-proxy.tar
Loaded image: k8s.gcr.io/kube-proxy:v1.14.0
+ docker load -i k8s.gcr.io_kubernetes-dashboard-amd64.tar
Loaded image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
+ docker load -i k8s.gcr.io_kube-scheduler.tar
Loaded image: k8s.gcr.io/kube-scheduler:v1.14.0
+ docker load -i k8s.gcr.io_pause.tar
Loaded image: k8s.gcr.io/pause:3.1
+ docker load -i nginx.tar
Loaded image: nginx:latest
```
