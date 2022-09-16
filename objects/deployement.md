# Deployement

Un Deployment (déploiement en français) fournit des mises à jour déclaratives pour Pods et ReplicaSets.

* Vous décrivez un état désiré dans un déploiement et le controlleur déploiement change l'état réel à l'état souhaité à un rythme contrôlé (progressivement).
* Vous pouvez définir des Deployments pour créer de nouveaux ReplicaSets, ou pour supprimer des déploiements existants et adopter toutes leurs ressources avec de nouveaux déploiements.
* Premet le retour en arriere ROLLINBACK

```bash
> cat deployement-demo.yaml
```

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

```bash
kubectl create -f deployement-demo.yaml
kubectl create deployement --image=nginx nginx
```

astuce:

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod-demo.yml
kubectl create deployment --image=nginx nginx --dry-run=client -o yaml > deployment-demo.yml
```

update Objects

```bash
kubectl edit deployment nginx
kubectl scale deployement nginx --replicas=5
kubectl set image deployement nginx nginx=nginx:1.18

```

```bash
kubectl get deployement
kubectl describe deployement nginx-deployement
kubectl get rs # creation automatique de rs
kubectl get pods
```

```bash
kubectl delete deployement nginx-deployement
```

#### Mise à jour et retour en arriere

il existe deux stratégies de deploiement

* recreate: kill tout les pods et recriee d'autres de la nouvelle version (DOWN - indisponibilité de l'application)
* rolling update: remplace les pods un par un.

```bash
kubectl rollout status deployement/nginx-deployment
kubectl rollout history deployment/nginx-deployement
kubectl rollout undo deployement/nginx-deployement   # pour le retour en arriere
```

> Next: [Namespace](../objects/namespace.md)
> [cheat sheet](../useful.md)
