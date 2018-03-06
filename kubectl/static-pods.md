## Static Pods

https://kubernetes.io/docs/tasks/administer-cluster/static-pod/

## configuration file

```
cat <<EOF >/etc/kubelet.d/static-web.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-web
  labels:
    role: myrole
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - name: web
          containerPort: 80
          protocol: TCP
EOF
```

--pod-manifest-path=/etc/kubelet.d/

