# Labels et Sélecteurs Kubernetes

Les labels (étiquettes) et les sélecteurs sont des mécanismes fondamentaux dans Kubernetes pour organiser et sélectionner des groupes d'objets. Ils sont essentiels pour la gestion des ressources, le routage des services, et l'orchestration des déploiements.

## Labels

### Définition et Objectif

Les **labels** sont des paires clé-valeur qui sont attachées aux objets Kubernetes, tels que les Pods, Services, Deployments, ReplicaSets, etc.

*   **Objectif Principal :** Organiser et permettre la sélection de sous-ensembles d'objets.
*   **Pas d'Unicité :** Les labels ne sont pas destinés à fournir une unicité aux objets. De nombreux objets peuvent porter les mêmes labels. Ils sont utilisés pour le regroupement et le filtrage.
*   **Utilité :** Les utilisateurs et les composants du système peuvent interroger les objets en fonction de leurs labels.

### Syntaxe et Jeu de Caractères

Les clés et les valeurs des labels doivent respecter certaines règles :

*   **Clés de Label :**
    *   Doivent comporter au maximum 63 caractères.
    *   Peuvent contenir des lettres (minuscules et majuscules), des chiffres, des tirets (`-`), des underscores (`_`) et des points (`.`).
    *   Doivent commencer et se terminer par un caractère alphanumérique (lettre ou chiffre).
    *   Les clés peuvent également avoir un préfixe optionnel séparé par un `/`. Si un préfixe est utilisé, il doit être un sous-domaine DNS valide (par exemple, `kubernetes.io/`) et avoir au maximum 253 caractères. Le système Kubernetes réserve les préfixes `kubernetes.io/` et `k8s.io/`.
*   **Valeurs de Label :**
    *   Doivent comporter au maximum 63 caractères.
    *   Peuvent contenir des lettres (minuscules et majuscules), des chiffres, des tirets (`-`), des underscores (`_`) et des points (`.`).
    *   Doivent commencer et se terminer par un caractère alphanumérique.

### Cas d'Usage Courants

Voici quelques exemples de labels fréquemment utilisés pour organiser les ressources :

*   `environment: dev` / `environment: staging` / `environment: production` : Pour distinguer les environnements.
*   `app: mon-app-nom` : Pour identifier l'application à laquelle l'objet appartient.
*   `tier: frontend` / `tier: backend` / `tier: database` : Pour spécifier la couche applicative.
*   `release: stable` / `release: canary` / `release: beta` : Pour gérer les versions de déploiement.
*   `owner: equipe-a` / `project: projet-x` : Pour indiquer l'équipe responsable ou le projet associé.
*   `version: v1.2.3` : Pour spécifier la version d'un composant.

### Exemple de Définition de Labels dans un Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod-etiquete
  labels:
    environment: production
    app: nginx
    tier: frontend
    version: "1.21.6" # Les valeurs peuvent être des chaînes, même si elles ressemblent à des nombres
spec:
  containers:
  - name: nginx
    image: nginx:1.21.6 # Correspondance avec le label de version pour la clarté
    ports:
    - containerPort: 80
```

### Gestion des Labels avec `kubectl`

Vous pouvez gérer les labels des objets existants en utilisant `kubectl label` :

*   **Afficher les labels des pods :**
    ```bash
    kubectl get pods --show-labels
    ```
*   **Ajouter un nouveau label à un pod :**
    ```bash
    kubectl label pods mon-pod-etiquete nouveau-label=super
    ```
    (Si `mon-pod-etiquete` n'existe pas, cette commande retournera une erreur.)
*   **Modifier un label existant (nécessite `--overwrite`) :**
    ```bash
    kubectl label pods mon-pod-etiquete environment=staging --overwrite
    ```
*   **Supprimer un label d'un pod (en ajoutant un `-` à la fin du nom du label) :**
    ```bash
    kubectl label pods mon-pod-etiquete tier-
    ```

## Sélecteurs (Selectors)

### Définition

Les **sélecteurs** sont le mécanisme par lequel Kubernetes filtre et sélectionne les objets en fonction de leurs labels. Ils constituent la "requête" pour trouver des objets ayant des labels spécifiques.

### Types de Sélecteurs

Il existe deux principaux types de sélecteurs de labels :

1.  **Sélecteurs Basés sur l'Égalité (Equality-based) :**
    *   Ils filtrent en fonction de la correspondance exacte des clés et des valeurs de labels.
    *   Opérateurs supportés :
        *   `=` : égal (ou `==`)
        *   `!=` : différent
    *   Vous pouvez spécifier plusieurs conditions, qui sont combinées avec un opérateur logique ET (toutes les conditions doivent être vraies).
    *   Exemples :
        *   `environment = production` (sélectionne les objets où le label `environment` est `production`)
        *   `tier != frontend` (sélectionne les objets où le label `tier` n'est pas `frontend`)
        *   `environment=production,tier=frontend` (sélectionne les objets avec `environment=production` ET `tier=frontend`)

2.  **Sélecteurs Basés sur les Ensembles (Set-based) :**
    *   Ils filtrent en fonction d'un ensemble de valeurs pour une clé de label donnée, ou sur la présence/absence d'une clé.
    *   Opérateurs supportés :
        *   `in` : la valeur du label doit être dans l'ensemble spécifié.
        *   `notin` : la valeur du label ne doit pas être dans l'ensemble spécifié.
        *   `exists` : l'objet doit avoir un label avec la clé spécifiée (la valeur n'importe pas).
        *   `!exists` : l'objet ne doit PAS avoir de label avec la clé spécifiée.
    *   Exemples :
        *   `environment in (production, qa)` (sélectionne les objets où `environment` est `production` OU `qa`)
        *   `tier notin (frontend, backend)` (sélectionne les objets où `tier` n'est ni `frontend` ni `backend`)
        *   `exists version` (sélectionne les objets qui ont un label `version`)
        *   `!exists old-label` (sélectionne les objets qui n'ont pas le label `old-label`)

### Utilisation dans les Commandes `kubectl`

Les sélecteurs sont largement utilisés avec `kubectl get` pour lister des objets :

*   **Sélecteur basé sur l'égalité :**
    ```bash
    kubectl get pods -l environment=production
    kubectl get pods -l 'environment=production,tier=frontend' # Les guillemets sont utiles si la chaîne contient des virgules ou des espaces
    ```
*   **Sélecteur basé sur les ensembles :**
    ```bash
    kubectl get pods -l 'environment in (production, qa)'
    kubectl get pods -l 'exists(version)'
    kubectl get pods -l '!exists(deprecated-label)'
    kubectl get pods -l 'tier notin (frontend)'
    ```

### Utilisation dans les Définitions d'Objets Kubernetes

Les sélecteurs sont cruciaux dans les définitions d'objets pour lier différentes ressources entre elles.

#### Services

Un `Service` utilise un sélecteur pour déterminer à quel ensemble de Pods il doit acheminer le trafic. Le `Service` recherche les Pods dont les labels correspondent à son `spec.selector`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-app-service
spec:
  selector:
    app: mon-app      # Sélectionne les Pods avec le label "app: mon-app"
    tier: frontend   # ET le label "tier: frontend"
  ports:
    - protocol: TCP
      port: 80         # Port exposé par le Service
      targetPort: 8080 # Port sur lequel les Pods écoutent
```
Dans cet exemple, `mon-app-service` dirigera le trafic vers tous les Pods qui ont à la fois le label `app: mon-app` ET `tier: frontend`.

#### Deployments, ReplicaSets, StatefulSets, DaemonSets, Jobs

Ces contrôleurs utilisent des sélecteurs pour savoir quels Pods ils doivent gérer (créer, mettre à jour, supprimer).

*   `spec.selector` est un champ obligatoire pour ces objets.
*   Il est composé de `matchLabels` (pour les sélecteurs basés sur l'égalité) et/ou `matchExpressions` (pour les sélecteurs basés sur les ensembles).
*   **Important :** Les labels définis dans `spec.template.metadata.labels` du Pod template **doivent correspondre** au `spec.selector` du contrôleur. Si ce n'est pas le cas, la création de l'objet contrôleur sera rejetée par l'API Kubernetes. Cela garantit que le contrôleur gère bien les Pods qu'il crée.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app-deploiement
spec:
  replicas: 3
  selector: # Définit comment le Deployment trouve les Pods à gérer
    matchLabels: # Sélecteur basé sur l'égalité
      app: mon-app-backend
      tier: backend
    # Vous pourriez aussi utiliser matchExpressions pour des logiques plus complexes :
    # matchExpressions:
    #   - {key: tier, operator: In, values: [backend, worker]}
    #   - {key: environment, operator: NotIn, values: [dev]}
    #   - {key: critical, operator: Exists}
  template: # Modèle pour créer les Pods
    metadata:
      labels: # Les labels des Pods créés par ce Deployment
        app: mon-app-backend # Doit correspondre à matchLabels.app
        tier: backend        # Doit correspondre à matchLabels.tier
        # Vous pouvez ajouter d'autres labels ici qui ne sont pas dans le sélecteur
        version: "v1.0.1"
    spec:
      containers:
      - name: mon-app-conteneur
        image: mon-app-image:latest
        ports:
        - containerPort: 8080
```
Dans cet exemple, le Deployment `mon-app-deploiement` gérera 3 réplicas de Pods. Il identifiera ces Pods en utilisant le sélecteur `app=mon-app-backend,tier=backend`. Les Pods qu'il crée auront ces labels (plus `version: "v1.0.1"`), assurant ainsi qu'ils sont bien gérés par ce Deployment.

## Bonnes Pratiques pour les Labels et Sélecteurs

*   **Utilisez des labels significatifs et descriptifs :** Choisissez des clés et des valeurs qui décrivent clairement le rôle, l'état, ou l'appartenance de l'objet. Par exemple, `app: checkout-service`, `environment: production`, `release: canary`.
*   **N'abusez pas du nombre de labels :** Bien qu'il n'y ait pas de limite stricte (autre que la taille totale de l'objet), un trop grand nombre de labels peut rendre la gestion complexe. Concentrez-vous sur les labels essentiels pour l'organisation et la sélection.
*   **Soyez cohérent :** Utilisez les mêmes clés de labels pour des concepts similaires à travers vos applications et équipes. Par exemple, utilisez toujours `app` pour le nom de l'application, et non pas `application` ou `service-name` de manière interchangeable.
*   **Utilisez les labels pour les tâches opérationnelles :** Les labels sont très utiles pour des opérations comme les mises à jour progressives (rolling updates), les rollbacks, le monitoring ciblé, ou le dépannage. Par exemple, sélectionner tous les pods d'une version spécifique d'une application.
*   **Préférez les labels pour l'identification, les annotations pour les métadonnées non identifiantes :** Les labels sont faits pour être sélectionnés. Si vous avez des informations supplémentaires qui ne servent pas au filtrage (par exemple, un message de build, un contact téléphonique), utilisez plutôt les [Annotations](./annotations.md).

---

> Next: [Annotations](./annotations.md)

> [cheat sheet](../useful.md)
