# Role-Based Access Control (RBAC) dans Kubernetes

Le RBAC (Role-Based Access Control) est un mécanisme de sécurité fondamental dans Kubernetes qui permet de contrôler finement qui peut accéder à quelles ressources de l'API Kubernetes et quelles actions peuvent être effectuées sur ces ressources.

## Introduction au RBAC

### Qu'est-ce que le RBAC ?

Le RBAC est une méthode de régulation de l'accès aux ressources informatiques ou réseau basée sur les **rôles** des utilisateurs individuels au sein d'une organisation. Au lieu d'assigner des permissions directement aux utilisateurs, les permissions sont regroupées dans des rôles, et ces rôles sont ensuite assignés aux utilisateurs.

Dans Kubernetes, le "qui" peut être un utilisateur humain, un groupe d'utilisateurs, ou un **ServiceAccount** (une identité pour les processus s'exécutant dans les Pods). Le "quoi" (les actions) sont les verbes HTTP (`get`, `post`, `put`, `delete`) qui sont mappés à des verbes d'autorisation comme `list`, `create`, `update`, `patch`, `delete`, `watch`, etc. Les "ressources" sont les objets de l'API Kubernetes (Pods, Deployments, Services, Secrets, Nodes, etc.).

### Importance dans Kubernetes

Le RBAC est crucial pour la sécurité de Kubernetes car il permet d'appliquer le **principe du moindre privilège**. Ce principe stipule que chaque utilisateur, application ou composant du système ne doit disposer que des permissions strictement nécessaires pour accomplir ses tâches, et rien de plus.

Sans RBAC (ou avec des configurations RBAC trop permissives), un utilisateur ou une application compromis pourrait potentiellement prendre le contrôle de l'ensemble du cluster. Le RBAC permet de segmenter les permissions et de réduire considérablement la surface d'attaque.

Le RBAC est activé par défaut sur la plupart des clusters Kubernetes modernes (généralement depuis la version 1.6+).

---

## Objets Clés du RBAC

Le système RBAC de Kubernetes est basé sur quatre objets principaux : `Role`, `ClusterRole`, `RoleBinding`, et `ClusterRoleBinding`.

### 1. Role (Rôle)

*   Un **Role** est utilisé pour accorder des permissions sur des ressources **au sein d'un namespace spécifique**.
*   Il contient un ensemble de `rules` (règles) qui définissent les actions (verbes) autorisées sur un ensemble de ressources.

**Champs d'une Règle (`rules`) :**
*   **`apiGroups` :** Le groupe d'API de la ressource.
    *   `""` (chaîne vide) : Indique le groupe d'API principal (core), qui contient des ressources comme `pods`, `services`, `configmaps`, `secrets`.
    *   `apps` : Pour les ressources comme `deployments`, `statefulsets`, `daemonsets`.
    *   `batch` : Pour les ressources `jobs`, `cronjobs`.
    *   `networking.k8s.io` : Pour `ingresses`, `networkpolicies`.
    *   Et bien d'autres (par exemple, `storage.k8s.io` pour `storageclasses`, `rbac.authorization.k8s.io` pour les objets RBAC eux-mêmes).
*   **`resources` :** La liste des types de ressources auxquels la règle s'applique (par exemple, `pods`, `deployments`, `services`). Vous pouvez aussi spécifier des sous-ressources comme `pods/log` ou `deployments/scale`.
*   **`verbs` :** La liste des actions autorisées. Verbes courants :
    *   `get` : Lire une ressource spécifique.
    *   `list` : Lister toutes les ressources d'un type donné.
    *   `watch` : Surveiller les changements sur les ressources.
    *   `create` : Créer une nouvelle ressource.
    *   `update` : Mettre à jour une ressource existante.
    *   `patch` : Appliquer une modification partielle à une ressource.
    *   `delete` : Supprimer une ressource.
    *   `deletecollection` : Supprimer une collection de ressources.

**Exemple de YAML pour un Role :**
Ce Role `pod-reader` dans le namespace `my-namespace` permet de lire les Pods et leurs logs, et de lister les Deployments.
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: my-namespace
  name: pod-reader
rules:
- apiGroups: [""] # "" indique le groupe d'API principal (core)
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["list"]
```

### 2. ClusterRole

*   Un **ClusterRole** est similaire à un Role, mais il accorde des permissions **à l'échelle du cluster entier**.
*   Il est utilisé pour :
    *   Accorder des permissions sur des **ressources cluster-scoped** (qui ne sont pas dans un namespace), comme les Nœuds (`nodes`), les PersistentVolumes (`persistentvolumes`), les Namespaces (`namespaces`), les ClusterRoles eux-mêmes, etc.
    *   Accorder des permissions sur des **ressources namespacées, mais dans tous les namespaces** (par exemple, permettre à un administrateur de lister tous les Pods dans tous les namespaces).
    *   Accorder des permissions sur des **endpoints non-ressource** (par exemple, `/healthz`, `/metrics`).

**Exemple de YAML pour un ClusterRole :**
Ce ClusterRole `secret-reader` permet de lire les Secrets dans n'importe quel namespace et de lister les Nœuds.
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # Pas de champ 'namespace' pour un ClusterRole, car il est global
  name: secret-reader 
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
- apiGroups: [""]
  resources: ["nodes"] # Ressource cluster-scoped
  verbs: ["get", "list"]
```

### 3. RoleBinding (Liaison de Rôle)

*   Un **RoleBinding** accorde les permissions définies dans un `Role` à un ensemble de `subjects` (utilisateurs, groupes, ou ServiceAccounts) **au sein d'un namespace spécifique**.
*   Il lie un `Role` à des `subjects` dans le même namespace que le RoleBinding.

**Champs Clés :**
*   **`subjects` :** Une liste d'entités auxquelles les permissions sont accordées. Chaque sujet peut être :
    *   `kind: User` : Pour un utilisateur humain. `name` est le nom de l'utilisateur (sensible à la casse).
    *   `kind: Group` : Pour un groupe d'utilisateurs. `name` est le nom du groupe.
    *   `kind: ServiceAccount` : Pour un ServiceAccount. `name` est le nom du ServiceAccount, et `namespace` (optionnel ici, car le RoleBinding est déjà namespacé) est le namespace du ServiceAccount.
    *   `apiGroup: rbac.authorization.k8s.io` est utilisé pour `User` et `Group`.
*   **`roleRef` :** Référence le `Role` (ou un `ClusterRole`, bien que ce soit moins courant pour un RoleBinding) dont les permissions sont accordées.
    *   `kind: Role` : Indique que `name` fait référence à un Role.
    *   `name` : Le nom du Role (doit exister dans le même namespace que le RoleBinding).
    *   `apiGroup: rbac.authorization.k8s.io` : Le groupe d'API pour les objets RBAC.

**Exemple de YAML pour un RoleBinding :**
Ce RoleBinding `read-pods-in-my-namespace` accorde les permissions du Role `pod-reader` (défini précédemment) à un utilisateur, un groupe, et un ServiceAccount dans `my-namespace`.
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-in-my-namespace
  namespace: my-namespace # Le RoleBinding est dans ce namespace
subjects: # À qui les permissions sont accordées
- kind: User
  name: "jane.doe@example.com" # Nom d'utilisateur (sensible à la casse)
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: "developers" # Nom du groupe
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: "my-service-account" # Nom du ServiceAccount
  namespace: "my-namespace" # Doit être dans le même namespace que le RoleBinding
roleRef: # Quel rôle est accordé
  kind: Role # Le type de rôle (Role ou ClusterRole)
  name: pod-reader # Nom du Role dans 'my-namespace'
  apiGroup: rbac.authorization.k8s.io
```

### 4. ClusterRoleBinding (Liaison de Rôle de Cluster)

*   Un **ClusterRoleBinding** accorde les permissions définies dans un `ClusterRole` (ou parfois un `Role`) à des `subjects` **à l'échelle du cluster entier**.
*   Il est utilisé pour accorder des permissions sur des ressources cluster-scoped, ou sur des ressources namespacées dans tous les namespaces.

**Exemple de YAML pour un ClusterRoleBinding :**
Ce ClusterRoleBinding `read-secrets-global` accorde les permissions du ClusterRole `secret-reader` (défini précédemment) au groupe `auditors` pour l'ensemble du cluster.
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  # Pas de champ 'namespace' pour un ClusterRoleBinding
  name: read-secrets-global
subjects:
- kind: Group
  name: "auditors"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole # Peut être Role ou ClusterRole. Généralement ClusterRole pour un accès global.
  name: secret-reader # Nom du ClusterRole
  apiGroup: rbac.authorization.k8s.io
```
**Note :** Vous pouvez utiliser un ClusterRoleBinding pour lier un `Role` à des sujets cluster-wide. Dans ce cas, le `Role` sera appliqué à chaque namespace individuellement pour ces sujets. Cependant, il est plus courant d'utiliser un ClusterRoleBinding avec un ClusterRole pour des permissions véritablement globales.

---

## Sujets (Subjects) : Utilisateurs, Groupes, ServiceAccounts

Les permissions RBAC sont accordées à des "sujets" :

*   **Utilisateurs (Users) :**
    *   Représentent des personnes humaines qui interagissent avec le cluster (développeurs, administrateurs).
    *   Kubernetes **ne gère pas** les comptes utilisateurs de manière native. Il n'y a pas d'objet API pour les utilisateurs.
    *   L'identité de l'utilisateur est généralement fournie par un système d'authentification externe (certificats clients, tokens de porteur, OpenID Connect (OIDC), etc.). Le nom d'utilisateur est la chaîne de caractères que ce système externe fournit à l'API Server.
*   **Groupes (Groups) :**
    *   Une collection d'utilisateurs. Comme les utilisateurs, les groupes sont gérés en dehors de Kubernetes.
    *   Le système d'authentification peut fournir des informations de groupe pour un utilisateur.
    *   Exemples de groupes intégrés : `system:authenticated` (tous les utilisateurs authentifiés), `system:unauthenticated` (utilisateurs anonymes).
*   **ServiceAccounts (Comptes de Service) :**
    *   Sont des identités pour les **processus s'exécutant à l'intérieur des Pods** lorsqu'ils ont besoin d'accéder à l'API Kubernetes.
    *   Ils sont gérés par Kubernetes et sont des objets API (vous pouvez les créer, les lister, etc.).
    *   Chaque namespace a un ServiceAccount par défaut nommé `default`. Si un Pod ne spécifie pas de `serviceAccountName`, il utilise le ServiceAccount `default` de son namespace.
    *   Les ServiceAccounts sont souvent utilisés pour accorder des permissions granulaires aux applications (par exemple, un Pod qui a besoin de lister d'autres Pods dans son namespace).

**Création d'un ServiceAccount :**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: my-app-ns
# Vous pouvez aussi ajouter des annotations ou des labels ici
# imagePullSecrets: # Si ce SA a besoin de tirer des images depuis des registres privés
# - name: my-registry-secret
```
Pour l'utiliser, un Pod dans le namespace `my-app-ns` spécifierait `spec.serviceAccountName: my-app-sa`.

---

## Commandes `kubectl` Courantes pour RBAC

*   **Vérifier si une action est autorisée (`kubectl auth can-i`) :**
    Permet de tester si un utilisateur ou un ServiceAccount a la permission d'effectuer une action.
    ```bash
    # En tant que vous-même (votre identité kubectl actuelle)
    kubectl auth can-i list pods --namespace my-namespace
    kubectl auth can-i create deployments --namespace my-namespace
    kubectl auth can-i get nodes # Pour une ressource cluster-scoped

    # En tant qu'un autre utilisateur
    kubectl auth can-i list secrets --namespace dev --as jane.doe@example.com

    # En tant qu'un ServiceAccount
    kubectl auth can-i get configmaps --namespace prod --as system:serviceaccount:prod:my-sa
    ```

*   **Lister les objets RBAC :**
    ```bash
    kubectl get roles --namespace my-namespace
    kubectl get rolebindings --namespace my-namespace

    kubectl get clusterroles
    kubectl get clusterrolebindings

    # Pour voir tous les rôles/bindings dans tous les namespaces (si vous avez les droits)
    kubectl get roles --all-namespaces
    kubectl get rolebindings --all-namespaces
    ```

*   **Décrire un objet RBAC (voir ses détails, y compris les règles) :**
    ```bash
    kubectl describe role pod-reader --namespace my-namespace
    kubectl describe clusterrole secret-reader

    kubectl describe rolebinding read-pods-in-my-namespace --namespace my-namespace
    kubectl describe clusterrolebinding read-secrets-global
    ```

---

## Bonnes Pratiques pour RBAC

*   **Principe du Moindre Privilège :** C'est la règle d'or. N'accordez que les permissions strictement nécessaires pour la tâche à accomplir. Évitez d'accorder des permissions larges comme `cluster-admin` à des utilisateurs ou des ServiceAccounts à moins que ce ne soit absolument indispensable.
*   **Utiliser les Roles et RoleBindings pour les Permissions Namespacées :** Pour les permissions qui ne concernent qu'un seul namespace, préférez toujours `Role` et `RoleBinding`.
*   **Utiliser les ClusterRoles et ClusterRoleBindings avec Parcimonie :** N'accordez des permissions à l'échelle du cluster que lorsque c'est réellement nécessaire (par exemple, pour des administrateurs de cluster, des contrôleurs qui opèrent sur tout le cluster).
*   **Favoriser les ServiceAccounts pour les Applications :** Ne laissez pas les Pods utiliser le ServiceAccount `default` s'ils ont besoin d'interagir avec l'API Kubernetes. Créez des ServiceAccounts dédiés avec des permissions RBAC minimales. N'utilisez pas de comptes utilisateurs humains pour les applications.
*   **Réviser Régulièrement les Politiques RBAC :** Les besoins changent. Auditez périodiquement vos configurations RBAC pour vous assurer qu'elles sont toujours appropriées et pour supprimer les permissions inutiles.
*   **Être Spécifique dans les Règles :**
    *   Évitez d'utiliser des wildcards (`*`) pour `apiGroups`, `resources`, ou `verbs` autant que possible. Soyez aussi spécifique que possible.
    *   Au lieu de donner `get`, `list`, `watch`, `create`, `update`, `patch`, `delete` sur `*` ressources dans `*` apiGroups, ciblez précisément ce qui est nécessaire.
*   **Noms de Rôles Descriptifs :** Utilisez des noms clairs pour vos Roles et ClusterRoles qui indiquent leur fonction (par exemple, `configmap-editor`, `log-viewer`, `deployment-manager`).
*   **Utiliser des Groupes :** Lorsque c'est possible (si votre système d'authentification le supporte), gérez les permissions pour des groupes d'utilisateurs plutôt que pour des utilisateurs individuels. Cela simplifie l'administration.

---

> Next: [Security Contexts](./security-contexts.md)

> [cheat sheet](../useful.md)
