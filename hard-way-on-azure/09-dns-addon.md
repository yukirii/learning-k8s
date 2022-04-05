# Deploying the DNS Cluster Add-on

Original: https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/12-dns-addon.md

## The DNS Cluster Add-on

```
kubectl apply -f https://storage.googleapis.com/kubernetes-the-hard-way/coredns-1.8.yaml

$ kubectl get pods -l k8s-app=kube-dns -n kube-system
NAME                       READY   STATUS    RESTARTS   AGE
coredns-8494f9c688-2b8x7   1/1     Running   0          11s
coredns-8494f9c688-hn4mx   1/1     Running   0          12s
```

## Verification

```
$ kubectl run busybox --image=busybox:1.28 --command -- sleep 3600
pod/busybox created

$ kubectl get pods -l run=busybox
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          13s

$ POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")

$ kubectl exec -ti $POD_NAME -- nslookup kubernetes
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```
