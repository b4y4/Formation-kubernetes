## ConfigMap
* Les ConfigMaps sont utilisés pour transmettre les données de configuration sous forme de paires clé/valeur dans kubernetes, et les injectez dans les pods
* les clé/valeur sont disponibles en tant que variables d'envirennement pour l'application hébergée à l'interieur du conteneur
------------------------------------------------------
``` |
Methode 1|

kubectl create configmap config_name \ |
	--from-literal=APP_VERION=1 \ |
	--from-literal=APPP_ENV=prod |
```|

```						
Methode 2
kubectl create configmap config_name \		       
	--from-file=app_config.properties	
```						

```yaml
---

```
Next: [ReplicatSets](../objects/service.md)
