# Introduction au Stockage dans Kubernetes

Le stockage est un aspect fondamental de la plupart des applications, en particulier pour les applications avec état (stateful) qui nécessitent de conserver des données au-delà du cycle de vie d'un Pod. Kubernetes offre un ensemble puissant de primitives pour gérer le stockage de manière flexible et découplée de l'exécution des conteneurs.

## Pourquoi le Stockage est-il Crucial ?

Dans un environnement conteneurisé comme Kubernetes, les conteneurs eux-mêmes sont souvent éphémères. Si un conteneur redémarre ou si un Pod est replanifié sur un autre nœud, toutes les données écrites directement dans le système de fichiers du conteneur sont perdues (sauf si un mécanisme de stockage externe est utilisé).

Pour les applications qui ont besoin de :
*   Conserver des données de manière persistante (bases de données, fichiers utilisateurs, etc.).
*   Partager des fichiers entre plusieurs conteneurs au sein d'un même Pod.
*   Accéder à des configurations ou des secrets montés en tant que fichiers.

Kubernetes fournit plusieurs abstractions pour résoudre ces défis. Les concepts clés que nous allons explorer sont les **Volumes**, les **PersistentVolumes (PV)**, les **PersistentVolumeClaims (PVC)**, et les **StorageClasses**.

---

## Volumes

Un **Volume** dans Kubernetes est essentiellement un répertoire, potentiellement avec des données, qui est accessible aux conteneurs d'un Pod.

*   **Cycle de Vie :** La durée de vie d'un Volume est liée à celle du Pod qui l'encapsule. Un Volume survit aux redémarrages des conteneurs au sein du Pod. Ainsi, les données stockées dans un Volume sont préservées tant que le Pod existe. Lorsque le Pod est supprimé, le Volume est également détruit (sauf pour certains types de volumes persistants).
*   **Partage :** Les Volumes peuvent être partagés entre les conteneurs d'un même Pod.

### Types de Volumes Courants

Kubernetes supporte de nombreux types de volumes. Voici quelques-uns des plus courants :

*   **`emptyDir` :**
    *   Crée un répertoire vide lorsque le Pod est assigné à un nœud.
    *   Le répertoire existe tant que le Pod s'exécute sur ce nœud.
    *   Si le Pod est supprimé, les données dans `emptyDir` sont définitivement perdues.
    *   Utile pour l'espace de travail temporaire, le partage de fichiers entre conteneurs d'un même Pod.
*   **`hostPath` :**
    *   Monte un fichier ou un répertoire du système de fichiers du nœud hôte directement dans le Pod.
    *   **Attention :** À utiliser avec une extrême prudence. Cela crée un couplage fort entre le Pod et le nœud, peut présenter des risques de sécurité, et n'est pas portable si le chemin n'existe pas sur tous les nœuds ou a un contenu différent.
    *   Cas d'usage : Agents de nœud (comme des collecteurs de logs) qui ont besoin d'accéder à des parties spécifiques du système de fichiers du nœud.
*   **`configMap` et `secret` :**
    *   Permettent d'exposer des données de [ConfigMaps](./configApp.md#configmaps) ou des [Secrets](./configApp.md#secrets) en tant que fichiers dans un Pod.
    *   Les fichiers sont en lecture seule par défaut.
    *   Utile pour injecter des fichiers de configuration, des certificats, ou des clés API.
*   **`persistentVolumeClaim` :**
    *   Permet à un Pod d'utiliser un **PersistentVolume** (PV), qui est une ressource de stockage persistante gérée par le cluster. C'est la méthode privilégiée pour le stockage durable des données applicatives. Nous allons détailler ce concept ci-dessous.

D'autres types de volumes existent pour des intégrations spécifiques avec des fournisseurs de stockage cloud (par exemple, `awsElasticBlockStore`, `azureDisk`, `gcePersistentDisk`) ou des systèmes de stockage réseau (par exemple, `nfs`, `cephfs`, `iscsi`).

---

## Concepts de Stockage Persistant (Aperçu)

Pour les données qui doivent survivre au cycle de vie d'un Pod, Kubernetes introduit trois concepts principaux :

### 1. PersistentVolume (PV)

*   Un **PersistentVolume (PV)** est un morceau de stockage dans le cluster qui a été provisionné par un administrateur ou dynamiquement provisionné à l'aide d'une `StorageClass`.
*   C'est une ressource du cluster, tout comme un nœud est une ressource du cluster. Les PVs sont des plugins de volume comme les Volumes, mais ont un cycle de vie indépendant de tout Pod individuel qui utilise le PV.
*   Un PV capture les détails de l'implémentation du stockage, que ce soit NFS, iSCSI, ou un stockage spécifique à un fournisseur de cloud.

### 2. PersistentVolumeClaim (PVC)

*   Un **PersistentVolumeClaim (PVC)** est une demande de stockage faite par un utilisateur (ou une application via sa définition de Pod).
*   Il est similaire à un Pod : les Pods consomment des ressources de nœud (CPU, mémoire) et les PVCs consomment des ressources de PV (stockage).
*   Les PVCs peuvent demander une taille spécifique et des modes d'accès (par exemple, `ReadWriteOnce`, `ReadOnlyMany`, `ReadWriteMany`).
    *   `ReadWriteOnce` (RWO) : Le volume peut être monté en lecture-écriture par un seul nœud.
    *   `ReadOnlyMany` (ROX) : Le volume peut être monté en lecture seule par plusieurs nœuds.
    *   `ReadWriteMany` (RWX) : Le volume peut être monté en lecture-écriture par plusieurs nœuds. (Supporté par certains types de stockage comme NFS).
*   Lorsqu'un PVC est créé, Kubernetes tente de trouver un PV disponible qui satisfait aux exigences du PVC (taille, modes d'accès, StorageClass). Si un PV correspondant est trouvé (ou peut être provisionné), le PVC est lié (bound) à ce PV. Le Pod peut alors utiliser ce PVC pour accéder au stockage.

### 3. StorageClass

*   Une **StorageClass** fournit un moyen pour les administrateurs de décrire les "classes" de stockage qu'ils offrent. Différentes classes peuvent correspondre à :
    *   Des niveaux de qualité de service (QoS) (par exemple, stockage SSD rapide vs. disques durs lents).
    *   Des politiques de sauvegarde.
    *   Des fournisseurs de stockage spécifiques ou des types de stockage (par exemple, `aws-ebs-gp3`, `nfs-fast`).
    *   Des politiques arbitraires déterminées par les administrateurs du cluster.
*   **Provisionnement Dynamique :** Le rôle principal des StorageClasses est de permettre le **provisionnement dynamique** des PersistentVolumes. Lorsqu'un PersistentVolumeClaim spécifie une StorageClass, et qu'aucun PV statique ne correspond, la StorageClass peut instruire le fournisseur de stockage sous-jacent (par exemple, AWS EBS, GCE PD, Azure Disk, ou un provisionneur interne comme NFS) de créer automatiquement un nouveau PV qui correspond aux besoins du PVC.

---

## Diagramme Conceptuel du Flux de Stockage

Voici comment ces composants interagissent :

1.  **Administrateur/Infrastructure :**
    *   (Optionnel, pour le provisionnement statique) L'administrateur crée des `PersistentVolumes (PV)` qui représentent le stockage disponible.
    *   L'administrateur définit des `StorageClasses` pour décrire les types de stockage et permettre le provisionnement dynamique.

2.  **Utilisateur/Développeur :**
    *   L'utilisateur crée un `PersistentVolumeClaim (PVC)` pour demander du stockage avec des spécifications (taille, mode d'accès, optionnellement une StorageClass).

3.  **Système Kubernetes :**
    *   Si une `StorageClass` est spécifiée dans le PVC et permet le provisionnement dynamique :
        `StorageClass --(provisionne dynamiquement)--> PersistentVolume (PV)`
    *   Kubernetes lie le `PVC` à un `PV` approprié (soit un PV existant qui correspond, soit un PV dynamiquement provisionné).
        `PVC --(se lie à)--> PV`

4.  **Pod :**
    *   Le Pod définit un `Volume` de type `persistentVolumeClaim` qui référence le `PVC`.
        `Pod --(utilise via un Volume)--> PVC`

**Flux simplifié :**
```
Pod (référence un PVC dans sa définition de Volume)
  |
  v
PersistentVolumeClaim (PVC) (demande de stockage)
  |
  v (est lié à)
PersistentVolume (PV) (représente une pièce de stockage réelle)
  |
  v
Stockage Physique/Cloud (ex: disque EBS, partage NFS, etc.)
```

**Avec Provisionnement Dynamique :**
```
Pod (référence un PVC)
  |
  v
PersistentVolumeClaim (PVC) (spécifie une StorageClass)
  |
  v (déclenche)
StorageClass (définit comment provisionner) --(crée)--> PersistentVolume (PV)
                                                            |
                                                            v (est lié au PVC ci-dessus)
                                                          Stockage Physique/Cloud
```

---

## Objectif de cette Section

Cette page a introduit les concepts de base du stockage dans Kubernetes. Les pages suivantes de ce tutoriel approfondiront :

*   **[PersistentVolumes (PV)](./persistentvolumes.md) :** Comment ils sont définis et gérés.
*   **[PersistentVolumeClaims (PVC)](./persistentvolumeclaims.md) :** Comment les utilisateurs demandent du stockage.
*   **[StorageClasses](./storageclasses.md) :** Pour le provisionnement dynamique et la classification du stockage.
*   **Utilisation des Volumes dans les Pods :** Exemples concrets de montage de PVCs.

Ces sections fourniront des exemples pratiques pour vous aider à configurer et utiliser le stockage persistant pour vos applications Kubernetes.

---

> Next: [PersistentVolumes (PV)](./persistentvolumes.md)

> [cheat sheet](../useful.md)
