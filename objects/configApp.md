# Gestion de la Configuration des Applications

Dans Kubernetes, la configuration des applications et la gestion des données sensibles (comme les mots de passe ou les clés API) sont découplées du code de l'application. Cela se fait principalement à l'aide de deux types d'objets : `ConfigMaps` pour les données de configuration non sensibles, et `Secrets` pour les informations sensibles.

---

# ConfigMaps

Les ConfigMaps sont utilisés pour stocker des données de configuration non sensibles sous forme de paires clé-valeur. Ces configurations peuvent ensuite être injectées dans les Pods, soit en tant que variables d'environnement, soit en tant que fichiers dans un volume.

## Création des ConfigMaps

Il existe plusieurs façons de créer des ConfigMaps :

### Méthode 1 : À partir de littéraux (valeurs directes)

```bash
kubectl create configmap app-config-literal \
 --from-literal=APP_VERSION=1.0 \
 --from-literal=APP_ENV=production
```
Cela crée un ConfigMap nommé `app-config-literal` avec deux clés : `APP_VERSION` et `APP_ENV`.

### Méthode 2 : À partir d'un fichier

Supposons que vous ayez un fichier `app_config.properties` :
```properties
# Contenu de app_config.properties
APP_COLOR=blue
APP_MODE=standalone
```
Vous pouvez créer un ConfigMap à partir de ce fichier :
```bash
kubectl create configmap app-config-file \
 --from-file=app_config.properties
```
Cela créera un ConfigMap nommé `app-config-file` où le nom du fichier (`app_config.properties`) devient une clé, et le contenu du fichier sa valeur. Pour utiliser les clés du fichier directement, utilisez `--from-env-file`.

### Méthode 3 : À partir d'un fichier YAML (Déclaratif)

Vous pouvez définir un ConfigMap directement dans un fichier YAML :
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-yaml
data:
  # Les données sont des paires clé-valeur
  LOG_LEVEL: "info"
  API_URL: "https://api.example.com/v1"
  FEATURE_FLAG_NEW_UI: "true"
```
Appliquez ce fichier avec `kubectl apply -f <nom-du-fichier.yaml>`.

## Consommer les ConfigMaps

Une fois créés, les ConfigMaps peuvent être consommés par les Pods de plusieurs manières.

### 1. Injecter toutes les clés comme Variables d'Environnement (`envFrom`)

C'est une méthode courante pour exposer toutes les données d'un ConfigMap comme variables d'environnement dans un conteneur.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-envfrom-demo
spec:
  containers:
  - name: mon-conteneur
    image: busybox # Ou votre image d'application
    command: [ "/bin/sh", "-c", "env | grep APP_" ] # Commande pour afficher les variables d'env
    envFrom:
      - configMapRef:
          name: app-config-literal # Nom du ConfigMap à utiliser
```
Dans cet exemple, les variables `APP_VERSION` et `APP_ENV` du ConfigMap `app-config-literal` seront disponibles dans le conteneur.

### 2. Injecter des Clés Spécifiques comme Variables d'Environnement (`env.valueFrom`)

Vous pouvez injecter des valeurs spécifiques d'un ConfigMap comme variables d'environnement.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-env-specific-demo
spec:
  containers:
  - name: mon-conteneur
    image: busybox
    command: [ "/bin/sh", "-c", "env | grep MY_APP" ]
    env:
      - name: MY_APP_VERSION # Nom de la variable d'environnement dans le Pod
        valueFrom:
          configMapKeyRef:
            name: app-config-literal # Nom du ConfigMap
            key: APP_VERSION        # Clé à récupérer dans le ConfigMap
      - name: MY_APP_ENVIRONMENT
        valueFrom:
          configMapKeyRef:
            name: app-config-literal
            key: APP_ENV
```

### 3. Monter un ConfigMap comme Volume

Chaque clé du ConfigMap devient un fichier dans le répertoire de montage. C'est utile pour les fichiers de configuration que les applications s'attendent à lire depuis le système de fichiers.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-volume-demo
spec:
  containers:
  - name: mon-conteneur
    image: busybox
    command: [ "/bin/sh", "-c", "ls /etc/config/ && cat /etc/config/LOG_LEVEL" ]
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config # Chemin où le ConfigMap sera monté
  volumes:
  - name: config-volume
    configMap:
      name: app-config-yaml # Nom du ConfigMap à monter
      # Optionnel: spécifier quelles clés monter et sous quels noms de fichiers
      # items:
      # - key: API_URL
      #   path: api_url.txt # Monte la clé API_URL vers /etc/config/api_url.txt
      # - key: LOG_LEVEL
      #   path: log_level_setting # Monte la clé LOG_LEVEL vers /etc/config/log_level_setting
```
Si `items` n'est pas spécifié, toutes les clés du ConfigMap `app-config-yaml` (par exemple, `LOG_LEVEL`, `API_URL`, `FEATURE_FLAG_NEW_UI`) deviendront des fichiers portant ces noms dans `/etc/config/`. Si `items` est utilisé, seuls les éléments spécifiés sont montés, et `path` définit le nom du fichier dans le volume.

**Mise à jour des ConfigMaps :** Si un ConfigMap monté comme volume est mis à jour, les fichiers montés dans les Pods sont également mis à jour dynamiquement (généralement en quelques secondes à une minute). Cependant, l'application doit être capable de détecter ces changements et de recharger sa configuration. Les variables d'environnement ne sont pas mises à jour dynamiquement après le démarrage du Pod.

### ConfigMaps Immuables

Depuis Kubernetes 1.19 (stable en 1.21), vous pouvez créer des ConfigMaps immuables en définissant `immutable: true` dans leur `metadata`.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-immuable
  immutable: true # Empêche toute modification ultérieure de ce ConfigMap
data:
  config.setting: "valeur-fixe"
```
*   **Avantages :**
    *   Protège contre les modifications accidentelles.
    *   Peut améliorer les performances du cluster en réduisant la charge sur l'API server (kube-apiserver n'a plus besoin de surveiller les changements sur ces ConfigMaps).

---

# Secrets

Les objets Secret sont utilisés pour stocker et gérer des informations sensibles, telles que les mots de passe, les tokens OAuth, les clés SSH, ou les certificats TLS.

**Important :** Par défaut, les données des Secrets dans etcd (la base de données de Kubernetes) sont encodées en base64. **Base64 est un encodage, pas un chiffrement.** Pour protéger réellement les Secrets, il est crucial d'activer le chiffrement au repos (encryption at rest) pour les Secrets dans etcd et de contrôler l'accès aux Secrets via RBAC.

## Création des Secrets

### Méthode 1 : À partir de littéraux

```bash
kubectl create secret generic app-secret-literal \
 --from-literal=DB_USER=admin \
 --from-literal=DB_PASSWORD='S!p3rS3cr3t'
```

### Méthode 2 : À partir de fichiers

Supposons que vous ayez des fichiers `username.txt` et `password.txt` :
```bash
# Contenu de username.txt
admin_user
# Contenu de password.txt
P@@sswOrd!

kubectl create secret generic app-secret-files \
 --from-file=DB_USER_FILE=./username.txt \
 --from-file=DB_PASS_FILE=./password.txt
```
Les clés dans le Secret seront `DB_USER_FILE` et `DB_PASS_FILE`.

### Méthode 3 : À partir d'un fichier YAML (Déclaratif)

Les valeurs dans le champ `data` doivent être encodées en base64.
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret-yaml
type: Opaque # Type de Secret par défaut
data:
  # Les valeurs sont encodées en base64
  # echo -n 'monutilisateur' | base64  -> bW9udXRpbGlzYXRldXI=
  # echo -n 'MotDePasseTresSecret' | base64 -> TW90RGVQYXNzZVRyZXNTZWNyZXQ=
  DB_USERNAME: bW9udXRpbGlzYXRldXI=
  DB_PASSWORD: TW90RGVQYXNzZVRyZXNTZWNyZXQ=
```
Appliquez avec `kubectl apply -f <nom-du-fichier.yaml>`.

## Consommer les Secrets

Similaire aux ConfigMaps, les Secrets peuvent être injectés dans les Pods.

### 1. Injecter tous les Secrets comme Variables d'Environnement (`envFrom`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret-envfrom-demo
spec:
  containers:
  - name: mon-conteneur
    image: busybox
    command: [ "/bin/sh", "-c", "env | grep DB_" ]
    envFrom:
      - secretRef:
          name: app-secret-literal # Nom du Secret
```
Les variables `DB_USER` et `DB_PASSWORD` seront disponibles dans le conteneur.

### 2. Injecter des Clés Spécifiques comme Variables d'Environnement (`env.valueFrom`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret-env-specific-demo
spec:
  containers:
  - name: mon-conteneur
    image: busybox
    command: [ "/bin/sh", "-c", "env | grep DATABASE_" ]
    env:
      - name: DATABASE_USERNAME # Nom de la variable d'environnement dans le Pod
        valueFrom:
          secretKeyRef:
            name: app-secret-literal # Nom du Secret
            key: DB_USER            # Clé à récupérer dans le Secret
      - name: DATABASE_PASSWORD
        valueFrom:
          secretKeyRef:
            name: app-secret-literal
            key: DB_PASSWORD
```

### 3. Monter un Secret comme Volume

C'est souvent la méthode préférée pour les données sensibles, car elle évite de les exposer en tant que variables d'environnement (qui peuvent être loguées ou inspectées plus facilement).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret-volume-demo
spec:
  containers:
  - name: mon-conteneur
    image: busybox
    command: [ "/bin/sh", "-c", "ls /etc/app-secrets/ && cat /etc/app-secrets/DB_PASSWORD" ]
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/app-secrets # Chemin où le Secret sera monté
      readOnly: true # Bonne pratique pour les volumes de Secrets
  volumes:
  - name: secret-volume
    secret:
      secretName: app-secret-literal # Nom du Secret à monter
      # Optionnel: spécifier quelles clés monter et sous quels noms de fichiers
      # items:
      # - key: DB_USER
      #   path: username.txt # Monte la clé DB_USER vers /etc/app-secrets/username.txt
      # defaultMode: 0400 # Permissions pour les fichiers (si non spécifié, 0644)
```
Chaque clé du Secret devient un fichier dans le répertoire `/etc/app-secrets/`. L'utilisation de `readOnly: true` est une bonne pratique.

## Types de Secrets

Kubernetes supporte plusieurs types de Secrets, spécifiés par le champ `type` dans la définition du Secret.

*   **`Opaque` :** (Défaut) Secret arbitraire défini par l'utilisateur. Les données ne sont soumises à aucune validation de format.
*   **`kubernetes.io/service-account-token` :** Utilisé pour stocker un token qui identifie un [ServiceAccount](../serviceAccount.md). Ces Secrets sont automatiquement créés par Kubernetes pour chaque ServiceAccount et montés dans les Pods qui utilisent ce ServiceAccount.
*   **`kubernetes.io/dockerconfigjson` :** Utilisé pour stocker les identifiants d'un registre Docker privé. Lorsque vous créez un Secret de ce type, Kubernetes l'utilise pour authentifier les tirages d'images (`image pull`) pour les Pods.
    *   Création :
        ```bash
        kubectl create secret docker-registry my-docker-registry-secret \
          --docker-server=<votre-serveur-de-registre> \
          --docker-username=<votre-utilisateur> \
          --docker-password=<votre-mot-de-passe> \
          --docker-email=<votre-email>
        ```
    *   Utilisation dans un Pod (pour tirer des images privées) :
        ```yaml
        apiVersion: v1
        kind: Pod
        metadata:
          name: pod-avec-image-privee
        spec:
          containers:
          - name: mon-conteneur-prive
            image: <votre-registre-prive>/mon-image:tag
          imagePullSecrets:
          - name: my-docker-registry-secret # Nom du Secret de type dockerconfigjson
        ```
*   **Autres types :** Il existe d'autres types pour des cas d'usage spécifiques comme `kubernetes.io/tls` pour les certificats TLS, `kubernetes.io/ssh-auth` pour les clés SSH, etc.

## Bonnes Pratiques pour les Secrets

*   **Activer le Chiffrement au Repos (Encryption at Rest) :** Configurez votre cluster Kubernetes pour chiffrer les ressources Secret dans etcd. C'est une configuration de niveau cluster.
*   **Utiliser RBAC (Role-Based Access Control) :** Limitez strictement qui peut lire, créer, ou modifier les Secrets. Accordez les permissions `get`, `list`, `watch` pour les Secrets uniquement aux principaux (utilisateurs, ServiceAccounts) qui en ont absolument besoin.
*   **Éviter de Stocker des Secrets en Clair dans Git :** Ne commitez pas de manifestes YAML de Secrets contenant des données sensibles en clair dans votre dépôt Git. Utilisez des solutions comme :
    *   **Sealed Secrets** (par Bitnami) : Permet de chiffrer les Secrets avant de les commiter. Un contrôleur dans le cluster les déchiffre.
    *   **SOPS** (par Mozilla) : Un éditeur de fichiers chiffrés qui supporte YAML, JSON, etc., et peut s'intégrer avec des services KMS (Key Management Service).
    *   **HashiCorp Vault** ou d'autres gestionnaires de secrets externes avec des intégrations Kubernetes.
*   **Préférer les Volumes aux Variables d'Environnement :** Si l'application le supporte, montez les Secrets comme des volumes de fichiers plutôt que de les injecter comme variables d'environnement. Les variables d'environnement peuvent être plus facilement exposées (par exemple, dans les logs d'erreurs ou via `kubectl describe pod`). Les volumes de Secrets peuvent également être mis à jour dynamiquement si le Secret change (bien que l'application doive relire le fichier).
*   **Limiter la Durée de Vie des Secrets :** Pour les tokens ou les informations d'identification temporaires, mettez en place des mécanismes de rotation.
*   **Utiliser des Identités de Charge de Travail (Workload Identities) :** Pour interagir avec des services cloud (AWS, GCP, Azure), préférez utiliser des mécanismes d'identité de charge de travail (comme IAM Roles for Service Accounts sur AWS/GCP) plutôt que de stocker des clés d'accès cloud à long terme dans des Secrets Kubernetes.

---

> Next: [Ingress](../objects/ingress.md)

> [cheat sheet](../useful.md)
