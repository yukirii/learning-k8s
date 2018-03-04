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
