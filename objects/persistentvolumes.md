# PersistentVolumes (PV) Kubernetes

Un PersistentVolume (PV) est une abstraction de stockage dans Kubernetes qui représente une "pièce" de stockage physique ou réseau dans le cluster. Il est provisionné par un administrateur ou dynamiquement par une `StorageClass` et a un cycle de vie indépendant de tout Pod individuel.

## Plongée dans les PersistentVolumes (PV)

### Définition

Un **PersistentVolume (PV)** est une ressource de l'API Kubernetes qui représente un volume de stockage existant, qu'il soit sur un système de stockage en réseau (comme NFS, iSCSI, Ceph), un service de stockage cloud (comme AWS EBS, Azure Disk, GCE Persistent Disk), ou même un répertoire local sur un nœud (pour le développement, via `hostPath`).

*   **Ressource de Cluster :** Un PV est une ressource du cluster, tout comme un nœud. Il n'appartient pas à un namespace spécifique.
*   **Gestion par l'Administrateur ou Dynamique :**
    *   **Statique :** Un administrateur de cluster peut créer manuellement plusieurs PVs qui représentent le stockage disponible. Ces PVs sont alors disponibles pour être consommés par les utilisateurs via des PersistentVolumeClaims (PVCs).
    *   **Dynamique :** Si aucun PV statique ne correspond à un PVC, le cluster peut essayer de provisionner dynamiquement un PV spécifiquement pour ce PVC, en utilisant une `StorageClass`.

### Cycle de Vie

Le cycle de vie d'un PV est distinct de celui d'un Pod. Les données stockées dans un PV persistent au-delà du redémarrage ou de la suppression des Pods qui l'utilisent. Un PV existe jusqu'à ce qu'il soit explicitement supprimé.

### Analogie

Pensez à un PV comme à un "disque dur physique" ou un "LUN (Logical Unit Number)" disponible dans le cluster. Bien qu'il puisse être virtuel ou défini par logiciel (comme un partage NFS ou un volume Ceph RBD), il représente une unité de stockage concrète que les applications peuvent utiliser.

---

## Champs Clés et Concepts d'un PV

Un manifeste de PersistentVolume contient plusieurs champs importants :

### `capacity`

*   Définit la quantité totale de stockage du PV.
*   Exemple : `capacity: { storage: 5Gi }` (5 Gibibytes), `capacity: { storage: 100Mi }` (100 Mebibytes).
*   Un PV doit avoir une capacité spécifiée. Les PVCs demanderont une certaine capacité, et Kubernetes essaiera de faire correspondre cette demande à un PV disponible.

### `volumeMode`

*   Spécifie si le volume sera utilisé comme un système de fichiers ou un périphérique bloc brut.
*   Valeurs possibles :
    *   **`Filesystem` (par défaut) :** Le volume est monté dans les Pods comme un répertoire. Un système de fichiers est supposé exister (ou sera créé par Kubernetes pour certains types de volumes) sur le périphérique avant le montage.
    *   **`Block` :** Le volume est présenté au Pod comme un périphérique bloc brut (par exemple, `/dev/xvda`). L'application dans le Pod est responsable de la gestion du périphérique bloc. Ce mode est utile pour les applications qui nécessitent un accès direct et de bas niveau au stockage, comme certaines bases de données.

### `accessModes`

*   Définit comment le volume peut être monté sur un nœud (et par conséquent, par les Pods sur ce nœud). C'est une liste de modes d'accès supportés par le PV.
*   Modes d'accès courants :
    *   **`ReadWriteOnce` (RWO) :** Le volume peut être monté en lecture-écriture par un **seul nœud** à la fois. Cela signifie que tous les Pods sur ce nœud spécifique peuvent lire et écrire sur le volume. C'est le mode le plus courant et il est supporté par la plupart des types de stockage.
    *   **`ReadOnlyMany` (ROX) :** Le volume peut être monté en lecture seule par **plusieurs nœuds** simultanément.
    *   **`ReadWriteMany` (RWX) :** Le volume peut être monté en lecture-écriture par **plusieurs nœuds** simultanément. Ce mode n'est supporté que par certains types de stockage partagé (par exemple, NFS, CephFS, GlusterFS, Azure Files). Les stockages bloc comme AWS EBS, GCE PD, Azure Disk ne supportent généralement pas RWX.
    *   **`ReadWriteOncePod` (RWOP) :** (Stable/GA depuis Kubernetes 1.29) Le volume peut être monté en lecture-écriture par **un seul Pod** à la fois dans tout le cluster. C'est une contrainte plus forte que RWO, utile pour certaines applications sensibles.
*   **Important :** Un PV peut spécifier plusieurs modes d'accès qu'il supporte. Un PVC demandera un mode d'accès spécifique, qui doit être l'un de ceux supportés par le PV pour qu'il y ait une liaison. Le fournisseur de stockage sous-jacent détermine quels modes d'accès sont réellement possibles.

### `storageClassName`

*   Un PV peut appartenir à une "classe" de stockage en spécifiant le nom d'une [StorageClass](./storageclasses.md) dans ce champ.
*   **Liaison avec les PVCs :**
    *   Un PV avec une `storageClassName` spécifique ne peut être lié qu'à des PVCs qui demandent explicitement cette même classe.
    *   Un PV sans `storageClassName` (c'est-à-dire `storageClassName: ""`) ne peut être lié qu'à des PVCs qui ne spécifient pas de `storageClassName`.
*   Ce champ est crucial pour le provisionnement dynamique, où la StorageClass définit comment créer de nouveaux PVs. Pour les PVs créés manuellement, il aide à les catégoriser.

### `persistentVolumeReclaimPolicy`

*   Définit ce qui arrive au PV (et à son volume de stockage sous-jacent) lorsque le PersistentVolumeClaim (PVC) auquel il était lié est supprimé.
*   Valeurs possibles :
    *   **`Retain` :** (Par défaut pour les PVs créés manuellement) Le PV reste dans le cluster après la suppression du PVC. Les données sur le volume sont préservées. L'administrateur du cluster doit manuellement nettoyer les données et supprimer le PV pour le rendre à nouveau disponible (ou le supprimer complètement). Le statut du PV passe à `Released`.
    *   **`Delete` :** Le volume de stockage sous-jacent (par exemple, le disque EBS sur AWS, le disque persistant sur GCE) est supprimé lorsque le PVC est supprimé. C'est généralement la politique par défaut pour les PVs provisionnés dynamiquement par une StorageClass.
    *   **`Recycle` (Obsolète) :** Effectue un nettoyage basique sur le volume (par exemple, `rm -rf /thevolume/*`) pour le rendre disponible pour un nouveau PVC. Cette politique est obsolète. Il est recommandé d'utiliser `Delete` et le provisionnement dynamique à la place. La plupart des provisionneurs de stockage ne supportent plus `Recycle`.

### `status` (Phase)

*   Indique l'état actuel du PV. Ce champ est géré par Kubernetes.
*   Phases possibles :
    *   **`Available` :** Le PV est libre et n'est pas encore lié à un PVC.
    *   **`Bound` :** Le PV est lié à un PVC.
    *   **`Released` :** Le PVC auquel le PV était lié a été supprimé, mais le PV n'a pas encore été récupéré par le cluster (par exemple, si la politique de récupération est `Retain`).
    *   **`Failed` :** Le PV a rencontré une erreur (par exemple, lors du provisionnement ou de la suppression).

---

## Exemple de PV Créé Manuellement (type NFS)

Voici un exemple de manifeste pour un PersistentVolume qui utilise un partage NFS existant.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-nfs-pv-example
  labels: # Optionnel: pour l'organisation et la sélection
    storage-type: network-attached-storage
    environment: production
spec:
  capacity:
    storage: 5Gi # Taille du volume
  volumeMode: Filesystem # Mode du volume (système de fichiers)
  accessModes:
    - ReadWriteMany # NFS supporte généralement RWX, permettant à plusieurs nœuds de lire/écrire
  persistentVolumeReclaimPolicy: Retain # Conserver le PV et les données après la suppression du PVC
  storageClassName: "slow-nfs" # Associe ce PV à la StorageClass nommée "slow-nfs"
                               # Si omis, ce PV ne peut être lié qu'à des PVCs sans storageClassName.
  mountOptions: # Options de montage spécifiques au type de volume
    - hard # Option de montage NFS "hard"
    - nfsvers=4.1 # Utiliser la version 4.1 de NFS
  nfs: # Section spécifique pour le type de volume NFS
    path: /exports/data/prod # Chemin du répertoire exporté sur le serveur NFS
    server: nfs-server.example.com # Adresse IP ou nom d'hôte du serveur NFS
    readOnly: false # Indique si le montage doit être en lecture seule (ici, non)
```

**Explication des Champs Spécifiques :**

*   **`spec.nfs` :** Cette section contient la configuration spécifique pour un volume de type NFS.
    *   `path` : Le chemin qui est exporté par le serveur NFS.
    *   `server` : L'adresse IP ou le nom d'hôte du serveur NFS.
    *   `readOnly` : Si `true`, le montage sera en lecture seule.
*   **`spec.mountOptions` :** Permet de spécifier des options de montage pour le volume, qui sont passées à l'outil de montage sous-jacent.
*   **Autres Types de Volumes :** Kubernetes supporte de nombreux autres types de volumes pour les PVs, chacun avec sa propre section de configuration. Quelques exemples :
    *   `awsElasticBlockStore` : Pour les volumes EBS d'Amazon Web Services.
    *   `azureDisk` : Pour les disques managés Microsoft Azure.
    *   `gcePersistentDisk` : Pour les disques persistants Google Compute Engine.
    *   `iscsi` : Pour les volumes iSCSI.
    *   `cephfs` : Pour les systèmes de fichiers CephFS.
    *   `hostPath` : (Principalement pour le développement et les tests mono-nœud) Monte un fichier ou un répertoire du système de fichiers du nœud hôte.

---

## Interaction avec les PVCs et les StorageClasses

*   **PVs comme "Offre" :** Les PersistentVolumes constituent l' "offre" de stockage disponible dans le cluster. Ils décrivent des ressources de stockage concrètes avec leurs capacités et caractéristiques.
*   **PVCs comme "Demande" :** Les [PersistentVolumeClaims (PVCs)](./persistentvolumeclaims.md) sont la "demande" de stockage faite par les utilisateurs ou les applications. Un PVC demande une certaine quantité de stockage, des modes d'accès spécifiques, et peut optionnellement demander une `StorageClass`.
*   **Liaison (Binding) :** Kubernetes tente de faire correspondre un PVC à un PV approprié. Pour qu'une liaison se produise :
    *   Le PV doit avoir une capacité suffisante pour satisfaire la demande du PVC.
    *   Les modes d'accès demandés par le PVC doivent être un sous-ensemble des modes d'accès offerts par le PV.
    *   La `storageClassName` doit correspondre (si spécifiée dans le PVC et/ou le PV). Si le PVC spécifie une `storageClassName`, il ne peut être lié qu'à un PV de cette classe. Si le PVC ne spécifie pas de `storageClassName`, il ne peut être lié qu'à un PV qui n'a pas non plus de `storageClassName`.
*   **Provisionnement Statique vs. Dynamique :**
    *   **Statique :** Un administrateur crée manuellement des PVs (comme l'exemple NFS ci-dessus). Ces PVs sont ensuite disponibles pour être réclamés par des PVCs.
    *   **Dynamique :** Si un PVC demande une `StorageClass` et qu'aucun PV statique ne correspond, la `StorageClass` peut automatiquement provisionner un nouveau PV pour répondre à la demande du PVC. Cette page se concentre sur l'objet PV lui-même, qui peut être le résultat de l'un ou l'autre de ces processus. Le provisionnement dynamique via les [StorageClasses](./storageclasses.md) sera détaillé dans une section ultérieure.

---

> Next: [PersistentVolumeClaims (PVC)](./persistentvolumeclaims.md)

> [cheat sheet](../useful.md)
