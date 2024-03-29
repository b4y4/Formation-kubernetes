# ConfigMaps

---

* Les ConfigMaps sont utilisés pour transmettre les données de configuration sous forme de paires clé/valeur dans kubernetes, et les injectez dans les pods
* les clé/valeur sont disponibles en tant que variables d'envirennement pour l'application hébergée à l'interieur du conteneur

---

### Methode 1

```bash
kubectl create configmap config_name \
 --from-literal=APP_VERION=1 \
 --from-literal=APP_ENV=prod
```

### Methode 2

```bash
cat app_config.properties
APP_VERSION=1
APP_ENV=prod


kubectl create configmap config_name \
 --from-file=app_config.properties
```

### Methode 3

```yaml

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
  name: pod-configmap-demo
  labels:
    name: pod-configmap-demo
spec:
  containers:
  - name: pod-demo
    image: busybox
    ports:
      - conatnerPort: 8080
    envFrom:
      - configMapRef:
          name: app-config

```

```bash
kubectl create -f configmaps-demo.yaml
kubectl create -f pod-demo.yaml
```

## Secret

* Les object Secret permettent de stocker et de gérer des informations sensibles, telles que les mots de pass, les tokens ou le clés SSH
* l'objet Secret est similaire à Configmaps, sauf qu'il est stockés dans un format hashé (base64)

---

```bash
kubectl create secret generic \
 app-secret --from-literal=DB_HOST=mysql \
     --from-literal=DB_USER=Admin \
     --from-literal=DB-PASS=P@$$w0rd
```

### Methode 2

```bash
cat secret
> DB_HOST=mysql
> DB_USER=Admin
> DB_PASS=P@$$w0rd


kubectl create secret generic \
        --from-file=secret
```

### Methode 3

```yaml

apiVerion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_HOST: bXlzcWw=           # il faut les convertir en base64 avant de les mettre dans le manifest Secret
  DB_USER: YWRtaW4=
  DB_PASS: UEA0MjU2MHcwcmQ=

---
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret-demo
  labels:
    name: pod-secret-demo
spec:
  containers:
  - name: pod-secret-demo
    image: busybox
    ports:
      - conatnerPort: 8080
    envFrom:
      - secretRef:
          name: app-config
````

```bash
kubectl get secrets
kubectl describe secrets
```

> Next: [Ingress](../objects/ingress.md)

> [cheat sheet](../useful.md)
