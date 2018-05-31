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
