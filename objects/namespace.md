# Namespace (Espace de Noms)

Kubernetes prend en charge plusieurs clusters virtuels sauvegardés par le même cluster physique. Ces clusters virtuels sont appelés `namespaces` (espaces de noms). Ils constituent un moyen fondamental pour organiser les clusters en environnements logiquement isolés.

Par défaut, Kubernetes démarre avec quelques namespaces :
*   `default`: L'espace de noms par défaut pour les objets sans autre espace de noms assigné.
*   `kube-system`: Pour les objets créés par le système Kubernetes.
*   `kube-public`: Cet espace de noms est lisible par tous les utilisateurs (y compris ceux non authentifiés). Il est principalement réservé à un usage interne au cluster.
*   `kube-node-lease`: Pour les objets de bail de nœud qui déterminent la disponibilité des nœuds.

## Cas d'Usage des Namespaces

Les namespaces sont un outil puissant pour diviser et gérer les ressources d'un cluster. Voici quelques cas d'usage courants :

*   **Isolation des Environnements :** C'est l'usage le plus fréquent. On peut créer des namespaces distincts pour les environnements de développement (`dev`), de pré-production (`preprod`), et de production (`prod`). Cela permet de déployer et de gérer différentes versions d'applications sans interférences.
*   **Séparation par Équipes ou Projets :** Dans les grandes organisations, plusieurs équipes peuvent partager un même cluster Kubernetes. Les namespaces permettent à chaque équipe (par exemple, `equipe-frontend`, `equipe-backend`) ou à chaque projet (`projet-alpha`, `projet-beta`) de disposer de son propre espace de travail isolé. Cela aide à éviter les conflits de noms et à gérer les ressources de manière indépendante.
*   **Gestion des Applications Multiples :** Si vous hébergez plusieurs applications distinctes sur le même cluster, vous pouvez les assigner à des namespaces différents pour une meilleure organisation et pour appliquer des politiques spécifiques à chaque application.
*   **Portée des Noms :** Il est crucial de comprendre que la plupart des ressources Kubernetes (comme les Pods, Services, Deployments, Secrets, ConfigMaps) sont confinées à un namespace. Le nom d'une ressource doit être unique au sein d'un namespace, mais pas nécessairement à travers tous les namespaces. Par exemple, vous pouvez avoir un Pod nommé `mon-app` dans le namespace `dev` et un autre Pod nommé `mon-app` dans le namespace `prod`.
    *   **Ressources non namespacées :** Certaines ressources sont globales au cluster et ne sont pas associées à un namespace. Celles-ci incluent les Nœuds (Nodes), les PersistentVolumes (PV), les StorageClasses, et les Namespaces eux-mêmes.

## Bonnes Pratiques

Pour utiliser efficacement les namespaces, il est recommandé de suivre certaines bonnes pratiques :

*   **ResourceQuotas (Quotas de Ressources) :**
    *   Les `ResourceQuotas` permettent de définir des limites sur la quantité de ressources (CPU, mémoire, nombre de pods, etc.) qu'un namespace peut consommer.
    *   Ceci est essentiel pour éviter qu'une équipe ou une application "affamée" ne monopolise toutes les ressources du cluster, garantissant une répartition équitable. Le fichier contient déjà un exemple de ResourceQuota (voir ci-dessous).
*   **NetworkPolicies (Politiques Réseau) :**
    *   Les `NetworkPolicies` permettent de contrôler le flux de trafic réseau entre les pods au sein d'un namespace, ainsi que vers et depuis d'autres namespaces ou points externes.
    *   Bien que nous ayons une section dédiée plus tard, il est important de noter ici que les NetworkPolicies sont scopées au namespace.
*   **Accès Réseau par Défaut :**
    *   Par défaut, tous les pods dans tous les namespaces d'un cluster Kubernetes peuvent communiquer entre eux sans restriction.
    *   Pour isoler les namespaces au niveau réseau (par exemple, pour empêcher les pods de `dev` de communiquer avec ceux de `prod`), vous devez implémenter des `NetworkPolicies`.
*   **Conventions de Nommage :**
    *   Adoptez des conventions de nommage claires et cohérentes pour vos namespaces. Cela facilite grandement la gestion et la compréhension de l'organisation du cluster.
    *   Exemples : `equipe-a-dev`, `equipe-a-prod`, `projet-x-testing`, `app-nomapp-staging`.

<center><img src="../images/namespace.png" alt="arch" width="400" height="200"/></center>

## Gestion des Namespaces avec `kubectl`

Voici les commandes `kubectl` courantes pour interagir avec les namespaces :

*   **Lister les namespaces :**
    ```bash
    kubectl get namespaces
    # Alias: kubectl get ns
    ```
*   **Créer un namespace (impératif) :**
    ```bash
    kubectl create namespace mon-nouveau-namespace
    ```
*   **Décrire un namespace (voir détails et labels) :**
    ```bash
    kubectl describe namespace mon-nouveau-namespace
    # Alias: kubectl describe ns mon-nouveau-namespace
    ```
*   **Supprimer un namespace :**
    Attention : La suppression d'un namespace supprime **toutes** les ressources qu'il contient !
    ```bash
    kubectl delete namespace mon-nouveau-namespace
    # Alias: kubectl delete ns mon-nouveau-namespace
    ```
*   **Travailler dans un namespace spécifique :**
    *   Pour la plupart des commandes `kubectl`, vous pouvez spécifier le namespace avec le flag `-n` ou `--namespace` :
        ```bash
        kubectl get pods --namespace=dev
        # ou
        kubectl get pods -n dev
        ```
    *   Pour lister les ressources dans tous les namespaces :
        ```bash
        kubectl get pods --all-namespaces
        # ou
        kubectl get pods -A
        ```
*   **Changer le contexte actuel pour un namespace :**
    Pour éviter de taper `--namespace` à chaque fois, vous pouvez définir le namespace par défaut pour votre contexte `kubectl` actuel :
    ```bash
    kubectl config set-context $(kubectl config current-context) --namespace=mon-namespace-de-travail
    ```
    Après cette commande, toutes les commandes `kubectl` s'appliqueront par défaut à `mon-namespace-de-travail`. Pour revenir au namespace `default`, réexécutez la commande avec `default`.

## Exemple YAML de Création de Namespace

Voici un exemple de fichier YAML pour créer un namespace de manière déclarative. L'utilisation de labels est une bonne pratique pour organiser et sélectionner des namespaces.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mon-espace-perso
  labels:
    equipe: "backend-devs"
    environnement: "developpement"
    projet: "nouveau-projet-secret"
```

Pour créer le namespace à partir de ce fichier (par exemple, `mon-namespace.yaml`) :
```bash
kubectl create -f mon-namespace.yaml
```
Les labels peuvent ensuite être utilisés pour sélectionner des namespaces, par exemple : `kubectl get namespaces -l environnement=developpement`.

## Exemple de Pod dans un Namespace Spécifique

Lorsque vous définissez un objet Kubernetes, vous pouvez spécifier le namespace auquel il appartient via le champ `metadata.namespace`.

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo
  namespace: dev  # Spécifie que ce pod appartient au namespace 'dev'
  labels:
    app: myapp
    type: front-end
spec:
  containers:
    - name:  nginx-container
      image: nginx
```

Si le namespace `dev` n'existe pas au moment de la création du pod, la commande échouera, sauf si vous créez le pod et le namespace en même temps (par exemple, dans un seul fichier YAML ou via des commandes successives).

Pour créer ce pod :
```bash
# Assurez-vous que le namespace 'dev' existe d'abord :
# kubectl create namespace dev
# ou via un fichier YAML comme montré précédemment.

kubectl create -f pod-dans-namespace.yaml # en supposant que le YAML ci-dessus est dans ce fichier
```

## DNS Inter-Namespace

Kubernetes crée des entrées DNS pour les services. Un service peut être atteint depuis n'importe quel autre namespace en utilisant son nom complet : `nom-du-service.nom-du-namespace.svc.cluster.local`.
Par exemple, un service `database-sql` dans le namespace `dev` aura une adresse DNS `database-sql.dev.svc.cluster.local`.

## Quotas de Ressources (ResourceQuota)

Pour contrôler la consommation des ressources au sein d'un namespace, on utilise les `ResourceQuotas`.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev # Applique ce quota au namespace 'dev'
spec:
  hard:
    pods: "10"                # Nombre maximum de pods
    requests.cpu: "4"         # Somme totale des CPU demandés par les pods ne peut pas dépasser 4
    requests.memory: 5Gi      # Somme totale de la mémoire demandée ne peut pas dépasser 5Gi
    limits.cpu: "10"          # Somme totale des limites CPU ne peut pas dépasser 10
    limits.memory: 10Gi       # Somme totale des limites mémoire ne peut pas dépasser 10Gi
    configmaps: "10"          # Nombre maximum de ConfigMaps
    secrets: "10"             # Nombre maximum de Secrets
    services.loadbalancers: "2" # Nombre maximum de Services de type LoadBalancer
```

Pour créer ce quota :
```bash
# Assurez-vous que le namespace 'dev' existe.
kubectl create -f compute-quota.yaml --namespace=dev
```
Si un utilisateur tente de créer une ressource dans le namespace `dev` qui excéderait ces quotas, la création échouera.

> Next: [Labels et Sélecteurs](./labels-selectors.md)

> [cheat sheet](../useful.md)
