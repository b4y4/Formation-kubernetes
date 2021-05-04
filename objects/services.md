## Service
* Une manière abstraite d'exposer une application s'exécutant sur un ensemble de Pods en tant que service réseau.
* Un service identifie ses pods membres à l'aide d'un sélecteur. Pour qu'un pod soit membre du service, il doit comporter tous les libellés spécifiés dans le sélecteur.

![](../images/service.png)



```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: frontend
  ports:
  ...
```

### types de services:
* **ClusterIP**: le service crée une adresse IP virtuelle à l'intérieur du cluster pour permettre la communication entre différents services tels qu'un ensemble de serveurs frontaux à un ensemble de serveurs principaux.
* **NodePort**: le service rend un POD interne accessible sur un port du nœud [30000-32767]
* **Loadbalancer** : il fournit un équilibreur de charge pour notre service dans les fournisseurs de cloud pris en charge.







```
kubectl create -f pod.yml
kubectl create -f pod.yml --namespace=dev
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

```
kubectl create -f namespace.yml
kubectl create namespace dev
kubectl get pods --namespace=dev
kubectl get pods --all-namespace

kubectl config set-context $(kubectl config current-context) --namespace=dev
```


## Quota
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    pods: "10"
    requests.cpu: "4" 		# un pod ne peut pas demander plus de 4 CPU
    requests.memory: 5Gi 	# un pod ne peut pas demander plus de 5G de memoire
    limits.cpu: "10"	
    limits.memory: 10Gi
```

```
kubectl create -f compute-quota.yml
```

Next: [ReplicatSets](../objects/service.md)
