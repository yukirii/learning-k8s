# etcd

## create snapshot

```bash
ETCDCTL_API=3 etcdctl --endpoints http://127.0.0.1:2379 snapshot save snapshot.db

ETCDCTL_API=3 etcdctl --endpoints https://127.0.0.1:2379 snapshot save etcd-snapshot.db \
--cert="etcd-client.crt" --key="etcd-client.key" --cacert="ca.crt"
```
