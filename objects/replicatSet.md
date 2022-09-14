## ReplicatSet
Un ReplicaSet (ensemble de réplicas en français) a pour but de maintenir un ensemble stable de Pods à un moment donné. Cet objet est souvent utilisé pour garantir la disponibilité d'un certain nombre identique de Pods.

Un ReplicaSet est défini avec des champs, incluant un selecteur qui spécifie comment identifier les Pods qu'il peut posséder, un nombre de replicas indiquant le nombre de Pods qu'il doit maintenir et un modèle de Pod spécifiant les données que les nouveaux Pods que le replicatSet va créer jusqu'au nombre de replicas demandé.


## replicatset-demo.yaml

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      type: front-end
  template:
    metadata:
      labels:
        type: front-end
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3

```

```console
kubectl create -f replicatset-demo.yaml
```

```console
kubectl get rs
kubectl describe rs/frontend
kubectl get pods
```

```console
kubectl replace -f replicatset-demo.yaml
kubectl scale --replicas=5 -f replicatset-demo.yaml
kubectl scale --replicas=6 replicatset frontend

kubectl delete replicaset frontend
```

Next: [Deployment](../objects/deployement.md)

[Useful commands](../useful.md)
