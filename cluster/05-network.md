# Network

## flannel

```bash
wget https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
```

変更

```
data:
  cni-conf.json: |
    {
      "name": "docker0",
      "type": "flannel",
      "delegate": {
        "isDefaultGateway": true
      }
    }
  net-conf.json: |
    {
      "Network": "10.1.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```

```
kubectl apply -f kube-flannel.yml
```
