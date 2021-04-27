## Deployement
Un Deployment (déploiement en français) fournit des mises à jour déclaratives pour Pods et ReplicaSets.

Vous décrivez un état désiré dans un déploiement et le controlleur déploiement change l'état réel à l'état souhaité à un rythme contrôlé. Vous pouvez définir des Deployments pour créer de nouveaux ReplicaSets, ou pour supprimer des déploiements existants et adopter toutes leurs ressources avec de nouveaux déploiements.

## deployement-demo.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

```
kubectl create -f replicatset-demo.yaml
```

```
kubectl get rs
kubectl describe rs/frontend
kubectl get pods
```

```
kubectl replace -f replicatset-demo.yaml
kubectl scale --replicas=5 -f replicatset-demo.yaml
kubectl scale --replicas=6 replicatset frontend

kubectl delete replicaset frontend
```

Next: [Deployment](../objects/namespace.md)
