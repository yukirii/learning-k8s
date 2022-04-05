# metrics-server

```
% kubectl top pods
error: Metrics API not available
```

https://github.com/kubernetes-sigs/metrics-server#installation

```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

```
% kubectl get pods -n kube-system
NAME                              READY   STATUS    RESTARTS   AGE
coredns-8494f9c688-2b8x7          1/1     Running   0          2d2h
coredns-8494f9c688-hn4mx          1/1     Running   0          2d2h
metrics-server-64cf6869bd-495bz   1/1     Running   0          2m4s

% kubectl top pods
Error from server (ServiceUnavailable): the server is currently unable to handle the request (get pods.metrics.k8s.io)
```

```
% kubectl logs -n kube-system metrics-server-64cf6869bd-495bz
...
E0215 12:27:40.556227       1 configmap_cafile_content.go:242] kube-system/extension-apiserver-authentication failed with : missing content for CA bundle "client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
E0215 12:27:45.189836       1 configmap_cafile_content.go:242] key failed with : missing content for CA bundle "client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
E0215 12:28:45.190382       1 configmap_cafile_content.go:242] key failed with : missing content for CA bundle "client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
E0215 12:29:45.189901       1 configmap_cafile_content.go:242] key failed with : missing content for CA bundle "client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
```

https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/

## TLS Certificates

```
cat > aggregator-ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > aggregator-ca-csr.json <<EOF
{
  "CN": "Kubernetes aggregator",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Minato-ku",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Tokyo"
    }
  ]
}
EOF

cfssl gencert -initca aggregator-ca-csr.json | cfssljson -bare aggregator-ca
```

```
cat > proxy-client-csr.json <<EOF
{
  "CN": "aggregator",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Minato-ku",
      "O": "system:masters",
      "OU": "CA",
      "ST": "Tokyo"
    }
  ]
}
EOF

cfssl gencert \
  -ca=aggregator-ca.pem \
  -ca-key=aggregator-ca-key.pem \
  -config=aggregator-ca-config.json \
  -profile=kubernetes \
  proxy-client-csr.json | cfssljson -bare proxy-client
```

### Distribute the Client and Server Certificates

```
for instance in controller-0 controller-1 controller-2; do
  scp aggregator-ca.pem aggregator-ca-key.pem proxy-client-key.pem proxy-client.pem ${instance}:~/
done
```

## Configure kube-apiserver

```
sudo mv aggregator-ca.pem aggregator-ca-key.pem \
  proxy-client-key.pem proxy-client.pem /var/lib/kubernetes/

vim /etc/systemd/system/kube-apiserver.service

...

  --requestheader-client-ca-file=/var/lib/kubernetes/aggregator-ca.pem \
  --requestheader-allowed-names=front-proxy-client \
  --requestheader-extra-headers-prefix=X-Remote-Extra- \
  --requestheader-group-headers=X-Remote-Group \
  --requestheader-username-headers=X-Remote-User \
  --proxy-client-cert-file=/var/lib/kubernetes/proxy-client.pem \
  --proxy-client-key-file=/var/lib/kubernetes/proxy-client-key.pem \
  --enable-aggregator-routing=true \

sudo systemctl daemon-reload
sudo systemctl restart kube-apiserver
```

## Verification

```
% kubectl get cm -n kube-system
NAME                                 DATA   AGE
coredns                              1      3d1h
extension-apiserver-authentication   6      3d2h
kube-root-ca.crt                     1      3d2h

% kubectl top nodes
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
worker-0   46m          4%     1346Mi          40%
worker-1   35m          3%     739Mi           22%

% kubectl top pods -n kube-system
NAME                              CPU(cores)   MEMORY(bytes)
coredns-8494f9c688-2b8x7          2m           16Mi
coredns-8494f9c688-hn4mx          2m           15Mi
metrics-server-64cf6869bd-495bz   3m           26Mi
```
