## Pods 
* Kubernetes ne deploie pas des conteneurs directement sur les nodes
* les conteneurs sont encapsulées dans un object appelé pods.
* Un pod est un groupe d'un ou de plusieurs conteneurs
* Un pod est une instance unique d'une application
* les conteneurs du mème pod partage le meme reseau


![Pods](../images/pod.jpeg)



```
kubectl run nginx --image nginx

kubectl get pods [-o wide|yaml|json]
```

## fichier Yaml


```yaml
---
apiVersion:
kind
metadatas:


spec:

```
