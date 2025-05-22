# StorageClasses Kubernetes

Une StorageClass est un objet de l'API Kubernetes qui permet aux administrateurs de définir différentes "classes" de stockage qu'ils offrent. Elles sont un composant clé pour le provisionnement dynamique des PersistentVolumes (PVs).

## Introduction aux StorageClasses

### Définition

Une **StorageClass** fournit un moyen pour les administrateurs de décrire les "classes" (ou types) de stockage qu'ils proposent dans un cluster Kubernetes. Chaque StorageClass représente une configuration de stockage spécifique. Différentes classes peuvent correspondre à :

*   Des niveaux de qualité de service (QoS) (par exemple, stockage SSD rapide vs. disques durs standards).
*   Des politiques de sauvegarde.
*   Des fournisseurs de stockage spécifiques ou des types de stockage (par exemple, `aws-ebs-gp3`, `nfs-fast`, `ceph-rbd`).
*   Des politiques arbitraires déterminées par les administrateurs du cluster (par exemple, `billing-code=department-x`).

### Objectif Principal : Provisionnement Dynamique

L'objectif principal des StorageClasses est de permettre le **provisionnement dynamique** des PersistentVolumes (PVs).

*   **Sans Provisionnement Dynamique (Statique) :** Un administrateur de cluster doit créer manuellement des PVs. Ces PVs représentent le stockage existant disponible pour être utilisé par les [PersistentVolumeClaims (PVCs)](./persistentvolumeclaims.md). Si aucun PV statique ne correspond à un PVC, le PVC reste en attente (`Pending`).
*   **Avec Provisionnement Dynamique :** Au lieu que les administrateurs créent des PVs à l'avance, une StorageClass peut être configurée pour utiliser un **provisionneur** (un plugin de volume) qui créera automatiquement un nouveau PV lorsqu'un PVC le demande. Le PVC doit spécifier le nom de la StorageClass à utiliser.

Cela simplifie grandement la gestion du stockage, car les utilisateurs n'ont plus besoin d'attendre qu'un administrateur crée un PV pour eux. Ils demandent simplement du stockage via un PVC en référençant une StorageClass, et le stockage est provisionné à la volée.

---

## Champs Clés et Concepts d'une StorageClass

Un manifeste de StorageClass contient plusieurs champs importants :

### `provisioner` (Obligatoire)

*   Détermine quel plugin de volume est utilisé pour provisionner les PVs. Ce champ est **obligatoire**.
*   Chaque fournisseur de stockage (cloud ou sur site) a son propre provisionneur.
*   **Exemples de Provisionneurs Courants :**
    *   `kubernetes.io/aws-ebs` : Pour les volumes Amazon Web Services Elastic Block Store.
    *   `kubernetes.io/gce-pd` : Pour les Google Compute Engine Persistent Disks.
    *   `kubernetes.io/azure-disk` : Pour les Microsoft Azure Managed Disks.
    *   `kubernetes.io/azure-file` : Pour les partages de fichiers Microsoft Azure.
    *   `kubernetes.io/cinder-block-storage` (ou anciennement `kubernetes.io/cinder`) : Pour OpenStack Cinder.
    *   `kubernetes.io/vsphere-volume` : Pour VMware vSphere.
    *   Provisionneurs pour des systèmes de stockage sur site comme Ceph RBD, GlusterFS, Portworx, etc. (par exemple, `ceph.com/rbd`, `kubernetes.io/glusterfs`).
    *   `your-company.com/nfs` : Pour un provisionneur NFS externe ou personnalisé.
*   **Note sur NFS :** Kubernetes lui-même ne fournit **pas** de provisionneur NFS interne. Si vous souhaitez un provisionnement dynamique pour NFS, vous devez déployer un provisionneur NFS externe (par exemple, le [NFS subdir external provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)).

### `parameters` (Optionnel)

*   Contient des paramètres spécifiques au provisionneur défini dans le champ `provisioner`. Ces paramètres sont opaques pour Kubernetes et sont interprétés uniquement par le provisionneur.
*   La nature de ces paramètres varie considérablement d'un provisionneur à l'autre.
*   **Exemples de Paramètres :**
    *   Pour `kubernetes.io/aws-ebs` :
        *   `type`: Type de volume EBS (par exemple, `gp2`, `gp3`, `io1`, `st1`, `sc1`).
        *   `fsType`: Type de système de fichiers à créer (par exemple, `ext4`, `xfs`). Par défaut, `ext4`.
        *   `encrypted`: `"true"` ou `"false"` pour activer/désactiver le chiffrement EBS.
        *   `iopsPerGB`: Pour les volumes `io1`/`io2`/`gp3`, le nombre d'IOPS par GiB.
    *   Pour `kubernetes.io/gce-pd` :
        *   `type`: Type de disque (par exemple, `pd-standard`, `pd-ssd`).
        *   `replication-type`: Pour les disques régionaux (par exemple, `regional-pd`), mettez à `regional-pd`. Sinon, `none`.
        *   `fsType`: `ext4` ou `xfs`.
    *   Pour un provisionneur NFS externe, les paramètres pourraient inclure le serveur NFS, le chemin de base, etc., bien que ces informations soient souvent configurées dans le déploiement du provisionneur lui-même.

### `reclaimPolicy` (Optionnel)

*   Spécifie la politique de récupération (`persistentVolumeReclaimPolicy`) pour les PVs qui sont dynamiquement créés par cette StorageClass.
*   Valeurs possibles :
    *   **`Delete` :** Lorsque le PVC est supprimé, le PV dynamiquement provisionné et le volume de stockage sous-jacent (par exemple, le disque EBS, le disque GCE) sont également supprimés. C'est souvent la politique par défaut pour les StorageClasses.
    *   **`Retain` :** Lorsque le PVC est supprimé, le PV dynamiquement provisionné n'est pas supprimé. Il passe à l'état `Released` et doit être manuellement récupéré par un administrateur (ce qui peut impliquer la suppression manuelle des données sur le volume sous-jacent, puis la suppression du PV).
*   Si ce champ est omis dans la StorageClass, il est généralement par défaut à `Delete`.

### `volumeBindingMode` (Optionnel)

*   Contrôle quand la liaison du PVC au PV (et donc le provisionnement dynamique) doit se produire.
*   Valeurs possibles :
    *   **`Immediate` (par défaut) :** Le provisionnement dynamique et la liaison se produisent dès que le PVC est créé.
    *   **`WaitForFirstConsumer` :** Le provisionnement dynamique et la liaison sont retardés jusqu'à ce qu'un Pod qui utilise ce PVC soit créé et planifié.
        *   **Utilité :** Ce mode est essentiel pour le stockage sensible à la topologie (par exemple, s'assurer qu'un volume EBS est créé dans la même zone de disponibilité que le nœud où le Pod sera planifié). Il permet au planificateur Kubernetes de prendre en compte les contraintes du Pod (comme les sélecteurs de nœuds, les affinités/anti-affinités) avant de décider où et comment provisionner le volume.

### `allowVolumeExpansion` (Optionnel)

*   Si mis à `true`, les PVCs créés à partir de cette StorageClass peuvent être agrandis après leur création.
*   **Conditions :**
    *   Le provisionneur de volume sous-jacent doit supporter l'expansion de volume.
    *   Le PV doit avoir été créé avec `allowVolumeExpansion: true` (ce qui est hérité de la StorageClass).
*   Pour agrandir un PVC, vous modifiez le champ `spec.resources.requests.storage` du PVC avec la nouvelle taille souhaitée.

### `mountOptions` (Optionnel)

*   Spécifie une liste d'options de montage par défaut pour les PVs qui sont dynamiquement provisionnés par cette StorageClass.
*   Ces options sont passées à l'outil de montage du système d'exploitation lorsque le volume est monté sur un nœud.
*   Exemple : `mountOptions: ["debug", "hard", "nfsvers=4.1"]` (les options spécifiques dépendent du type de volume).

---

## Exemple de StorageClass

Voici un exemple de manifeste pour une StorageClass qui utilise le provisionneur AWS EBS.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd-aws # Nom de la StorageClass, utilisé par les PVCs
provisioner: kubernetes.io/aws-ebs # Indique que nous utilisons le provisionneur AWS EBS
parameters: # Paramètres spécifiques au provisionneur AWS EBS
  type: gp3        # Type de volume EBS : General Purpose SSD v3
  fsType: ext4     # Créer un système de fichiers ext4 sur le volume
  # encrypted: "true" # Optionnel: pour chiffrer le volume EBS
  # iopsPerGB: "50"   # Optionnel: pour gp3, peut être utilisé pour provisionner des IOPS spécifiques
reclaimPolicy: Delete # Lorsque le PVC est supprimé, le volume EBS sous-jacent est également supprimé
allowVolumeExpansion: true # Permet d'augmenter la taille des volumes créés avec cette classe
volumeBindingMode: Immediate # Provisionner et lier le volume dès que le PVC est créé
# mountOptions: # Optionnel: options de montage pour les volumes
#   - debug
```

---

## StorageClass par Défaut

Un cluster Kubernetes peut avoir une StorageClass marquée comme **par défaut**.

*   **Comportement :** Si un PersistentVolumeClaim (PVC) est créé sans spécifier de `storageClassName`, la StorageClass par défaut du cluster sera automatiquement utilisée pour le provisionnement dynamique (si le provisionnement dynamique est possible et qu'aucun PV statique ne correspond).
*   **Comment Marquer comme Défaut :** Pour désigner une StorageClass comme étant celle par défaut, vous devez ajouter l'annotation `storageclass.kubernetes.io/is-default-class: "true"` à sa définition `metadata`.
    ```yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: standard-gp2
      annotations:
        storageclass.kubernetes.io/is-default-class: "true" # Marque cette SC comme défaut
    provisioner: kubernetes.io/aws-ebs
    parameters:
      type: gp2
    reclaimPolicy: Delete
    # ... autres champs
    ```
*   **Unicité :** Il ne devrait y avoir qu'une seule StorageClass marquée comme défaut dans un cluster. Si plusieurs StorageClasses sont marquées comme défaut, Kubernetes peut ne pas se comporter comme prévu lors du traitement des PVCs sans `storageClassName` spécifié. Les nouvelles versions de Kubernetes peuvent empêcher de marquer plusieurs classes comme défaut.

---

## Utilisation avec les PVCs

Les PersistentVolumeClaims (PVCs) utilisent les StorageClasses de deux manières principales :

1.  **Pour le Provisionnement Dynamique :**
    *   Un PVC spécifie le nom d'une StorageClass dans son champ `spec.storageClassName`.
    *   Si aucun [PersistentVolume (PV)](./persistentvolumes.md) préexistant ne correspond aux critères du PVC (taille, modes d'accès, et cette `storageClassName`), le provisionneur associé à la StorageClass est invoqué pour créer un nouveau PV.
    *   Ce nouveau PV est alors automatiquement lié au PVC.

2.  **Pour la Liaison à des PVs Statiques :**
    *   Même si des PVs sont créés manuellement (statiquement), ils peuvent être associés à une `storageClassName`.
    *   Un PVC peut alors spécifier cette `storageClassName` pour s'assurer qu'il ne se lie qu'à des PVs de cette classe particulière, en plus de satisfaire les autres exigences comme la taille et les modes d'accès.

Si un PVC ne spécifie pas de `storageClassName`, il peut :
*   Utiliser la StorageClass par défaut du cluster pour le provisionnement dynamique.
*   Se lier à un PV statique qui n'a pas non plus de `storageClassName` spécifié.

Les StorageClasses offrent donc une grande flexibilité pour gérer comment et quel type de stockage est fourni aux applications dans Kubernetes.

---

> Next: [Sécurité dans Kubernetes](../security/security-overview.md)

> [cheat sheet](../useful.md)
