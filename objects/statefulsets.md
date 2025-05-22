# StatefulSets Kubernetes

Les StatefulSets sont des objets de l'API Kubernetes utilisés pour gérer les applications avec état. Ils sont conçus pour les applications qui nécessitent des identifiants réseau stables, un stockage persistant stable, un déploiement et une mise à l'échelle ordonnés et gracieux.

## Introduction aux StatefulSets

### Que sont-ils ?

Un **StatefulSet** est un contrôleur Kubernetes qui gère le déploiement et la mise à l'échelle d'un ensemble de Pods et garantit l'ordre et l'unicité de ces Pods. Contrairement à un Deployment, un StatefulSet maintient une identité persistante pour chacun de ses Pods.

### Pourquoi sont-ils nécessaires ? (Stateful vs. Stateless)

*   **Deployments (Stateless) :** Les Deployments sont idéaux pour les applications sans état (stateless). Les Pods gérés par un Deployment sont interchangeables ; ils n'ont pas d'identité fixe. Si un Pod est remplacé, il obtient un nouveau nom, une nouvelle adresse IP, et n'est pas lié à un volume de stockage spécifique de manière persistante. C'est parfait pour les serveurs web ou les applications backend qui ne stockent pas de données localement.
*   **StatefulSets (Stateful) :** Les applications avec état (stateful), comme les bases de données ou les systèmes de messagerie distribués, nécessitent que leurs instances (Pods) aient une identité stable et un stockage persistant qui leur est propre. Si une instance de base de données `db-0` tombe et est recréée, elle doit revenir en tant que `db-0` et se reconnecter à son volume de données spécifique. C'est ce que permettent les StatefulSets.

### Caractéristiques Clés

*   **Identifiants Réseau Stables et Uniques :** Chaque Pod dans un StatefulSet reçoit un nom d'hôte ordinal et persistant. Par exemple, si le StatefulSet s'appelle `web`, les Pods seront nommés `web-0`, `web-1`, `web-2`, etc. Ces noms sont conservés même si les Pods sont redémarrés ou replanifiés. L'adresse DNS de chaque Pod est également stable (par exemple, `web-0.my-headless-service.my-namespace.svc.cluster.local`).
*   **Stockage Persistant Stable :** Chaque Pod peut être associé à un volume de stockage persistant unique (PersistentVolumeClaim). Ce stockage "colle" au Pod, même en cas de redémarrages. Si `web-0` est redémarré, il se rattachera au même PersistentVolume.
*   **Déploiement et Mise à l'Échelle Ordonnés et Gracieux :**
    *   **Déploiement :** Les Pods sont créés séquentiellement, un par un. Le Pod `web-0` est créé et devient prêt avant que `web-1` ne soit lancé, et ainsi de suite.
    *   **Mise à l'échelle (Scaling Up) :** Lorsque vous augmentez le nombre de réplicas, les nouveaux Pods sont ajoutés dans l'ordre (par exemple, si vous passez de 2 à 4 réplicas, `web-2` sera créé, puis `web-3`).
*   **Suppression et Terminaison Ordonnées et Gracieuses :**
    *   **Mise à l'échelle (Scaling Down) :** Lorsque vous réduisez le nombre de réplicas, les Pods sont terminés dans l'ordre inverse. Si vous avez 3 Pods (`web-0`, `web-1`, `web-2`) et que vous réduisez à 1 réplica, `web-2` sera terminé en premier, puis `web-1`.
    *   **Suppression :** La suppression des Pods suit également cet ordre inverse.

## Cas d'Usage

Les StatefulSets sont particulièrement adaptés pour :

*   **Bases de Données Distribuées et Clusterisées :**
    *   MySQL, PostgreSQL (surtout en mode réplication primaire-secondaire)
    *   Bases de données NoSQL comme Cassandra, MongoDB, Elasticsearch, CockroachDB.
*   **Systèmes de Messagerie (Queues) :**
    *   Apache Kafka, RabbitMQ, Zookeeper.
*   **Applications avec État Nécessitant des Leaders/Followers :**
    *   Consul, etcd.
*   **Toute application qui requiert :**
    *   Un identifiant réseau stable par instance.
    *   Un stockage persistant dédié par instance.
    *   Un ordre de démarrage et d'arrêt contrôlé.

## Composants et Concepts Clés

### Service "Headless" (Sans Tête)

Un StatefulSet nécessite généralement un Service "headless" pour contrôler le domaine réseau de ses Pods. Un service headless ne fait pas de répartition de charge (load balancing) et ne possède pas d'adresse IP de cluster (ClusterIP). Au lieu de cela, il retourne les adresses IP des Pods qu'il sélectionne.

*   **Objectif :** Fournir des enregistrements DNS uniques pour chaque Pod du StatefulSet, permettant aux Pods de se découvrir et de communiquer entre eux via des noms d'hôtes stables. Par exemple, `my-stateful-app-0.my-headless-svc.my-namespace.svc.cluster.local`.
*   **Configuration :** Pour rendre un Service headless, vous devez définir `spec.clusterIP` à `None`.

**Exemple de Service Headless :**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-headless-svc # Ce nom sera utilisé dans spec.serviceName du StatefulSet
  labels:
    app: my-stateful-app
spec:
  ports:
  - port: 80
    name: web # Nom du port (optionnel mais bonne pratique)
  clusterIP: None # Rend le service headless
  selector:
    app: my-stateful-app # Doit correspondre aux labels des Pods du StatefulSet
```

### VolumeClaimTemplates (Modèles de Revendication de Volume)

Pour fournir un stockage persistant stable à chaque Pod, les StatefulSets utilisent `volumeClaimTemplates`.

*   **Fonctionnement :** Pour chaque Pod créé par le StatefulSet, un PersistentVolumeClaim (PVC) est automatiquement créé à partir du modèle défini dans `volumeClaimTemplates`.
*   **Nommage :** Le nom du PVC est une combinaison du nom du volume dans le template et du nom du Pod. Par exemple, si le template de volume s'appelle `data` et le Pod `app-0`, le PVC sera `data-app-0`.
*   **Persistance :** Chaque PVC est lié à un PersistentVolume (PV). Lorsque le Pod est replanifié, il se rattache à son PVC et donc à son PV existant, conservant ainsi ses données.
*   **Suppression :** Par défaut, lorsque vous supprimez un StatefulSet ou réduisez le nombre de réplicas, les PVCs ne sont **pas** supprimés automatiquement (pour protéger les données). Vous devez les supprimer manuellement si vous souhaitez libérer le stockage. Ce comportement peut être modifié via la politique `persistentVolumeClaimRetentionPolicy`.

## Exemple de StatefulSet YAML

Voici un exemple simple de StatefulSet utilisant Nginx, qui monte un volume persistant pour chaque Pod.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-stateful-app
spec:
  serviceName: "my-headless-svc" # Doit correspondre au nom du Service headless créé précédemment
  replicas: 3 # Nombre de Pods souhaités
  selector:
    matchLabels:
      app: my-stateful-app # Doit correspondre à .spec.template.metadata.labels
  template: # Modèle pour les Pods
    metadata:
      labels:
        app: my-stateful-app # Doit correspondre à .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10 # Temps accordé au Pod pour s'arrêter proprement
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web # Doit correspondre au port nommé dans le Service headless si utilisé
        volumeMounts:
        - name: www # Nom du volume défini dans volumeClaimTemplates
          mountPath: /usr/share/nginx/html # Chemin de montage dans le conteneur
  volumeClaimTemplates: # Modèle pour les PersistentVolumeClaims
  - metadata:
      name: www # Nom du volume, sera utilisé pour nommer le PVC (ex: www-my-stateful-app-0)
    spec:
      accessModes: [ "ReadWriteOnce" ] # Mode d'accès. RWO est typique pour un Pod unique.
                                      # Pour des bases de données partagées, on pourrait voir RWX (ReadWriteMany)
                                      # si le fournisseur de stockage le supporte.
      storageClassName: "standard" # Nom de la StorageClass à utiliser.
                                   # Assurez-vous qu'elle existe ou utilisez la classe par défaut.
      resources:
        requests:
          storage: 1Gi # Taille du volume à provisionner pour chaque Pod
```
**Avant d'appliquer ce YAML :**
1.  Assurez-vous d'avoir un Service headless nommé `my-headless-svc` (comme l'exemple précédent).
2.  Vérifiez que vous avez une `StorageClass` nommée `standard` ou remplacez par une classe existante dans votre cluster, ou que le provisionnement dynamique est configuré.

## Considérations de Déploiement et de Mise à l'Échelle

*   **Déploiement Ordonné :** Les Pods sont créés un par un, de l'indice 0 à N-1. Le Pod `my-stateful-app-0` sera créé et devra être `Running` et `Ready` avant que `my-stateful-app-1` ne soit tenté, et ainsi de suite. Cela est utile pour les applications qui nécessitent qu'un "leader" ou un "master" soit disponible avant que les "followers" ou "workers" ne démarrent.
*   **Mise à l'Échelle Ordonnée (Scaling Up) :** Si vous augmentez `spec.replicas` de 3 à 5, `my-stateful-app-3` sera créé en premier, puis `my-stateful-app-4`, en respectant l'ordre.
*   **Mise à l'Échelle Ordonnée (Scaling Down) :** Si vous réduisez `spec.replicas` de 5 à 2, `my-stateful-app-4` sera terminé en premier, puis `my-stateful-app-3`. Le Pod avec l'indice le plus élevé est supprimé en premier. Chaque Pod attend que le précédent soit complètement arrêté avant de commencer sa propre terminaison.

## Mise à Jour des StatefulSets

Les StatefulSets supportent plusieurs stratégies de mise à jour, définies dans `spec.updateStrategy.type` :

*   **`RollingUpdate` (Par Défaut) :**
    *   C'est la stratégie par défaut. Lorsque vous modifiez le `spec.template` du StatefulSet (par exemple, pour changer l'image du conteneur), les Pods sont mis à jour de manière ordonnée et progressive.
    *   Les Pods sont mis à jour dans l'ordre inverse de leur indice (de N-1 à 0). Un Pod est supprimé, puis recréé avec la nouvelle configuration. Le StatefulSet attend que le Pod mis à jour soit `Running` et `Ready` avant de passer au suivant.
    *   Vous pouvez contrôler le comportement avec `spec.updateStrategy.rollingUpdate.partition`. Si `partition` est spécifié, tous les Pods avec un ordinal supérieur ou égal à la valeur de `partition` seront mis à jour. Les Pods avec un ordinal inférieur ne seront pas affectés. Cela permet des mises à jour canary ou des déploiements par étapes.
*   **`OnDelete` :**
    *   Avec cette stratégie, le contrôleur StatefulSet ne met pas automatiquement à jour les Pods lorsque le `spec.template` change.
    *   Pour appliquer la nouvelle configuration, vous devez supprimer manuellement les Pods existants. Le StatefulSet les recréera alors avec la nouvelle version. Cette méthode donne un contrôle plus direct sur le moment de la mise à jour de chaque Pod.

## Limitations

*   **Complexité :** Les StatefulSets sont plus complexes à configurer et à gérer que les Deployments en raison de leurs garanties d'ordre, d'identité et de stockage.
*   **Provisionnement du Stockage :** Le stockage persistant (PersistentVolumes) doit être disponible. Si vous n'utilisez pas de provisionnement dynamique avec une `StorageClass`, vous devrez créer les PVs manuellement.
*   **Dépendances :** Fortement dépendant de la disponibilité et de la performance du fournisseur de stockage sous-jacent.
*   **Services Headless :** La nécessité d'un service headless ajoute un objet supplémentaire à gérer.
*   **Mises à Jour :** Bien que les mises à jour soient ordonnées, elles peuvent être lentes pour les grands StatefulSets, car chaque Pod doit être mis à jour séquentiellement.

---

> Next: [DaemonSets](./daemonsets.md)

> [cheat sheet](../useful.md)
