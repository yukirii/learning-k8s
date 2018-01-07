## etcd

etcd v3.2.12
https://github.com/coreos/etcd/releases/tag/v3.2.12

```bash
wget https://github.com/coreos/etcd/releases/download/v3.2.12/etcd-v3.2.12-linux-amd64.tar.gz
tar zxvf etcd-v3.2.12-linux-amd64.tar.gz

cd etcd-v3.2.12-linux-amd64
sudo mv etcd /usr/local/bin
sudo mv etcdctl /usr/local/bin

sudo mkdir -p /var/lib/etcd
```

env 用ファイルと systemd サービスファイルの作成

```bash
ETCD1_NAME=kirii-k8s01
ETCD2_NAME=kirii-k8s02
ETCD3_NAME=kirii-k8s03
ETCD1_IP=$(getent ahostsv4 ${ETCD1_NAME} | tail -1 | awk '{print $1}')
ETCD2_IP=$(getent ahostsv4 ${ETCD2_NAME} | tail -1 | awk '{print $1}')
ETCD3_IP=$(getent ahostsv4 ${ETCD3_NAME} | tail -1 | awk '{print $1}')
INTERNAL_IP=$(hostname -I | awk '{print $1}')
ETCD_CLUSTER_TOKEN=k8s_etcd

cat > /etc/etcd.env << EOF
ETCD_DATA_DIR=/var/lib/etcd
ETCD_ADVERTISE_CLIENT_URLS=http://${INTERNAL_IP}:2379
ETCD_INITIAL_ADVERTISE_PEER_URLS=http://${INTERNAL_IP}:2380
ETCD_LISTEN_CLIENT_URLS=http://${INTERNAL_IP}:2379,http://127.0.0.1:2379
ETCD_ELECTION_TIMEOUT=5000
ETCD_HEARTBEAT_INTERVAL=250
ETCD_INITIAL_CLUSTER_TOKEN=${ETCD_CLUSTER_TOKEN}
ETCD_LISTEN_PEER_URLS=http://${INTERNAL_IP}:2380
ETCD_NAME=$(hostname --short)
ETCD_PROXY=off
ETCD_AUTO_COMPACTION_RETENTION=8
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_INITIAL_CLUSTER=${ETCD1_NAME}=http://${ETCD1_IP}:2380,${ETCD2_NAME}=http://${ETCD2_IP}:2380,${ETCD3_NAME}=http://${ETCD3_IP}:2380
EOF
```

```bash
cat > /etc/systemd/system/etcd.service << EOF
[Unit]
Description=etcd
Documentation=https://coreos.com/etcd/docs/latest/
After=network.target

[Service]
User=root
PermissionsStartOnly=true
EnvironmentFile=/etc/etcd.env
ExecStart=/usr/local/bin/etcd
Restart=always
RestartSec=15s
TimeoutStartSec=30s

[Install]
WantedBy=multi-user.target
EOF
```

起動

```
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
systemctl status etcd -l
```

動作確認

```bash
etcdctl cluster-health
etcdctl member list
```

---

## k8s master

* kube-apiserver
* kube-controller-manager
* kube-scheduler

```bash
sudo su -

TAG=v1.8.2
URL=https://storage.googleapis.com/kubernetes-release/release/$TAG/bin/linux/amd64

curl -# -L -o /usr/bin/kube-apiserver $URL/kube-apiserver
curl -# -L -o /usr/bin/kube-controller-manager $URL/kube-controller-manager
curl -# -L -o /usr/bin/kube-scheduler $URL/kube-scheduler
curl -# -L -o /usr/bin/kube-proxy $URL/kube-proxy
curl -# -L -o /usr/bin/kubelet $URL/kubelet
curl -# -L -o /usr/bin/kubectl $URL/kubectl

chmod +x -- /usr/bin/kube*
mkdir -p /var/lib/{kubelet,kube-proxy}
```

### kube-apiserver

```bash
MASTER1_NAME=kirii-k8s01
MASTER2_NAME=kirii-k8s02
MASTER3_NAME=kirii-k8s03
MASTER1_IP=$(getent ahostsv4 ${MASTER1_NAME} | tail -1 | awk '{print $1}')
MASTER2_IP=$(getent ahostsv4 ${MASTER2_NAME} | tail -1 | awk '{print $1}')
MASTER3_IP=$(getent ahostsv4 ${MASTER3_NAME} | tail -1 | awk '{print $1}')
INTERNAL_IP=$(hostname -I | awk '{print $1}')
SERVICE_CLUSTER_IP_RANGE="10.233.0.0/12"

cat > /etc/systemd/system/kube-apiserver.service << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/bin/kube-apiserver \\
  --apiserver-count=3 \\
  --allow-privileged=true \\
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --bind-address=0.0.0.0 \\
  --advertise-address=${INTERNAL_IP} \\
  --insecure-port=8080 \\
  --insecure-bind-address=127.0.0.1 \\
  --etcd-servers=http://${MASTER1_IP}:2379,http://${MASTER2_IP}:2379,http://${MASTER2_IP}:2379 \\
  --service-cluster-ip-range=${SERVICE_CLUSTER_IP_RANGE} \\
  --service-node-port-range=30000-32767 \\
  --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname \\
  --v=2
Restart=always
RestartSec=15s

[Install]
WantedBy=multi-user.target
EOF
```

起動

```bash
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver -l
```

### kube-controller-manager

```bash
INTERNAL_IP=$(hostname -I | awk '{print $1}')
KUBERNETES_PUBLIC_ADDRESS=$INTERNAL_IP

CLUSTER_NAME="default"
KCONFIG=controller-manager.kubeconfig
KUSER="system:kube-controller-manager"
KCERT=sa

cd /etc/kubernetes/

kubectl config set-cluster ${CLUSTER_NAME} \
  --server=http://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=${KCONFIG}

kubectl config set-context ${KUSER}@${CLUSTER_NAME} \
  --cluster=${CLUSTER_NAME} \
  --user=${KUSER} \
  --kubeconfig=${KCONFIG}

kubectl config use-context ${KUSER}@${CLUSTER_NAME} --kubeconfig=${KCONFIG}
kubectl config view --kubeconfig=${KCONFIG}
```

```bash
CLUSTER_CIDR="10.233.0.0/16"
SERVICE_CLUSTER_IP_RANGE="10.233.0.0/12"
CLUSTER_NAME="default"

cat > /etc/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/bin/kube-controller-manager \\
  --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig \\
  --address=127.0.0.1 \\
  --master=http://127.0.0.1:8080 \\
  --leader-elect=true \\
  --cluster-cidr=${CLUSTER_CIDR} \\
  --cluster-name=${CLUSTER_NAME} \\
  --service-cluster-ip-range=${SERVICE_CLUSTER_IP_RANGE} \\
  --v=2
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF
```

起動

```bash
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
systemctl status kube-controller-manager -l
```

### kube-scheduler

```bash
INTERNAL_IP=$(hostname -I | awk '{print $1}')
KUBERNETES_PUBLIC_ADDRESS=$INTERNAL_IP

CLUSTER_NAME="default"
KCONFIG=scheduler.kubeconfig
KUSER="system:kube-scheduler"

cd /etc/kubernetes/

kubectl config set-cluster ${CLUSTER_NAME} \
  --server=http://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=${KCONFIG}

kubectl config set-context ${KUSER}@${CLUSTER_NAME} \
  --cluster=${CLUSTER_NAME} \
  --user=${KUSER} \
  --kubeconfig=${KCONFIG}

kubectl config use-context ${KUSER}@${CLUSTER_NAME} --kubeconfig=${KCONFIG}
kubectl config view --kubeconfig=${KCONFIG}
```

```bash
cat > /etc/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/bin/kube-scheduler \\
  --master=http://127.0.0.1:8080 \\
  --leader-elect=true \\
  --kubeconfig=/etc/kubernetes/scheduler.kubeconfig \\
  --address=127.0.0.1 \\
  --v=2
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF
```

起動

```bash
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
systemctl status kube-scheduler -l
```

---

## k8s node

```
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

CNI Plugins のインストール

https://github.com/kubernetes/kubernetes/blob/8a4a39d6ae1be45115c12cfa166f2c8151d88d8c/build/debian-hyperkube-base/Makefile

```bash
URL=https://storage.googleapis.com/kubernetes-release/network-plugins/cni-plugins-amd64-v0.6.0.tgz
curl -sSL $URL | tar -xz -C /opt/cni/bin
```

http://tech.uzabase.com/entry/2017/09/12/164756

```
cat > /etc/cni/net.d/10-weave.conf << EOF
{
    "name": "weave",
    "type": "weave-net",
    "hairpinMode": true
}
```

### kubelet

```bash
INTERNAL_IP=$(hostname -I | awk '{print $1}')
KUBERNETES_PUBLIC_ADDRESS=$INTERNAL_IP

CLUSTER_NAME="default"
KCONFIG=kubelet.conf
KUSER="system:node"

cd /etc/kubernetes/

kubectl config set-cluster ${CLUSTER_NAME} \
  --server=http://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=${KCONFIG}

kubectl config set-context ${KUSER}@${CLUSTER_NAME} \
  --cluster=${CLUSTER_NAME} \
  --user=${KUSER} \
  --kubeconfig=${KCONFIG}

kubectl config use-context ${KUSER}@${CLUSTER_NAME} --kubeconfig=${KCONFIG}
kubectl config view --kubeconfig=${KCONFIG}
```

```bash
CLUSTER_DNS_IP=10.233.0.10

mkdir -p /etc/kubernetes/manifests

cat > /etc/systemd/system/kubelet.service << EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/bin/kubelet \\
  --kubeconfig=/etc/kubernetes/kubelet.conf \\
  --require-kubeconfig=true \\
  --pod-manifest-path=/etc/kubernetes/manifests \\
  --allow-privileged=true \\
  --network-plugin=cni \\
  --cni-conf-dir=/etc/cni/net.d \\
  --cni-bin-dir=/opt/cni/bin \\
  --cluster-dns=${CLUSTER_DNS_IP} \\
  --cluster-domain=cluster.local \\
  --cgroup-driver=systemd \\
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF
```

```bash
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet -l
```

### kube-proxy

```bash
INTERNAL_IP=$(hostname -I | awk '{print $1}')
KUBERNETES_PUBLIC_ADDRESS=$INTERNAL_IP

CLUSTER_NAME="default"
KCONFIG=kube-proxy.conf
KUSER="kube-proxy"

cd /etc/kubernetes/

kubectl config set-cluster ${CLUSTER_NAME} \
  --server=http://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=${KCONFIG}

kubectl config set-context ${KUSER}@${CLUSTER_NAME} \
  --cluster=${CLUSTER_NAME} \
  --user=${KUSER} \
  --kubeconfig=${KCONFIG}

kubectl config use-context ${KUSER}@${CLUSTER_NAME} --kubeconfig=${KCONFIG}
kubectl config view --kubeconfig=${KCONFIG}
```

```bash
mkdir /var/lib/kube-proxy

cat > /etc/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/bin/kube-proxy \\
  --kubeconfig=/etc/kubernetes/kube-proxy.conf \\
  --v=2
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF
```

```bash
systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy -l
```

### kube-dns

```
kubectl create -f https://storage.googleapis.com/kubernetes-the-hard-way/kube-dns.yaml
```
