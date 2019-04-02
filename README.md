# kubeadm 安装部署 kubernetes 1.14.0

## 1. 安装准备

- 系统：centos 7

- 用户： root

- 机器规划:

| 角色 | 数量 | 配置 | 物理ip | hostname |
| :------: | :------: | :------: | :------: | :------: |
| master | 1 | 2核 1.5gG | 192.168.110.185 | k8s-master |
| node1 | 1 | 2核 1.5gG | 192.168.110.185 | k8s-node1 |
| node2 | 1 | 2核 1.5gG | 192.168.110.185 | k8s-node2 |

硬件配置参考：CPU 2核或以上，内存1.5GB或以上。
机器最好都在同一个局域网，在三台机器上都设置好hostname

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
systemctl disable --now dnsmasq

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
cat << EOL >> /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
Environment="KUBELET_EXTRA_ARGS=--fail-swap-on=false"
EOL
systemctl daemon-reload && systemctl restrt kubelet
```

### 1.4 调整内核参数,解决路由异常
```shell
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
modprobe br_netfilter
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
    "insecure-registries": ["harbor.iibu.com","192.168.39.0/24","192.168.43.0/24"],
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
Environment="NO_PROXY=localhost,127.0.0.1,127.0.0.0/8,harbor.iibu.com,registry.docker-cn.com,8trm4p9x.mirror.aliyuncs.com,registry.cn-hangzhou.aliyuncs.com,docker.mirrors.ustc.edu.cn,010a79c4.m.daocloud.io,192.168.39.0/24,192.168.43.0/24"
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
sed '5i Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"' -i /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
systemctl enable kubelet
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

通过kubeadm init命令来初始化，指定一下kubernetes版本，并设置一下pod-network-cidr。此过程，需要从google cloud下载镜像，前面步骤中已做了docker代理，可以翻墙下载加上下载镜像时间，根据网速不同，需要时间不等，完成后如下：
- apiserver-advertise-address该参数一般指定为haproxy+keepalived 的vip。
- pod-network-cidr 主要是在搭建pod network（calico）时候需要在init时候指定。
```shell
    [root@app5-185 kubernetes]# kubeadm init --kubernetes-version=v1.14.0 --feature-gates=CoreDNS=true --apiserver-advertise-address=192.168.39.227 --pod-network-cidr=10.244.0.0/16
    [init] Using Kubernetes version: v1.10.3
    [init] Using Authorization modes: [Node RBAC]
    [preflight] Running pre-flight checks.
            [WARNING SystemVerification]: docker version is greater than the most recently validated version. Docker version: 18.03.1-ce. Max validated version: 17.03
            [WARNING FileExisting-crictl]: crictl not found in system path
    Suggestion: go get github.com/kubernetes-incubator/cri-tools/cmd/crictl
    [preflight] Starting the kubelet service
    [certificates] Generated ca certificate and key.
    [certificates] Generated apiserver certificate and key.
    [certificates] apiserver serving cert is signed for DNS names [app5-185 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.39.227]
    [certificates] Generated apiserver-kubelet-client certificate and key.
    [certificates] Generated etcd/ca certificate and key.
    [certificates] Generated etcd/server certificate and key.
    [certificates] etcd/server serving cert is signed for DNS names [localhost] and IPs [127.0.0.1]
    [certificates] Generated etcd/peer certificate and key.
    [certificates] etcd/peer serving cert is signed for DNS names [app5-185] and IPs [192.168.39.227]
    [certificates] Generated etcd/healthcheck-client certificate and key.
    [certificates] Generated apiserver-etcd-client certificate and key.
    [certificates] Generated sa key and public key.
    [certificates] Generated front-proxy-ca certificate and key.
    [certificates] Generated front-proxy-client certificate and key.
    [certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
    [kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
    [kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
    [kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
    [kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
    [controlplane] Wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
    [controlplane] Wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
    [controlplane] Wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
    [etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
    [init] Waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests".
    [init] This might take a minute or longer if the control plane images have to be pulled.
    [apiclient] All control plane components are healthy after 18.001292 seconds
    [uploadconfig] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
    [markmaster] Will mark node app5-185 as master by adding a label and a taint
    [markmaster] Master app5-185 tainted and labelled with key/value: node-role.kubernetes.io/master=""
    [bootstraptoken] Using token: wsbo3p.dq1tng869a7eaq7a
    [bootstraptoken] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
    [bootstraptoken] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
    [bootstraptoken] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
    [bootstraptoken] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
    [addons] Applied essential addon: CoreDNS
    [addons] Applied essential addon: kube-proxy

    Your Kubernetes master has initialized successfully!

    To start using your cluster, you need to run the following as a regular user:

      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config

    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
      https://kubernetes.io/docs/concepts/cluster-administration/addons/

    You can now join any number of machines by running the following on each node
    as root:

      kubeadm join 192.168.39.227:6443 --token wsbo3p.dq1tng869a7eaq7a --discovery-token-ca-cert-hash sha256:bf98608e4365404e4d326fafd0ff94b3d86f1e7914ee773463fc8595002a2074

配置kubectl认证信息（Master节点操作）

    # 非root用户
    mkdir -p $HOME/.kubeyu
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    # root用户
    export KUBECONFIG=/etc/kubernetes/admin.conf
    echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
    source  ~/.bash_profile
```

若加入集群口令忘记，重新生成:
```shell
[root@app5-185 kubernetes]# kubeadm token create
bo7osg.2h8z3o2ohqbh7vf5
[root@app5-185 kubernetes]# kubeadm token generate
a4vdjw.vfnsh0n46qjm747p
[root@app5-185 kubernetes]# kubeadm token list
TOKEN                     TTL       EXPIRES                     USAGES                   DESCRIPTION                                                EXTRA GROUPS
y0wgl6.66k32lp5180prqru   23h       2018-04-04T11:50:19+02:00   authentication,signing   The default bootstrap token generated by 'kubeadm init'.   system:bootstrappers:kubeadm:default-node-token

# 获取ca证书sha256编码hash值
[root@app5-185 kubernetes]#  openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
d70f5276d6a8da16dc4c34abaabaaaaec16ec6e1c359722f8a0f2f76eaeb480b
新的token
    kubeadm join 173.249.43.139:6443 --token y0wgl6.66k32lp5180prqru --discovery-token-ca-cert-hash sha256:d70f5276d6a8da16dc4c34abaabaaaaec16ec6e1c359722f8a0f2f76eaeb480b
```
如果初始化失败，可以重置下，再初始化
```shell
    kubeadm reset
```


### 3.2 安装network addon

要docker之间能互相通信需要做些配置，这里用calico来实现
```shell
kubectl apply -f https://docs.projectcalico.org/v3.0/getting-started/kubernetes/installation/hosted/kubeadm/1.7/calico.yaml
```
也可以先下载下来再运行，如下：
```shell
[root@app5-185 kubernetes]# kubectl apply -f calico.yaml
configmap "calico-config" created
daemonset "calico-etcd" created
service "calico-etcd" created
daemonset "calico-node" created
deployment "calico-kube-controllers" created
clusterrolebinding "calico-cni-plugin" created
clusterrole "calico-cni-plugin" created
serviceaccount "calico-cni-plugin" created
clusterrolebinding "calico-kube-controllers" created
clusterrole "calico-kube-controllers" created
serviceaccount "calico-kube-controllers" created
```
安装完成后，检查下kube-dns是否安装成功。kube-dns比较重要，它负责整个集群的解析，要确保它正常运行。使用kubectl get pods --all-namespaces命令查看
```shell
# 首次大概所有服务起来需要10分钟（下载镜像慢，根据网速不同而定）

# 启动中。。。
[root@app5-185 kubernetes]# kubectl get po --all-namespaces
NAMESPACE     NAME                                       READY     STATUS              RESTARTS   AGE
kube-system   calico-etcd-fxtvz                          1/1       Running             0          53s
kube-system   calico-kube-controllers-5d74847676-wn9gl   1/1       Running             0          52s
kube-system   calico-node-82xtt                          2/2       Running             0          52s
kube-system   coredns-7997f8864c-vv7d4                   0/1       ContainerCreating   0          6m
kube-system   coredns-7997f8864c-zvz5b                   0/1       ContainerCreating   0          6m
kube-system   etcd-app5-185                              1/1       Running             0          6m
kube-system   kube-apiserver-app5-185                    1/1       Running             0          5m
kube-system   kube-controller-manager-app5-185           1/1       Running             0          5m
kube-system   kube-proxy-6lfzm                           1/1       Running             0          6m
kube-system   kube-scheduler-app5-185                    1/1       Running             0          5m

# 启动完成
[root@app5-185 kubernetes]# kubectl get po --all-namespaces
NAMESPACE     NAME                                       READY     STATUS    RESTARTS   AGE
kube-system   calico-etcd-fxtvz                          1/1       Running   0          1m
kube-system   calico-kube-controllers-5d74847676-wn9gl   1/1       Running   0          1m
kube-system   calico-node-82xtt                          2/2       Running   0          1m
kube-system   coredns-7997f8864c-vv7d4                   1/1       Running   0          7m
kube-system   coredns-7997f8864c-zvz5b                   1/1       Running   0          7m
kube-system   etcd-app5-185                              1/1       Running   0          6m
kube-system   kube-apiserver-app5-185                    1/1       Running   0          6m
kube-system   kube-controller-manager-app5-185           1/1       Running   0          6m
kube-system   kube-proxy-6lfzm                           1/1       Running   0          7m
kube-system   kube-scheduler-app5-185                    1/1       Running   0          6m

[root@app5-185 kubernetes]# kubectl get nodes
NAME       STATUS    ROLES     AGE       VERSION
app5-185   Ready     master    8m        v1.10.3

[root@app5-185 kubernetes]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health": "true"}
```

## 4. node加入集群

登录192.168.110.182，安装kubernetes环境，并下载好相关镜像，输入token加入集群（切记时间要同步，否者证书验不过）

    kubeadm join 192.168.39.227:6443 --token u465qx.fh2um5oh5d44rxae --discovery-token-ca-cert-hash sha256:0ad9b1bb7e0f3205d61bf4e5a10f2f91b204b06c3780facb0f5cceecda4822dd
    [root@app5-185 kubernetes]# kubectl get nodes
    NAME       STATUS     ROLES     AGE       VERSION
    app2-182   Ready     <none>     1m        v1.10.3
    app5-185   Ready      master    10m       v1.10.3
    # 删除后重新加入，先运行
    kubeadm reset

## 5. 配置dashboard

默认是没web界面的，可以在master机器上安装一个dashboard插件，实现通过web来管理

### 5.1 下载配置文件
```shell
wget https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
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
kubectl create -f kubernetes-dashboard.yaml
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

## 6. 测试集群

在master节点上发起个创建应用请求, 这里我们创建个名为httpd-app的应用，镜像为httpd，有两个副本pod允许master创建pod

默认情况下，Master节点不参与工作负载，但如果希望安装出一个All-In-One的k8s环境，则可以执行以下命令，让Master节点也成为一个Node节点：
```shell
kubectl taint nodes --all node-role.kubernetes.io/master-
```
创建一个httpd服务
```shell
kubectl run httpd-app --image=httpd --replicas=2
```
查询结果
```shell
[root@app5-185 kubernetes]# kubectl get pod
NAME                         READY     STATUS    RESTARTS   AGE
httpd-app-5fbccd7c6c-4wh6b   0/1       Pending   0          34m
httpd-app-5fbccd7c6c-75ck4   0/1       Pending   0          34m
```
删除服务
```shell
[root@k8s-master ~]# kubectl delete deployment httpd-app
deployment "httpd-app" deleted
```
## 7. 部署heapster插件
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

## 8. 卸载
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

## 9. 离线安装镜像

保存镜像
```shell
docker save -o kube-controller-manager-amd64.tar gcr.io/google_containers/kube-controller-manager-amd64:v1.9.6
docker save -o kube-apiserver-amd64.tar gcr.io/google_containers/kube-apiserver-amd64:v1.9.6
docker save -o kube-proxy-amd64.tar gcr.io/google_containers/kube-proxy-amd64:v1.9.6
docker save -o kube-scheduler-amd64.tar gcr.io/google_containers/kube-scheduler-amd64:v1.9.6
docker save -o calico_node.tar quay.io/calico/node:v3.0.3
docker save -o kube-controllers.tar quay.io/calico/kube-controllers:v2.0.1
docker save -o calico_cni.tar quay.io/calico/cni:v2.0.1
docker save -o kubernetes-dashboard-amd64.tar k8s.gcr.io/kubernetes-dashboard-amd64:v1.8.3
docker save -o etcd-amd64.tar gcr.io/google_containers/etcd-amd64:3.1.11
docker save -o kube-addon-manager.tar gcr.io/google-containers/kube-addon-manager:v6.5
docker save -o storage-provisioner.tar gcr.io/k8s-minikube/storage-provisioner:v1.8.0
docker save -o k8s-dns-sidecar-amd64_1.14.7.tar gcr.io/google_containers/k8s-dns-sidecar-amd64:1.14.7
docker save -o k8s-dns-kube-dns-amd64.tar gcr.io/google_containers/k8s-dns-kube-dns-amd64:1.14.7
docker save -o k8s-dns-dnsmasq-nanny-amd64.tar gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.7
docker save -o k8s-dns-sidecar-amd64_1.14.5.tar k8s.gcr.io/k8s-dns-sidecar-amd64:1.14.5
docker save -o k8s-dns-kube-dns-amd64.tar k8s.gcr.io/k8s-dns-kube-dns-amd64:1.14.5
docker save -o k8s-dns-dnsmasq-nanny-amd64.tar k8s.gcr.io/k8s-dns-dnsmasq-nanny-amd64:1.14.5
docker save -o etcd.tar quay.io/coreos/etcd:v3.1.10
docker save -o google_containers_pause-amd64.tar gcr.io/google_containers/pause-amd64:3.0
docker save -o pause-amd64.tar k8s.gcr.io/pause-amd64:3.0
```
安装镜像
```shell
docker load -i kube-controller-manager-amd64.tar gcr.io/google_containers/kube-controller-manager-amd64:v1.9.6
docker load -i kube-apiserver-amd64.tar gcr.io/google_containers/kube-apiserver-amd64:v1.9.6
docker load -i kube-proxy-amd64.tar gcr.io/google_containers/kube-proxy-amd64:v1.9.6
docker load -i kube-scheduler-amd64.tar gcr.io/google_containers/kube-scheduler-amd64:v1.9.6
docker load -i calico_node.tar quay.io/calico/node:v3.0.3
docker load -i kube-controllers.tar quay.io/calico/kube-controllers:v2.0.1
docker load -i calico_cni.tar quay.io/calico/cni:v2.0.1
docker load -i kubernetes-dashboard-amd64.tar k8s.gcr.io/kubernetes-dashboard-amd64:v1.8.3
docker load -i etcd-amd64.tar gcr.io/google_containers/etcd-amd64:3.1.11
docker load -i kube-addon-manager.tar gcr.io/google-containers/kube-addon-manager:v6.5
docker load -i storage-provisioner.tar gcr.io/k8s-minikube/storage-provisioner:v1.8.0
docker load -i k8s-dns-sidecar-amd64_1.14.7.tar gcr.io/google_containers/k8s-dns-sidecar-amd64:1.14.7
docker load -i k8s-dns-kube-dns-amd64.tar gcr.io/google_containers/k8s-dns-kube-dns-amd64:1.14.7
docker load -i k8s-dns-dnsmasq-nanny-amd64.tar gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.7
docker load -i k8s-dns-sidecar-amd64_1.14.5.tar k8s.gcr.io/k8s-dns-sidecar-amd64:1.14.5
docker load -i k8s-dns-kube-dns-amd64.tar k8s.gcr.io/k8s-dns-kube-dns-amd64:1.14.5
docker load -i k8s-dns-dnsmasq-nanny-amd64.tar k8s.gcr.io/k8s-dns-dnsmasq-nanny-amd64:1.14.5
docker load -i etcd.tar quay.io/coreos/etcd:v3.1.10
docker load -i google_containers_pause-amd64.tar gcr.io/google_containers/pause-amd64:3.0
docker load -i pause-amd64.tar k8s.gcr.io/pause-amd64:3.0
```
