# secrets

https://kubernetes.io/docs/concepts/configuration/secret/

## create secret

`kubectl create secret generic <secret_name> --from-literal=password=rootpassword`

```bash
kubectl create secret generic rootpass --from-literal=password=rootpassword

# 確認
kubectl get secret rootpass -o yaml | grep "password" | awk -F ' ' '{ print $2 }' | base64 -d
```

```bash
SECRET=$(echo "this-is-secret" | base64)

cat > secret-dot-file.yml <<EOF
kind: Secret
apiVersion: v1
metadata:
  name: secret-dot-file
data:
  .secret-dot-file: ${SECRET}
EOF
```

## volume
```
apiVersion: v1
kind: Pod
metadata:
  name: volume-secret
spec:
  containers:
    - name: volume-secret
      image: nginx
      volumeMounts:
        - name: secret-volume
          mountPath: /config
  volumes:
    - name: secret-volume
      secret:
        secretName: secret_name

# 確認
kubectl exec -it volume-secret cat /config/password
```

## environment variable

```
# env
apiVersion: v1
kind: Pod
metadata:
  name: env-secret
spec:
  containers:
    - name: env-secret
      image: nginx
      env:
        - name: PASSWORD
          valueFrom:
            secretKeyRef:
              name: secret_name
              key: password

# 確認
kubectl exec -it env-secret env | grep PASSWORD
```
