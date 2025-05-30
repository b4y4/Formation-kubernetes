# Pods

* Kubernetes ne deploie pas des conteneurs directement sur les nodes
* les conteneurs sont encapsulées dans un object appelé `pod`.
* Un pod est un groupe d'un ou de plusieurs conteneurs
* Un pod est une instance unique d'une application
* les conteneurs du mème pod partage le meme reseau

![Pods](../images/pod.jpeg)

```bash
kubectl run nginx --image nginx

kubectl get pods [-o wide|yaml|json]
```

## Structure d'un manifeste YAML

```yaml
---
apiVersion:
kind:
metadatas:


spec:

```

```bash
> cat Pod.yaml
```

```yaml
---
apiVersion: v1                  # String
kind: Pod                       # String
metadata:                       # Dictionnaire
  name: pod-demo
  labels:
    app: myapp
    type: front-end
spec:
  containers:                   # List/Array
    - name:  nginx-container    # 1er element
      image: nginx
```

```bash
kubectl create -f pod.yaml
kubectl apply -f pod.yaml
```

```bash
kubectl describe pod pod-demo
```

> Next: [Namespace](./namespace.md)

> [cheat sheet](../useful.md)
