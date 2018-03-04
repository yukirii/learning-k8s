# Pre-config

## [master][node] Kernel Params & modules

```bash
echo -e "net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1" | sudo tee /etc/sysctl.d/60-kubernetes.conf

modprobe br_netfilter
sysctl -p /etc/sysctl.d/60-kubernetes.conf
```
