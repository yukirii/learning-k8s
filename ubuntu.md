## pre

### Kernel Params & modules

```bash
echo -e "net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1" | sudo tee /etc/sysctl.d/60-kubernetes.conf

modprobe br_netfilter
sysctl -p /etc/sysctl.d/60-kubernetes.conf
```

---

## Install docker

https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-using-the-repository

```bash
apt-get update
apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

apt-get update
apt-get install docker-ce

systemctl daemon-reload
systemctl start docker
systemctl enable docker
```

---

## install Kubernetes

### mkdir

```bash
mkdir -p /etc/kubernetes
```

### ダウンロードと展開

```bash
# kubernetes v.1.9.1 をダウンロードし展開
wget https://github.com/kubernetes/kubernetes/releases/download/v1.9.n/kubernetes.tar.gz
tar zxvf kubernetes.tar.gz

# バイナリをダウンロード
./kubernetes/cluster/get-kube-binaries.sh

# client, server 用バイナリを展開
# kubernetes/client/bin と kubernetes/server/bin にバイナリが展開される
tar xf kubernetes/client/kubernetes-client-linux-amd64.tar.gz
tar xf kubernetes/server/kubernetes-server-linux-amd64.tar.gz

cp kubernetes/server/bin/kubectl /usr/local/bin/
cp kubernetes/server/bin/kubelet /usr/local/bin/
cp kubernetes/server/bin/kube-proxy /usr/local/bin/
```

### load docker image

```bash
docker image load -i kubernetes/server/bin/kube-apiserver.tar
docker image load -i kubernetes/server/bin/kube-scheduler.tar
docker image load -i kubernetes/server/bin/kube-controller-manager.tar
docker image pull gcr.io/google-containers/etcd:3.1.11
```

---

## Preparing Certs, Credentials

### Certificate Authority

```bash
mkdir -p /etc/kubernetes/pki
cd /etc/kubernetes/pki

MASTER_IP=$(hostname -I | awk '{print $1}')

# ca.keyを生成
openssl genrsa -out ca.key 2048

# ca.crtを生成
openssl req -x509 -new -nodes -key ca.key -subj "/CN=${MASTER_IP}" -days 10000 -out ca.crt
```

### X.509 証明書の作成

#### admin Client Certificate

```bash
MASTER_IP=$(hostname -I | awk '{print $1}')
MASTER_CLUSTER_IP=10.1.0.1

cat > /etc/kubernetes/pki/admin-csr.conf << EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
O = system:masters
CN = admin

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
IP.1 = ${MASTER_IP}
IP.2 = ${MASTER_CLUSTER_IP}

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names
EOF

# admin.key を生成
openssl genrsa -out admin.key 2048

# admin.key に CSR を適用して x509 を生成する
openssl req -new -key admin.key -out admin.csr -config admin-csr.conf

# Server Certificate を生成する
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out admin.crt -days 10000 \
  -extensions v3_ext -extfile admin-csr.conf

# server certificateを表示する。これをそれぞれの設定におけるcertificateとして利用する
openssl x509 -noout -text -in ./admin.crt
```

#### kubelet Client Certificate

```bash
MASTER_IP=$(hostname -I | awk '{print $1}')
MASTER_CLUSTER_IP=10.1.0.1

for instance in kirii-k8s-master01 kirii-k8s-node01 kirii-k8s-node02; do
cat > /etc/kubernetes/pki/${instance}-csr.conf << EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
C = JP
ST = Tokyo
L = Shibuya-ku
O = system:nodes
OU = Technology Headquarters
CN = system:node:${instance}

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
DNS.6 = ${instance}
IP.1 = ${MASTER_IP}
IP.2 = ${MASTER_CLUSTER_IP}

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names
EOF

openssl genrsa -out ${instance}.key 2048
openssl req -new -key ${instance}.key -out ${instance}.csr -config ${instance}-csr.conf
openssl x509 -req -in ${instance}.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out ${instance}.crt -days 10000 \
  -extensions v3_ext -extfile ${instance}-csr.conf
done
```

#### kube-proxy Client Certificate

```bash
MASTER_IP=$(hostname -I | awk '{print $1}')
MASTER_CLUSTER_IP=10.1.0.1

cat > /etc/kubernetes/pki/kube-proxy-csr.conf << EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
O = system:node-proxier
CN = system:kube-proxy

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
IP.1 = ${MASTER_IP}
IP.2 = ${MASTER_CLUSTER_IP}

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names
EOF

openssl genrsa -out kube-proxy.key 2048
openssl req -new -key kube-proxy.key -out kube-proxy.csr -config kube-proxy-csr.conf
openssl x509 -req -in kube-proxy.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out kube-proxy.crt -days 10000 \
  -extensions v3_ext -extfile kube-proxy-csr.conf
```

#### kube-controller-manager Client Certificate

```bash
MASTER_IP=$(hostname -I | awk '{print $1}')
MASTER_CLUSTER_IP=10.1.0.1

cat > /etc/kubernetes/pki/kube-controller-manager-csr.conf << EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
CN = system:kube-controller-manager

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
IP.1 = ${MASTER_IP}
IP.2 = ${MASTER_CLUSTER_IP}

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names
EOF

openssl genrsa -out kube-controller-manager.key 2048
openssl req -new -key kube-controller-manager.key -out kube-controller-manager.csr -config kube-controller-manager-csr.conf
openssl x509 -req -in kube-controller-manager.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out kube-controller-manager.crt -days 10000 \
  -extensions v3_ext -extfile kube-controller-manager-csr.conf
```

#### kube-scheduler Client Certificate

```bash
MASTER_IP=$(hostname -I | awk '{print $1}')
MASTER_CLUSTER_IP=10.1.0.1

cat > /etc/kubernetes/pki/kube-scheduler-csr.conf << EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
CN = system:kube-scheduler

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
IP.1 = ${MASTER_IP}
IP.2 = ${MASTER_CLUSTER_IP}

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names
EOF

openssl genrsa -out kube-scheduler.key 2048
openssl req -new -key kube-scheduler.key -out kube-scheduler.csr -config kube-scheduler-csr.conf
openssl x509 -req -in kube-scheduler.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out kube-scheduler.crt -days 10000 \
  -extensions v3_ext -extfile kube-scheduler-csr.conf
```

#### kubernetes API Server Client Certificate

```bash
MASTER_IP=$(hostname -I | awk '{print $1}')
MASTER_CLUSTER_IP=10.1.0.1

cat > /etc/kubernetes/pki/kubernetes-csr.conf << EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
O = kubernetes
CN = kubernetes

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
IP.1 = ${MASTER_IP}
IP.2 = ${MASTER_CLUSTER_IP}

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names
EOF

openssl genrsa -out kubernetes.key 2048
openssl req -new -key kubernetes.key -out kubernetes.csr -config kubernetes-csr.conf
openssl x509 -req -in kubernetes.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out kubernetes.crt -days 10000 \
  -extensions v3_ext -extfile kubernetes-csr.conf
```

### kubernetes 用ルート証明書を配置する

https://cloudpack.media/14148

```bash
# for Ubuntu
mkdir /usr/share/ca-certificates/kubernetes
cp ca.crt /usr/share/ca-certificates/kubernetes

## /usr/share/ca-certificets からの相対パスでファイル名を追記
echo "kubernetes/ca.crt" >> /etc/ca-certificates.conf

update-ca-certificates
```

### Credential を client に公開する

#### kubelet kubeconfig

```bash
export CLUSTER_NAME=scratch
export MASTER_IP=$(hostname -I | awk '{print $1}')
export CA_CERT=/etc/kubernetes/pki/ca.crt

for instance in kirii-k8s-master01 kirii-k8s-node01 kirii-k8s-node02; do
  kubectl config set-cluster ${CLUSTER_NAME} \
    --certificate-authority=${CA_CERT} \
    --embed-certs=true \
    --server=https://${MASTER_IP}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.crt \
    --client-key=${instance}.key \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=${CLUSTER_NAME} \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```

#### kube-proxy kubeconfig

```bash
export CLUSTER_NAME=scratch
export MASTER_IP=$(hostname -I | awk '{print $1}')
export CA_CERT=/etc/kubernetes/pki/ca.crt

kubectl config set-cluster ${CLUSTER_NAME} \
  --certificate-authority=${CA_CERT} \
  --embed-certs=true \
  --server=https://${MASTER_IP}:6443 \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.crt \
  --client-key=kube-proxy.key \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=${CLUSTER_NAME} \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

#### kube-controller-manager kubeconfig

```bash
export CLUSTER_NAME=scratch
export MASTER_IP=$(hostname -I | awk '{print $1}')
export CA_CERT=/etc/kubernetes/pki/ca.crt

kubectl config set-cluster ${CLUSTER_NAME} \
  --certificate-authority=${CA_CERT} \
  --embed-certs=true \
  --server=https://${MASTER_IP}:6443 \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials kube-controller-manager \
  --client-certificate=kube-controller-manager.crt \
  --client-key=kube-controller-manager.key \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
  --cluster=${CLUSTER_NAME} \
  --user=kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```

#### kube-scheduler kubeconfig

```bash
export CLUSTER_NAME=scratch
export MASTER_IP=$(hostname -I | awk '{print $1}')
export CA_CERT=/etc/kubernetes/pki/ca.crt

kubectl config set-cluster ${CLUSTER_NAME} \
  --certificate-authority=${CA_CERT} \
  --embed-certs=true \
  --server=https://${MASTER_IP}:6443 \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials kube-scheduler \
  --client-certificate=kube-scheduler.crt \
  --client-key=kube-scheduler.key \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
  --cluster=${CLUSTER_NAME} \
  --user=kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```

---

## Master node で必要なミドルウェアの設定と起動

### docker

```bash
cat > /etc/docker/daemon.json << EOF
{
    "bip": "10.1.0.1/24",
    "mtu": 1450,
    "insecure-registries": []
}
EOF

systemctl restart docker
```

### kubelet

```bash
kubelet \
  --kubeconfig=/etc/kubernetes/pki/kirii-k8s-master01.kubeconfig \
  --pod-manifest-path=/etc/kubernetes/manifests \
  --cluster-dns=10.1.0.10 \
  --cluster-domain=cluster.local \
  --allow-privileged \
  --v=2
```

##### kubelet service

```bash
CLUSTER_DNS_IP=10.1.0.10

cat > /etc/systemd/system/kubelet.service << EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --kubeconfig=/etc/kubernetes/pki/kirii-k8s-master01.kubeconfig \\
  --pod-manifest-path=/etc/kubernetes/manifests \\
  --cluster-dns=10.1.0.10 \\
  --cluster-domain=cluster.local \\
  --allow-privileged \\
  --v=2
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet -l
```

### kube-proxy

設定ファイルを作成する。以下のコマンドで雛形を生成し編集する。

```bash
kube-proxy --write-config-to /etc/kubernetes/kube-proxy.yml
```

```yaml
apiVersion: componentconfig/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  kubeconfig: "/etc/kubernetes/pki/kube-proxy.kubeconfig"
  qps: 5
clusterCIDR: "10.1.0.0/16"
configSyncPeriod: 15m0s
conntrack:
  max: 0
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
enableProfiling: false
featureGates: ""
healthzBindAddress: 0.0.0.0:10256
hostnameOverride: ""
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
kind: KubeProxyConfiguration
metricsBindAddress: 127.0.0.1:10249
mode: "iptables"
oomScoreAdj: -999
portRange: ""
resourceContainer: /kube-proxy
udpTimeoutMilliseconds: 250ms
```

```bash
kube-proxy --config=/etc/kubernetes/kube-proxy.yml --v=2
```

##### kube-proxy service

```bash
cat > /etc/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/etc/kubernetes/kube-proxy.yml \\
  --v=2
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy -l
```


## 各サービスの Bootstrap

manifest ファイルは `/etc/kubernetes/manifests/<サービス名>.yaml` に保存する。

```bash
mkdir -p /etc/kubernetes/manifests
```

### etcd

```bash
cat > /etc/kubernetes/manifests/etcd.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  creationTimestamp: null
  labels:
    component: etcd
    tier: control-plane
  name: etcd
  namespace: kube-system
spec:
  containers:
  - command:
    - etcd
    - --listen-client-urls=http://127.0.0.1:2379
    - --advertise-client-urls=http://127.0.0.1:2379
    - --data-dir=/var/lib/etcd
    image: gcr.io/google_containers/etcd:3.1.11
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /health
        port: 2379
        scheme: HTTP
      initialDelaySeconds: 15
      timeoutSeconds: 15
    name: etcd
    resources: {}
    volumeMounts:
    - mountPath: /etc/ssl/certs
      name: certs
    - mountPath: /var/lib/etcd
      name: etcd
    - mountPath: /etc/kubernetes
      name: k8s
      readOnly: true
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/ssl/certs
    name: certs
  - hostPath:
      path: /var/lib/etcd
    name: etcd
  - hostPath:
      path: /etc/kubernetes
    name: k8s
status: {}
EOF
```

### kube-apiserver

```
INTERNAL_IP=$(hostname -I | awk '{print $1}')

cat > /etc/kubernetes/manifests/kube-apiserver.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --secure-port=6443
    - --requestheader-allowed-names=front-proxy-client
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --insecure-port=0
    - --admission-control=Initializers,NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --requestheader-username-headers=X-Remote-User
    - --tls-ca-file=/etc/kubernetes/pki/ca.crt
    - --tls-cert-file=/etc/kubernetes/pki/kubernetes.crt
    - --tls-private-key-file=/etc/kubernetes/pki/kubernetes.key
    - --kubelet-certificate-authority=/etc/kubernetes/pki/ca.crt
    - --kubelet-client-certificate=/etc/kubernetes/pki/kubernetes.crt
    - --kubelet-client-key=/etc/kubernetes/pki/kubernetes.key
    - --kubelet-https=true
    - --allow-privileged=true
    - --requestheader-group-headers=X-Remote-Group
    - --service-cluster-ip-range=10.1.0.0/16
    - --authorization-mode=Node,RBAC
    - --advertise-address=${INTERNAL_IP}
    - --etcd-servers=http://127.0.0.1:2379
    image: gcr.io/google_containers/kube-apiserver:v1.9.1
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 6443
        scheme: HTTPS
      initialDelaySeconds: 15
      timeoutSeconds: 15
    name: kube-apiserver
    resources:
      requests:
        cpu: 250m
    volumeMounts:
    - mountPath: /etc/kubernetes
      name: k8s
      readOnly: true
    - mountPath: /etc/ssl/certs
      name: certs
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/kubernetes
    name: k8s
  - hostPath:
      path: /etc/ssl/certs
    name: certs
status: {}
EOF
```

### kube-scheduler

```
cat > /etc/kubernetes/manifests/kube-scheduler.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  creationTimestamp: null
  labels:
    component: kube-scheduler
    tier: control-plane
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --kubeconfig=/etc/kubernetes/pki/kube-scheduler.kubeconfig
    - --address=127.0.0.1
    - --leader-elect=true
    image: gcr.io/google_containers/kube-scheduler:v1.9.1
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10251
        scheme: HTTP
      initialDelaySeconds: 15
      timeoutSeconds: 15
    name: kube-scheduler
    resources:
      requests:
        cpu: 100m
    volumeMounts:
    - mountPath: /etc/kubernetes
      name: k8s
      readOnly: true
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/kubernetes
    name: k8s
status: {}
EOF
```

### kube-controller-manager

```
cat > /etc/kubernetes/manifests/kube-controller-manager.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  creationTimestamp: null
  labels:
    component: kube-controller-manager
    tier: control-plane
  name: kube-controller-manager
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-controller-manager
    - --kubeconfig=/etc/kubernetes/pki/kube-controller-manager.kubeconfig
    - --address=127.0.0.1
    - --allocate-node-cidrs=true
    - --cluster-cidr=10.1.0.0/16
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --controllers=*,bootstrapsigner,tokencleaner
    - --leader-elect=true
    - --root-ca-file=/etc/kubernetes/pki/ca.crt
    - --use-service-account-credentials=true
    - --service-account-private-key-file=/etc/kubernetes/pki/ca.key
    image: gcr.io/google_containers/kube-controller-manager:v1.9.1
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10252
        scheme: HTTP
      initialDelaySeconds: 15
      timeoutSeconds: 15
    name: kube-controller-manager
    resources:
      requests:
        cpu: 200m
    volumeMounts:
    - mountPath: /etc/kubernetes
      name: k8s
      readOnly: true
    - mountPath: /etc/ssl/certs
      name: certs
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/kubernetes
    name: k8s
  - hostPath:
      path: /etc/ssl/certs
    name: certs
status: {}
EOF
```

