# kubectl

[kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)


## bash completion

```bash
echo "source <(kubectl completion bash)" >> ~/.bashrc
source .bashrc
```


## cluster-info & componentstatuses

```bash
kubectl cluster-info

kubectl get componentstatuses
```


## nginx をサクッと立ち上げ

```bash
## nginx の pod を立ち上げてみる
kubectl run nginx --image=nginx:1.13 --port=80

## service を作って叩けるようにする
### NodePort
kubectl expose deploy nginx --name=nginx --port=80 --target-port=80 --type="NodePort"
### LoadBalancer
kubectl expose deploy nginx --name=nginx --port=80 --target-port=80 --type="LoadBalancer"

## pod の数を増やしてみる
kubectl scale deployment/nginx --replicas=6
```

## deployment 操作

```bash
cat > deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        pod: nginx
    spec:
      containers:
      - image: nginx:1.11
        name: nginx
EOF

## apply deployment
kubectl apply -f deployment.yaml --record
kubectl get deployment -o wide

## set image (version up)
kubectl set image deployment/nginx nginx=nginx:1.13
kubectl get deployment -o wide

## check history
kubectl rollout history deployment/nginx
kubectl rollout history deployment/nginx --revision 2

## rollback (version down)
kubectl rollout undo deployment/nginx
kubectl rollout undo deployment/nginx --to-revision 1
kubectl get deployment -o wide
```


## kubectl get の結果を sort して表示

```bash
kubectl get pods --sort-by=.metadata.name
kubectl get pods --sort-by=.metadata.creationTimestamp
```

## label SELECTOR

```
kubectl get svc -o wide
kubectl get pod -l run=nginx
```

## kubectl drain

`--force` ReplicationController, Job, DaemonSet によって管理されていない Pod があっても続行

```
kubectl drain kirii-k8s-node01 --delete-local-data --ignore-daemonsets --force
```

## kubectl cordon

node を unschedulable にする

```
kubectl cordon kirii-k8s-node01
```

## nodeSelector

[Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/configuration/assign-pod-node)

```
# つける
kubectl label nodes kirii-k8s-node02 webserver=apache

# 一覧表示
kubectl get node --show-labels
# 絞り込み
kubectl get node -l webserver=apache

cat > nodeselector.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: httpd
spec:
  containers:
    - name: httpd
      image: httpd
  nodeSelector:
    webserver: apache
EOF

kubectl create -f nodeselector.yaml
kubectl get pods -o wide
```
