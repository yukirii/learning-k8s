# Network

## install cni

```
mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin
cd /opt/cni/bin
wget https://github.com/containernetworking/cni/releases/download/v0.6.0/cni-amd64-v0.6.0.tgz
wget https://github.com/containernetworking/plugins/releases/download/v0.6.0/cni-plugins-amd64-v0.6.0.tgz
gzip -dc cni-amd64-v0.6.0.tgz | tar xvf -
gzip -dc cni-plugins-amd64-v0.6.0.tgz | tar xvf -
```

kubelet に下記を追加

```
  --network-plugin=cni \
  --cni-conf-dir=/etc/cni/net.d \
  --cni-bin-dir=/opt/cni/bin \
```

## install flannel

```
wget https://github.com/coreos/flannel/releases/download/v0.9.1/flanneld-amd64
chmod 755 flanneld-amd64
cp flanneld-amd64 /usr/local/bin/flanneld
```

設定投入

```
docker run --rm --net=host quay.io/coreos/etcd etcdctl set /coreos.com/network/config '{ "Network": "10.1.0.0/16", "Backend": {"Type": "vxlan"}}'
```

```
export MASTER_IP=10.64.20.217

cat > /etc/sysconfig/flanneld <<EOF
ETCD_ENDPOINT=http://${MASTER_IP}:2379
IP_MASQ
IFACE
OPTS
EOF

cat > /etc/systemd/system/flannel.service <<EOF
[Unit]
Description=flanneld

[Service]
Type=simple
EnvironmentFile=-/etc/sysconfig/flanneld
ExecStart=/usr/local/sbin/flanneld \\
  -etcd-endpoint=${ETCD_ENDPOINT} \\
  -ip-masq=${IP_MASQ} \\
  -iface=${IFACE} \\
  ${OPTS}
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```
