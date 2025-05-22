# Ingress Kubernetes

Un Ingress est un objet de l'API Kubernetes qui gère l'accès externe aux services au sein d'un cluster, généralement pour le trafic HTTP et HTTPS. Il permet de définir des règles de routage basées sur l'hôte (nom de domaine) ou le chemin (URL), et peut fournir des fonctionnalités telles que la terminaison TLS et l'équilibrage de charge.

<center><img src="../images/nginx-ingress-controller.png" alt="Ingress Controller Architecture" width="500" height="300"/></center>

## Prérequis : Le Contrôleur Ingress

Pour qu'une ressource Ingress fonctionne, le cluster doit avoir un **Contrôleur Ingress** en cours d'exécution. Le contrôleur Ingress est une application (souvent un reverse proxy comme Nginx, Traefik, HAProxy, etc.) qui surveille les ressources Ingress dans le cluster et configure l'équilibreur de charge sous-jacent en conséquence.

*   **Rôle :** Il lit les règles définies dans les objets Ingress et les traduit en configuration pour un proxy inverse (par exemple, Nginx).
*   **Choix :** Il existe de nombreux contrôleurs Ingress (par exemple, Nginx Ingress Controller, Traefik, HAProxy Ingress, GKE Ingress, AWS ALB Ingress Controller). Le choix dépendra de votre environnement et de vos besoins.
*   **Déploiement :** Le déploiement d'un contrôleur Ingress est une étape distincte de la création des ressources Ingress. (Un exemple de déploiement du contrôleur Nginx est fourni plus bas).

Une fois qu'un contrôleur Ingress est déployé dans votre cluster, vous pouvez créer des objets Ingress pour exposer vos services.

---

## Définition d'une Ressource Ingress

Une ressource Ingress définit comment le trafic externe doit être acheminé vers les services internes. Voici les configurations courantes :

### 1. Ingress Simple (Fanout Basé sur le Chemin)

Cet exemple montre comment router le trafic vers différents services en fonction du chemin de l'URL.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-fanout-example
  annotations:
    # Annotation courante pour Nginx Ingress Controller
    # Indique que la partie du chemin correspondant à la règle ne doit pas être transmise au backend.
    # Par exemple, une requête à /foo/quelquechose deviendrait /quelquechose pour service-foo.
    # Si vous voulez que /foo soit transmis, omettez ou ajustez cette annotation.
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /foo # Trafic pour mondomaine.com/foo
        pathType: Prefix 
        backend:
          service:
            name: service-foo # Service qui gère les requêtes pour /foo
            port:
              number: 80 # Port du service-foo
      - path: /bar # Trafic pour mondomaine.com/bar
        pathType: Prefix
        backend:
          service:
            name: service-bar # Service qui gère les requêtes pour /bar
            port:
              number: 80 # Port du service-bar
```

**Explication des Champs Clés :**
*   **`apiVersion: networking.k8s.io/v1` :** Version de l'API pour les objets Ingress (obligatoire).
*   **`metadata.annotations` :** Les annotations sont utilisées pour configurer des options spécifiques au contrôleur Ingress utilisé. `nginx.ingress.kubernetes.io/rewrite-target: /` est une annotation courante pour le contrôleur Nginx, qui réécrit le chemin avant de le transmettre au service backend.
*   **`spec.rules` :** Une liste de règles de routage.
    *   **`http.paths` :** Définit les chemins et leurs backends.
    *   **`path` :** Le chemin de l'URL (par exemple, `/foo`).
    *   **`pathType` :** Spécifie comment le chemin doit être interprété.
        *   `Prefix` : Correspond aux URL qui commencent par le `path` spécifié (par exemple, `/foo` correspondra à `/foo`, `/foo/`, `/foo/abc`).
        *   `Exact` : Correspond exactement au `path` spécifié (par exemple, `/foo` ne correspondra qu'à `/foo`, pas à `/foo/`).
        *   `ImplementationSpecific` : Le comportement dépend du contrôleur Ingress. Il est recommandé d'utiliser `Prefix` ou `Exact`.
    *   **`backend.service.name` :** Nom du Service Kubernetes vers lequel le trafic doit être dirigé.
    *   **`backend.service.port.number` :** Port du Service.

### 2. Hébergement Virtuel Basé sur le Nom (Name-based Virtual Hosting)

Cet exemple montre comment router le trafic vers différents services en fonction du nom d'hôte (domaine) dans l'en-tête HTTP `Host`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: foo.example.com # Trafic destiné à foo.example.com
    http:
      paths:
      - path: / # Toutes les requêtes pour foo.example.com/
        pathType: Prefix
        backend:
          service:
            name: service-foo # Service pour foo.example.com
            port:
              number: 80
  - host: bar.example.com # Trafic destiné à bar.example.com
    http:
      paths:
      - path: / # Toutes les requêtes pour bar.example.com/
        pathType: Prefix
        backend:
          service:
            name: service-bar # Service pour bar.example.com
            port:
              number: 80
```
Avec cette configuration, les requêtes vers `foo.example.com` sont envoyées à `service-foo`, tandis que les requêtes vers `bar.example.com` sont envoyées à `service-bar`. Vous devrez configurer vos DNS pour que ces noms d'hôtes pointent vers l'adresse IP de votre contrôleur Ingress.

### 3. Terminaison TLS

L'Ingress peut terminer les connexions TLS (HTTPS), ce qui signifie que le contrôleur Ingress gère le certificat TLS et déchiffre le trafic avant de le transmettre aux services backend (qui peuvent alors écouter en HTTP simple).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls: # Section pour la configuration TLS
  - hosts:
      - myapp.example.com # Nom d'hôte pour lequel le certificat est valide
    secretName: myapp-tls-secret # Nom du Secret Kubernetes contenant le certificat et la clé privée
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service
            port:
              number: 80 # Le service backend peut écouter en HTTP
```

**Création du Secret TLS :**
Pour que cela fonctionne, vous devez d'abord créer un Secret Kubernetes de type `kubernetes.io/tls` contenant votre certificat (`tls.crt`) et votre clé privée (`tls.key`).
```bash
kubectl create secret tls myapp-tls-secret --key chemin/vers/votre/cle.key --cert chemin/vers/votre/cert.pem
# Exemple: kubectl create secret tls myapp-tls-secret --key key.pem --cert cert.pem
```
Le certificat et la clé doivent être au format PEM. Le Secret `myapp-tls-secret` doit exister dans le même namespace que l'Ingress, ou être un Secret de cluster si le contrôleur le supporte.

### 4. Backend par Défaut (Default Backend)

Vous pouvez définir un backend par défaut pour votre Ingress. Ce backend gère toutes les requêtes qui ne correspondent à aucune des règles spécifiées (par exemple, un hôte ou un chemin non défini). C'est utile pour afficher une page d'erreur personnalisée "404 Not Found" ou une page d'accueil par défaut.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-with-default-backend
spec:
  defaultBackend: # Backend par défaut si aucune règle ne correspond
    service:
      name: default-service # Service qui gère les requêtes non appariées
      port:
        number: 80
  rules:
  - host: specific.example.com
    http:
      paths:
      - path: /app
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
  # Les requêtes vers specific.example.com/autre ou autre.example.com
  # seront dirigées vers default-service.
```

### 5. Classe d'Ingress (Ingress Class)

Lorsque plusieurs contrôleurs Ingress sont présents dans un cluster (par exemple, un pour le trafic public et un pour le trafic interne), ou si vous voulez explicitement spécifier quel contrôleur doit gérer un Ingress, vous utilisez une **IngressClass**.

*   **Depuis Kubernetes 1.19+ :** Le champ `spec.ingressClassName` est la méthode recommandée.
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: ingress-with-class
    spec:
      ingressClassName: "nginx-public" # Nom de la ressource IngressClass
      rules:
      # ...
    ```
    Une ressource `IngressClass` correspondante doit être définie dans le cluster :
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: IngressClass
    metadata:
      name: nginx-public
    spec:
      controller: "k8s.io/ingress-nginx" # Identifiant du contrôleur Ingress
    ```

*   **Avant Kubernetes 1.19 (Méthode par Annotation) :** L'annotation `kubernetes.io/ingress.class` était utilisée.
    ```yaml
    apiVersion: networking.k8s.io/v1 # Ou extensions/v1beta1 pour les très anciennes versions
    kind: Ingress
    metadata:
      name: ingress-with-annotation-class
      annotations:
        kubernetes.io/ingress.class: "nginx-internal"
    spec:
      # ...
    ```
Chaque contrôleur Ingress écoute les Ingress qui correspondent à sa classe. Si `ingressClassName` n'est pas spécifié et qu'il n'y a pas d'IngressClass marquée comme défaut, le comportement peut varier.

---

## Exemple : Mise en Place d'un Contrôleur Ingress NGINX (Simplifié)

**Note :** Ce qui suit est un exemple simplifié pour illustrer les composants. Pour un déploiement en production, référez-vous toujours à la documentation officielle du contrôleur Ingress NGINX. Les manifestes peuvent évoluer.

Le contrôleur Ingress NGINX lui-même est généralement déployé en tant que `Deployment` et exposé via un `Service` de type `LoadBalancer` ou `NodePort`.

### 1. Déploiement du Contrôleur Nginx Ingress
(Ce YAML est basé sur l'ancien exemple, des ajustements pour les versions API modernes et la configuration RBAC seraient nécessaires pour un déploiement actuel.)

```yaml
# Attention : Ce YAML est simplifié et potentiellement obsolète.
# Pour une installation Nginx Ingress, suivez leur guide officiel :
# https://kubernetes.github.io/ingress-nginx/deploy/
apiVersion: apps/v1 # Anciennement extensions/v1beta1 ou apps/v1beta2
kind: Deployment # Modifié de "Deployement" à "Deployment"
metadata:
  name: nginx-ingress-controller
  # namespace: ingress-nginx # Il est recommandé de le déployer dans son propre namespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-ingress # Corrigé de "name" à "app" pour la cohérence
  template:
    metadata: # Corrigé de "matadata" à "metadata"
      labels:
        app: nginx-ingress # Corrigé de "name" à "app"
    spec:
      # serviceAccountName: ingress-nginx # Un ServiceAccount avec les permissions RBAC nécessaires est requis
      containers:
        - name: nginx-ingress-controller
          image: registry.k8s.io/ingress-nginx/controller:v1.8.0 # Image plus récente
          args:
            - /nginx-ingress-controller
            # - --configmap=$(POD_NAMESPACE)/nginx-configuration # Méthode plus ancienne
            - --ingress-class=nginx # Recommandé pour spécifier la classe
            # - --publish-service=$(POD_NAMESPACE)/nginx-ingress # Recommandé
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
            # - name: metrics
            #   containerPort: 10254
```

### 2. ConfigMap pour la Configuration Nginx (Optionnel)
Un ConfigMap peut être utilisé pour personnaliser le comportement de Nginx.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configuration
  # namespace: ingress-nginx
# data:
  # Exemples de configurations personnalisées pour Nginx
  # use-forwarded-headers: "true"
  # ssl-protocols: "TLSv1.3 TLSv1.2"
```

### 3. Service pour Exposer le Contrôleur Nginx Ingress
Ce Service expose le contrôleur Ingress au trafic externe.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
  # namespace: ingress-nginx
spec:
  type: LoadBalancer # Ou NodePort, selon votre environnement (cloud ou bare-metal)
  ports:
  - port: 80
    targetPort: 80 # Doit correspondre à containerPort http du Deployment
    protocol: TCP
    name: http
  - port: 443
    targetPort: 443 # Doit correspondre à containerPort https du Deployment
    protocol: TCP
    name: https
  selector:
    app: nginx-ingress # Doit correspondre aux labels du Pod du contrôleur
```
**Note Importante sur le Déploiement du Contrôleur :**
Le déploiement d'un contrôleur Ingress (comme NGINX) implique généralement la création de plusieurs autres ressources, y compris :
*   Un Namespace dédié (par exemple, `ingress-nginx`).
*   Des ServiceAccounts, Roles, ClusterRoles, RoleBindings, et ClusterRoleBindings pour accorder les permissions RBAC nécessaires au contrôleur pour interagir avec l'API Kubernetes (lire les Ingress, Secrets, Services, etc.).
*   Un ConfigMap pour la configuration globale du contrôleur.
*   Des ValidatingWebhookConfigurations pour valider les ressources Ingress.

**Il est fortement recommandé d'utiliser les méthodes d'installation officielles (par exemple, Helm ou les manifestes fournis par le projet NGINX Ingress) pour déployer le contrôleur.**

---

> Next: [NetworkPolicies](./networkpolicies.md)

> [cheat sheet](../useful.md)
