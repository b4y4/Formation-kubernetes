## Deployement
Un Deployment (déploiement en français) fournit des mises à jour déclaratives pour Pods et ReplicaSets.

** Vous décrivez un état désiré dans un déploiement et le controlleur déploiement change l'état réel à l'état souhaité à un rythme contrôlé (progressivement). 

** Vous pouvez définir des Deployments pour créer de nouveaux ReplicaSets, ou pour supprimer des déploiements existants et adopter toutes leurs ressources avec de nouveaux déploiements.

** Premet le retour en arriere ROLLINBACK

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
kubectl create -f deployement-demo.yaml
kubectl create deployement --image=nginx nginx

astuce:
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod-demo.yml
kubectl create deployment --image=nginx nginx --dry-run=client -o yaml > deployment-demo.yml

```

```
kubectl get deployement
kubectl describe deployement nginx-deployement
kubectl get rs # creation automatique de rs
kubectl get pods
```

```
kubectl delete deployement nginx-deployement
```

Next: [Deployment](../objects/namespace.md)
