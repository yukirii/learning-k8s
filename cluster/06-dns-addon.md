# DNS Add-on

## kube-dns

```
export DNS_DOMAIN="cluster.local"
export DNS_SERVER_IP="10.32.0.10"

cd /etc/kubernetes/manifests/

wget https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/kube-dns.yaml.sed
sed -i -e "s/\\\$DNS_DOMAIN/${DNS_DOMAIN}/g" kube-dns.yaml.sed
sed -i -e "s/\\\$DNS_SERVER_IP/${DNS_SERVER_IP}/g" kube-dns.yaml.sed
mv kube-dns.yaml.sed kube-dns.yaml

kubectl --namespace=kube-system apply -f kube-dns.yaml
```
