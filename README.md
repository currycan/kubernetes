kubeadm 安装部署 kubernetes 1.10.3

0 安装准备

系统：centos 7  

用户： root 

机器规划：

  角色    	数量  	配置   	物理ip           	hostname  
  master	1   	4核 6G	192.168.110.185	k8s-master
  node  	1   	4核 6G	               	k8s-node1 
  node  	1   	4核 6G	               	k8s-node2 

硬件配置参考：CPU 2核或以上，内存2GB或以上。 

 机器最好都在同一个局域网，在三台机器上都设置好hostname

0.1校准时区

    # timedatectl list-timezones
    timedatectl set-timezone Asia/Shanghai
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
    yum install -y ntpdate
    systemctl start ntpdate.service
    systemctl enable ntpdate.service
    echo '*/30 * * * * /usr/sbin/ntpdate cn.pool.ntp.org >/dev/null 2>&1' > /tmp/crontab2.tmp
    crontab /tmp/crontab2.tmp
    systemctl start ntpdate.service
    
    echo "* soft nofile 65536" >> /etc/security/limits.conf
    echo "* hard nofile 65536" >> /etc/security/limits.conf
    echo "* soft nproc 65536"  >> /etc/security/limits.conf
    echo "* hard nproc 65536"  >> /etc/security/limits.conf
    echo "* soft  memlock  unlimited"  >> /etc/security/limits.conf
    echo "* hard memlock  unlimited"  >> /etc/security/limits.conf

0.2 系统设置 

    关闭防火墙，swap，selinux

    systemctl stop firewalld && systemctl disable firewalld
    swapoff -a && sysctl -w vm.swappiness=0
    rm -rf $(swapon -s | awk 'NR>1{print $1}')
    sed -i 's/.*swap.*/#&/' /etc/fstab
    setenforce 0
    sed -i /etc/selinux/config -r -e 's/^SELINUX=.*/SELINUX=disabled/g'
    reboot

因为测试主机上还运行其他服务，关闭swap可能会对其他服务产生影响，所以建议修改kubelet的启动参数

    --fail-swap-on=false

 去掉这个限制。修改 

    /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

加入：

    # 已经安装好kubernetes环境
    cat << EOL >> /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
    Environment="KUBELET_EXTRA_ARGS=--fail-swap-on=false"
    EOL
    systemctl daemon-reload && systemctl restrt kubelet

0.3 调整内核参数,解决路由异常

    cat <<EOF >  /etc/sysctl.d/k8s.conf
    net.ipv4.ip_forward = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    EOF
    modprobe br_netfilter
    sysctl -p /etc/sysctl.d/k8s.conf

一 安装

1.0 安装docker 

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
    
    rm -rf /var/lib/docker
    
    yum install -y yum-utils device-mapper-persistent-data lvm2
    yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
    yum list docker-ce --showduplicates | sort -r
    yum install -y docker-ce
    
    systemctl enable docker
    systemctl daemon-reload && systemctl start docker
    systemctl status docker -l
    
    cat << EOF > /etc/docker/daemon.json
    {
        "registry-mirrors": [
            "https://registry.docker-cn.com",
            "https://8trm4p9x.mirror.aliyuncs.com",
            "http://010a79c4.m.daocloud.io",
            "https://docker.mirrors.ustc.edu.cn/"
        ],
        "insecure-registries": ["192.168.110.0/24","192.168.39.0/24","192.168.43.0/24","harbor.iibu.com"],
        "storage-driver": "overlay2",
        "exec-opts": ["native.cgroupdriver=cgroupfs"],
        "dns": ["8.8.8.8", "8.8.4.4"],
        "max-concurrent-downloads": 10,
        "log-driver": "json-file",
        "log-level": "warn",
        "log-opts": {
           "max-size": "10m",
           "max-file": "3"
        }
    }
    EOF
    systemctl daemon-reload && systemctl restart docker

设置docker代理配置，http代理(如果用代理就可以直接从gcr.io直接pull镜像)

    mkdir -p /etc/systemd/system/docker.service.d
    cat <<EOF > /etc/systemd/system/docker.service.d/http-proxy.conf
    [Service]
    Environment="HTTP_PROXY=http://currycan:shachao123321@192.168.39.43:1080"
    Environment="HTTPS_PROXY=http://currycan:shachao123321@192.168.39.43:1080"
    Environment="NO_PROXY=localhost,127.0.0.1,127.0.0.0/8,harbor.iibu.com,registry.docker-cn.com,8trm4p9x.mirror.aliyuncs.com,registry.cn-hangzhou.aliyuncs.com,docker.mirrors.ustc.edu.cn,010a79c4.m.daocloud.io,192.168.110.0/24,192.168.39.0/24,192.168.43.0/24"
    EOF
    systemctl daemon-reload && systemctl restart docker
    systemctl show --property=Environment docker

1.1 安装kubeadm, kubelet和kubectl

配置kubernetes源

    cat <<EOF > /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
    EOF
    yum install -y kubeadm kubelet kubectl
    sed -i "s/cgroup-driver=systemd/cgroup-driver=cgroupfs/g" /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
    systemctl enable kubelet.service && systemctl start kubelet.service

1.2 下载gcr.io镜像 

下载镜像并push到harbor私有仓

    cat <<EOF > k8s-images.sh
    #!/bin/bash
    set -o errexit
    # set -o nounset
    set -o pipefail
    
    KUBE_VERSION=v1.10.3
    KUBE_PAUSE_VERSION=3.1
    ETCD_VERSION=3.1.12
    DNS_VERSION=1.14.8
    DASHBOARD=v1.8.3
    HEAPSTER=v1.5.3
    HS_INFLUXDB=v1.3.3
    HS_GRAFANA=v4.4.3
    CALICO_NODE=v3.0.7
    CALICO_KUBE_CONTROLLERS=v2.0.4
    CALICO_CNI=v2.0.5
    COREOS_ETCD=v3.1.10
    
    GCR_URL=k8s.gcr.io
    CALICO_URL=quay.io/calico
    COREOS_URL=quay.io/coreos
    HARBOR_URL=harbor.iibu.com/kubernetes
    
    kube_images=(
        kube-proxy-amd64:\${KUBE_VERSION}
        kube-apiserver-amd64:\${KUBE_VERSION}
        kube-scheduler-amd64:\${KUBE_VERSION}
        kube-controller-manager-amd64:\${KUBE_VERSION}
        pause-amd64:\${KUBE_PAUSE_VERSION}
        etcd-amd64:\${ETCD_VERSION}
        k8s-dns-dnsmasq-nanny-amd64:\${DNS_VERSION}
        k8s-dns-sidecar-amd64:\${DNS_VERSION}
        k8s-dns-kube-dns-amd64:\${DNS_VERSION}
        kubernetes-dashboard-amd64:\${DASHBOARD}
        heapster-amd64:\${HEAPSTER}
        heapster-influxdb-amd64:\${HS_INFLUXDB}
        heapster-grafana-amd64:\${HS_GRAFANA}
    )
    
    calicao_images=(
        node:\${CALICO_NODE}
        kube-controllers:\${CALICO_KUBE_CONTROLLERS}
        cni:\${CALICO_CNI}
    )
    
    push(){
      for imageName in \${kube_images[@]} ; do
        docker pull \$GCR_URL/\$imageName
        docker tag \$GCR_URL/\$imageName \$HARBOR_URL/\$imageName
        docker push \$HARBOR_URL/\$imageName
        # docker rmi \$GCR_URL/\$imageName
      done
      for imageName in \${calicao_images[@]} ; do
        docker pull \$CALICO_URL/\$imageName
        docker tag \$CALICO_URL/\$imageName \$HARBOR_URL/\$imageName
        docker push \$HARBOR_URL/\$imageName
        # docker rmi \$GCR_URL/\$imageName
      done
      docker pull \$COREOS_URL/etcd:\${COREOS_ETCD}
      docker tag \$COREOS_URL/etcd:\${COREOS_ETCD} \$HARBOR_URL/etcd:\${COREOS_ETCD}
      docker push \$HARBOR_URL/etcd:\${COREOS_ETCD}
    }
    
    pull(){
      for imageName in \${kube_images[@]} ; do
        docker pull \$HARBOR_URL/\$imageName
        docker tag \$HARBOR_URL/\$imageName \$GCR_URL/\$imageName
      done
      for imageName in \${calicao_images[@]} ; do
        docker pull \$HARBOR_URL/\$imageName
        docker tag \$HARBOR_URL/\$imageName \$CALICO_URL/\$imageName
      done
      docker pull \$HARBOR_URL/etcd:\${COREOS_ETCD}
      docker tag \$HARBOR_URL/etcd:\${COREOS_ETCD} \$COREOS_URL/etcd:\${COREOS_ETCD}
    }
    
    untag(){
      for imageName in \${kube_images[@]} ; do
        docker rmi \$HARBOR_URL/\$imageName
      done
      for imageName in \${calicao_images[@]} ; do
        docker rmi \$HARBOR_URL/\$imageName
      done
      docker rmi \$HARBOR_URL/etcd:\${COREOS_ETCD}
    }
    
    if [ -n "\$1" ];then
      if [ \$1 = 'push' ];then
        push
      elif [ \$1 = 'pull' ]; then
        pull
      elif [ \$1 = 'untag' ]; then
        untag
      elif [[ \$1 = '-h' ]] || [[ \$1 = 'help' ]];then
        echo "Usage: \$0 COMMAND"
        echo 'Commands:'
        echo '  push:    push imagses from internet repository to local harbor repository'
        echo '  pull:    pull imagses from local harbor repository, and then tag it to internet repository'
        echo '  untag:   remove the local harbor repository tag of imagses'
        echo '  help:    help infomations'
      else
        echo "Error input command,please type '\$0 help' to get help infomations"
      fi
    else
      echo "Usage: \$0 COMMAND"
      echo 'Commands:'
      echo '  push:    push imagses from internet repository to local harbor repository'
      echo '  pull:    pull imagses from local harbor repository, and then tag it to internet repository'
      echo '  untag:   remove the local harbor repository tag of imagses'
      echo '  help:    help infomations'
    fi
    
    EOF
    chmod +x k8s-images.sh
    ./k8s-images.sh

二 在master上配置

2.0 初始化K8S

通过kubeadm init命令来初始化，指定一下kubernetes版本，并设置一下pod-network-cidr。

此过程，需要从google cloud下载镜像，前面步骤中已做了docker代理，可以翻墙下载加上下载镜像时间，根据网速不同，需要时间不等，完成后如下：

- apiserver-advertise-address该参数一般指定为haproxy+keepalived 的vip。
- pod-network-cidr 主要是在搭建pod network（calico）时候需要在init时候指定。

    [root@app5-185 kubernetes]# kubeadm init --kubernetes-version=v1.10.3 --feature-gates=CoreDNS=true --apiserver-advertise-address=192.168.39.227 --pod-network-cidr=10.244.0.0/16
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

若加入集群口令忘记，重新生成:

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

如果初始化失败，可以重置下，再初始化

    kubeadm reset



2.2 安装network addon 

要docker之间能互相通信需要做些配置，这里用calico来实现

    kubectl apply -f https://docs.projectcalico.org/v3.0/getting-started/kubernetes/installation/hosted/kubeadm/1.7/calico.yaml

也可以先下载下来再运行，如下：

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

安装完成后，检查下kube-dns是否安装成功。kube-dns比较重要，它负责整个集群的解析，要确保它正常运行。使用kubectl get pods --all-namespaces命令查看

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

三 node加入集群

登录192.168.110.182，安装kubernetes环境，并下载好相关镜像，输入token加入集群（切记时间要同步，否者证书验不过）

    kubeadm join 192.168.39.227:6443 --token u465qx.fh2um5oh5d44rxae --discovery-token-ca-cert-hash sha256:0ad9b1bb7e0f3205d61bf4e5a10f2f91b204b06c3780facb0f5cceecda4822dd
    [root@app5-185 kubernetes]# kubectl get nodes
    NAME       STATUS     ROLES     AGE       VERSION
    app2-182   Ready     <none>     1m        v1.10.3
    app5-185   Ready      master    10m       v1.10.3
    # 删除后重新加入，先运行
    kubeadm reset

四 配置dashboard

默认是没web界面的，可以在master机器上安装一个dashboard插件，实现通过web来管理

4.0 下载配置文件

    wget https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml

编辑kubernetes-dashboard.yaml文件，添加type:  NodePort 和 dashboard端口，暴露Dashboard服务。注意这里添加行type:  NodePor和32666端口即可，其他配置不用改，大概位置在末尾的Dashboard Service的spec中，162行和166行，参考如下。

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



4.1 安装Dashboard插件

     kubectl create -f kubernetes-dashboard.yaml

4.2 授予Dashboard账户集群管理权限

需要一个管理集群admin的权限，新建kubernetes-dashboard-admin.rbac.yaml文件，内容如下

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

执行命令

     kubectl create -f kubernetes-dashboard-admin.rbac.yaml

找到kubernete-dashboard-admin的token，用户登录使用 

 执行命令

    [root@app5-185 kubernetes]# kubectl -n kube-system get secret | grep kubernetes-dashboard-admin
    kubernetes-dashboard-admin-token-sn9p2           kubernetes.io/service-account-token   3         11s

可以看到名称是kubernetes-dashboard-admin-token-ddskx，使用该名称执行如下命令

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

记下这串token，等下登录使用，这个token默认是永久的。

4.3 找出Dashboard服务所在主机和端口

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

可以看到kubernetes-dashboard运行在app2-182主机上，端口32666。 

打开浏览器，访问https://192.168.43.91:32666/，证书会提示不可信，需要添加例外信任才能访问，

选择令牌，输入刚才的token即可进入 

测试集群



在master节点上发起个创建应用请求

这里我们创建个名为httpd-app的应用，镜像为httpd，有两个副本pod

允许master创建pod

默认情况下，Master节点不参与工作负载，但如果希望安装出一个All-In-One的k8s环境，则可以执行以下命令，让Master节点也成为一个Node节点：

    kubectl taint nodes --all node-role.kubernetes.io/master-

创建一个httpd服务

     kubectl run httpd-app --image=httpd --replicas=2

查询结果

 

    [root@app5-185 kubernetes]# kubectl get pod
    NAME                         READY     STATUS    RESTARTS   AGE
    httpd-app-5fbccd7c6c-4wh6b   0/1       Pending   0          34m
    httpd-app-5fbccd7c6c-75ck4   0/1       Pending   0          34m

删除服务

    [root@k8s-master ~]# kubectl delete deployment httpd-app
    deployment "httpd-app" deleted

五 部署heapster插件

    mkdir -p ./heapster
    cd ./heapster
    wget https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/grafana.yaml
    wget https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml
    wget https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster.yaml
    wget https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb.yaml
    kubectl create -f ./

安装完成后，重新登录即可看到。

六 卸载

    ## kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
    [root@app5-185 kubernetes]# kubectl drain app5-185 --delete-local-data --force --ignore-daemonsets
    node "k8s-master" cordoned
    WARNING: Ignoring DaemonSet-managed pods: calico-etcd-ffwff, calico-node-jsr48, kube-proxy-xfk5z; Deleting pods not managed by ReplicationController, ReplicaSet, Job, DaemonSet or StatefulSet: etcd-k8s-master, kube-apiserver-k8s-master, kube-controller-manager-k8s-master, kube-scheduler-k8s-master; Deleting pods with local storage: kubernetes-dashboard-5bd6f767c7-qsrpf
    pod "httpd-app-5fbccd7c6c-d657q" evicted
    pod "kubernetes-dashboard-5bd6f767c7-qsrpf" evicted
    pod "httpd-app-5fbccd7c6c-7tzz7" evicted
    pod "calico-kube-controllers-559b575f97-vt69n" evicted
    pod "kube-dns-6f4fd4bdf-22jk4" evicted
    node "k8s-master" drained
    
    ## kubectl delete node <node name>
    [root@app5-185 kubernetes]# kubectl delete node k8s-master
    node "k8s-master" deleted
    
    ## kubeadm reset
    [root@k8s-master ~]# kubeadm reset
    [preflight] Running pre-flight checks.
    [reset] Stopping the kubelet service.
    [reset] Unmounting mounted directories in "/var/lib/kubelet"
    [reset] Removing kubernetes-managed containers.
    [reset] Deleting contents of stateful directories: [/var/lib/kubelet /etc/cni/net.d /var/lib/dockershim /var/run/kubernetes /var/lib/etcd]
    [reset] Deleting contents of config directories: [/etc/kubernetes/manifests /etc/kubernetes/pki]
    [reset] Deleting files: [/etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf /etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf]



七 离线安装

保存镜像

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

安装镜像

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


