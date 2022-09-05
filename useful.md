## Check the K8S cluster health
---
```console
curl -k 'https://localhost:6443/readyz?verbose&exclude=etcd'
or
kubectl get --raw='/readyz?verbose'
```
