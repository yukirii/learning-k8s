# Install Components

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
