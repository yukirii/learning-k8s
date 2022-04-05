# Smoke Test

Original: https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/13-smoke-test.md

## Data Encryption

```
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"

$ ssh controller-0 \
  "sudo ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem\
  /registry/secrets/default/kubernetes-the-hard-way | hexdump -C"
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a 1e a3 d7 19 c1 aa 74  |:v1:key1:......t|
00000050  a9 cd b1 b7 e3 90 68 1c  e5 4d 49 b2 9d e6 13 e5  |......h..MI.....|
00000060  88 ee f2 0c 65 41 91 6b  a7 f3 18 b2 c1 53 3f ae  |....eA.k.....S?.|
00000070  60 6d 30 33 48 b6 b2 2f  25 c3 4c 6a 88 c1 46 72  |`m03H../%.Lj..Fr|
00000080  b3 ff 83 75 54 4f 2e 91  6b 5a 82 85 2e 04 71 88  |...uTO..kZ....q.|
00000090  3c c6 57 52 26 00 fe e1  b5 f3 69 62 5e 09 7b bc  |<.WR&.....ib^.{.|
000000a0  5c 1a 0d 8a 64 ad 81 28  dc 4a 05 2a e5 cb e3 06  |\...d..(.J.*....|
000000b0  16 db f0 b0 6b cc de 7c  23 a4 b4 4d 89 88 c4 11  |....k..|#..M....|
000000c0  8a 43 31 c2 aa 62 5b 0e  15 ab 7c 69 a9 1a bb 76  |.C1..b[...|i...v|
000000d0  fb 93 57 e4 3e e0 9a 96  00 a4 9e b3 51 d1 32 ee  |..W.>.......Q.2.|
000000e0  08 bd 54 50 4a 23 1e 3e  2d e2 5f d3 34 0a e6 bb  |..TPJ#.>-._.4...|
000000f0  05 dd 63 a0 3c fc 24 39  99 b5 3a f8 cc 6f 08 f1  |..c.<.$9..:..o..|
00000100  9f 50 3e 6f 02 6f 25 c6  61 3d 51 a2 30 83 95 c3  |.P>o.o%.a=Q.0...|
00000110  43 f2 ce a3 67 21 1d ba  7d 87 6b a7 53 87 e3 26  |C...g!..}.k.S..&|
00000120  2e 20 6a 1f 24 99 4e b5  1e 2a 0d b1 4c 39 c7 0c  |. j.$.N..*..L9..|
00000130  3d 45 39 f6 6f b3 e5 54  e1 e4 6b bb 74 b6 0f 4e  |=E9.o..T..k.t..N|
00000140  e5 55 bb e6 68 40 2b e8  00 a0 89 88 e8 4d b3 a0  |.U..h@+......M..|
00000150  54 0d 6b 80 5a 61 fd 83  01 0a                    |T.k.Za....|
0000015a
```

## Deployments

```
kubectl create deployment nginx --image=nginx

$ kubectl get pods -l app=nginx
NAME                     READY   STATUS    RESTARTS   AGE
nginx-6799fc88d8-fjtzx   1/1     Running   0          17s
```

## Port Forwarding

```
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
kubectl portforward $POD_NAME 8080:80
```

```
% curl --head http://127.0.0.1:8080

HTTP/1.1 200 OK
Server: nginx/1.21.6
Date: Sun, 13 Feb 2022 10:07:25 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 25 Jan 2022 15:03:52 GMT
Connection: keep-alive
ETag: "61f01158-267"
Accept-Ranges: bytes
```

## Logs

```
% kubectl logs $POD_NAME

/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2022/02/13 10:02:41 [notice] 1#1: using the "epoll" event method
2022/02/13 10:02:41 [notice] 1#1: nginx/1.21.6
2022/02/13 10:02:41 [notice] 1#1: built by gcc 10.2.1 20210110 (Debian 10.2.1-6)
2022/02/13 10:02:41 [notice] 1#1: OS: Linux 5.4.0-1067-azure
2022/02/13 10:02:41 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2022/02/13 10:02:41 [notice] 1#1: start worker processes
2022/02/13 10:02:41 [notice] 1#1: start worker process 30
127.0.0.1 - - [13/Feb/2022:10:07:25 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.68.0" "-"
```

## Exec

```
% kubectl exec -ti $POD_NAME -- nginx -v
nginx version: nginx/1.21.6
```

## Services

https://github.com/ivanfioravanti/kubernetes-the-hard-way-on-azure/blob/master/docs/13-smoke-test.md#services

```
kubectl expose deployment nginx --port 80 --type NodePort
```

```
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')

az network nsg rule create -g kubernetes \
  -n kubernetes-allow-nginx \
  --access allow \
  --destination-address-prefix '*' \
  --destination-port-range ${NODE_PORT} \
  --direction inbound \
  --nsg-name kubernetes-nsg \
  --protocol tcp \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --priority 1002
```

```
EXTERNAL_IP=$(az network public-ip show -g kubernetes \
  -n worker-0-pip --query "ipAddress" -otsv)

% curl -I http://$EXTERNAL_IP:$NODE_PORT

HTTP/1.1 200 OK
Server: nginx/1.21.6
Date: Sun, 13 Feb 2022 10:10:49 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 25 Jan 2022 15:03:52 GMT
Connection: keep-alive
ETag: "61f01158-267"
Accept-Ranges: bytes
```
