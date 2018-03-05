# Preparing Certs, Credentials

## [master] Certificate Authority

```bash
mkdir -p /etc/kubernetes/ssl
cd /etc/kubernetes/ssl

MASTER_IP=$(hostname -I | awk '{print $1}')

# ca.keyを生成
openssl genrsa -out ca.key 2048

# ca.crtを生成
openssl req -x509 -new -nodes -key ca.key -subj "/CN=${MASTER_IP}" -days 10000 -out ca.crt
```

### [master] Kubernetes 用ルート証明書を配置する

https://cloudpack.media/14148

```bash
# for Ubuntu
mkdir /usr/share/ca-certificates/kubernetes
cp /etc/kubernetes/ssl/ca.crt /usr/share/ca-certificates/kubernetes

## /usr/share/ca-certificets からの相対パスでファイル名を追記
echo "kubernetes/ca.crt" >> /etc/ca-certificates.conf

update-ca-certificates
```

---

## [master] X.509 証明書の作成

### admin Client Certificate

```bash
MASTER_IP=$(hostname -I | awk '{print $1}')
SERVICE_CLUSTER_IP=10.32.0.1
cd /etc/kubernetes/ssl

cat > /etc/kubernetes/ssl/admin-csr.conf << EOF
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
IP.2 = ${SERVICE_CLUSTER_IP}

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

# server certificateを表示する
# これをそれぞれの設定における certificate として利用する
openssl x509 -noout -text -in ./admin.crt
```

### kubelet Client Certificate

```bash
MASTER_IP=$(hostname -I | awk '{print $1}')
SERVICE_CLUSTER_IP=10.32.0.1
cd /etc/kubernetes/ssl

for instance in kirii-k8s-master01 kirii-k8s-node01 kirii-k8s-node02; do
cat > /etc/kubernetes/ssl/${instance}-csr.conf << EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
O = system:nodes
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
IP.2 = ${SERVICE_CLUSTER_IP}

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

### kube-proxy Client Certificate

```bash
MASTER_IP=$(hostname -I | awk '{print $1}')
SERVICE_CLUSTER_IP=10.32.0.1
cd /etc/kubernetes/ssl

cat > /etc/kubernetes/ssl/kube-proxy-csr.conf << EOF
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
IP.2 = ${SERVICE_CLUSTER_IP}

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

### kube-controller-manager Client Certificate

```bash
MASTER_IP=$(hostname -I | awk '{print $1}')
SERVICE_CLUSTER_IP=10.32.0.1
cd /etc/kubernetes/ssl

cat > /etc/kubernetes/ssl/kube-controller-manager-csr.conf << EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
O = system:kube-controller-manager
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
IP.2 = ${SERVICE_CLUSTER_IP}

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

### kube-scheduler Client Certificate

```bash
MASTER_IP=$(hostname -I | awk '{print $1}')
SERVICE_CLUSTER_IP=10.32.0.1
cd /etc/kubernetes/ssl

cat > /etc/kubernetes/ssl/kube-scheduler-csr.conf << EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
O = system:kube-scheduler
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
IP.2 = ${SERVICE_CLUSTER_IP}

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

### kubernetes API Server Client Certificate

```bash
MASTER_IP=$(hostname -I | awk '{print $1}')
SERVICE_CLUSTER_IP=10.32.0.1
cd /etc/kubernetes/ssl

cat > /etc/kubernetes/ssl/kubernetes-csr.conf << EOF
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
IP.2 = ${SERVICE_CLUSTER_IP}

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

---

## [master] kubeconfig 作成

### kubelet kubeconfig

```bash
export CLUSTER_NAME=scratch
export MASTER_IP=$(hostname -I | awk '{print $1}')

for instance in kirii-k8s-master01 kirii-k8s-node01 kirii-k8s-node02; do
  kubectl config set-cluster ${CLUSTER_NAME} \
    --certificate-authority=/etc/kubernetes/ssl/ca.crt \
    --embed-certs=true \
    --server=https://${MASTER_IP}:6443 \
    --kubeconfig=/etc/kubernetes/${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=/etc/kubernetes/ssl/${instance}.crt \
    --client-key=/etc/kubernetes/ssl/${instance}.key \
    --embed-certs=true \
    --kubeconfig=/etc/kubernetes/${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=${CLUSTER_NAME} \
    --user=system:node:${instance} \
    --kubeconfig=/etc/kubernetes/${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=/etc/kubernetes/${instance}.kubeconfig
done
```

### kube-proxy kubeconfig

```bash
export CLUSTER_NAME=scratch
export MASTER_IP=$(hostname -I | awk '{print $1}')

kubectl config set-cluster ${CLUSTER_NAME} \
  --certificate-authority=/etc/kubernetes/ssl/ca.crt \
  --embed-certs=true \
  --server=https://${MASTER_IP}:6443 \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

kubectl config set-credentials kube-proxy \
  --client-certificate=/etc/kubernetes/ssl/kube-proxy.crt \
  --client-key=/etc/kubernetes/ssl/kube-proxy.key \
  --embed-certs=true \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=${CLUSTER_NAME} \
  --user=kube-proxy \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig
```

### kube-controller-manager kubeconfig

```bash
export CLUSTER_NAME=scratch
export MASTER_IP=$(hostname -I | awk '{print $1}')

kubectl config set-cluster ${CLUSTER_NAME} \
  --certificate-authority=/etc/kubernetes/ssl/ca.crt \
  --embed-certs=true \
  --server=https://${MASTER_IP}:6443 \
  --kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig

kubectl config set-credentials kube-controller-manager \
  --client-certificate=/etc/kubernetes/ssl/kube-controller-manager.crt \
  --client-key=/etc/kubernetes/ssl/kube-controller-manager.key \
  --embed-certs=true \
  --kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig

kubectl config set-context default \
  --cluster=${CLUSTER_NAME} \
  --user=kube-controller-manager \
  --kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig
```

### kube-scheduler kubeconfig

```bash
export CLUSTER_NAME=scratch
export MASTER_IP=$(hostname -I | awk '{print $1}')

kubectl config set-cluster ${CLUSTER_NAME} \
  --certificate-authority=/etc/kubernetes/ssl/ca.crt \
  --embed-certs=true \
  --server=https://${MASTER_IP}:6443 \
  --kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig

kubectl config set-credentials kube-scheduler \
  --client-certificate=/etc/kubernetes/ssl/kube-scheduler.crt \
  --client-key=/etc/kubernetes/ssl/kube-scheduler.key \
  --embed-certs=true \
  --kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig

kubectl config set-context default \
  --cluster=${CLUSTER_NAME} \
  --user=kube-scheduler \
  --kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig
```

---

## [master] Distribute kubeconfig files

```bash
cd /etc/kubernetes
for instance in kirii-k8s-master01 kirii-k8s-node01 kirii-k8s-node02; do
  scp ${instance}.kubeconfig kube-proxy.kubeconfig ${instance}:~/
done
```
