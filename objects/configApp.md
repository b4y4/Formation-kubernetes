## ConfigMap
* Les ConfigMaps sont utilisés pour transmettre les données de configuration sous forme de paires clé/valeur dans kubernetes, et les injectez dans les pods
* les clé/valeur sont disponibles en tant que variables d'envirennement pour l'application hébergée à l'interieur du conteneur
------------------------------------------------------
``` 
Methode 1
------
kubectl create configmap config_name \ 
	--from-literal=APP_VERION=1 \ 
	--from-literal=APP_ENV=prod 
```

```						
Methode 2
------
cat app_config.properties
> APP_VERSION=1
> APP_ENV=prod


kubectl create configmap config_name \		       
	--from-file=app_config.properties	
```						

```yaml
Methode 3

apiVerion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_VERSION: 1
  APP_ENV: prod

---
apiVersion: v1
kind: Pod
metadata:
  name: 
  labels:
    name: 
spec:
  containers:
  - name: 
    image:
    ports:
      - conatnerPort: 8080
    envFrom:
      - configMapRef:
          name: app-config

```
```
kubectl create -f configmaps-demo.yaml
kubectl create -f pod-demo.yaml
```
Next: [Secret](../objects/secret.md)
