# PersistentVolumeClaims (PVC) Kubernetes

Un PersistentVolumeClaim (PVC) est une demande de stockage faite par un utilisateur ou une application au sein d'un cluster Kubernetes. Il permet aux Pods d'accéder à des ressources de stockage persistantes (définies par des PersistentVolumes) sans avoir à connaître les détails de l'infrastructure de stockage sous-jacente.

## Plongée dans les PersistentVolumeClaims (PVC)

### Définition

Un **PersistentVolumeClaim (PVC)** est un objet de l'API Kubernetes qui représente une requête pour un `PersistentVolume (PV)`. Les développeurs d'applications créent des PVCs pour "réclamer" une certaine quantité de stockage avec des caractéristiques spécifiques (comme la taille, le mode d'accès, et optionnellement une classe de stockage).

*   **Demande de Stockage :** Un PVC est une demande de ressources de stockage (taille, modes d'accès).
*   **Utilisation par les Pods :** Les Pods utilisent les PVCs comme des volumes. Le PVC abstrait les détails du PV sous-jacent.
*   **Lien avec les PVs :** Kubernetes tente de satisfaire un PVC en le liant à un PV disponible qui correspond aux critères de la demande. Si le provisionnement dynamique est configuré (via une `StorageClass`), un nouveau PV peut être créé automatiquement pour satisfaire le PVC.

### Analogie

Si un [PersistentVolume (PV)](./persistentvolumes.md) est comme un "disque dur physique" disponible dans le datacenter (ou un volume de stockage cloud), alors un **PersistentVolumeClaim (PVC)** est comme un "bon de commande" ou une "requête" pour obtenir un disque dur avec des spécifications précises (par exemple, "j'ai besoin d'un disque de 10Go avec accès en lecture-écriture").

### Portée au Namespace (Namespace-scoped)

Contrairement aux PVs qui sont des ressources globales au cluster, les **PVCs doivent exister dans le même namespace que le Pod qui les utilise.** Un Pod dans le namespace `ns-app` ne peut utiliser qu'un PVC qui existe également dans `ns-app`.

---

## Champs Clés et Concepts d'un PVC

Un manifeste de PersistentVolumeClaim contient plusieurs champs importants :

### `accessModes`

*   Définit les modes d'accès souhaités pour le volume. C'est une liste, et le PVC ne sera lié qu'à un PV qui supporte au moins l'un des modes demandés.
*   Les modes sont les mêmes que pour les PVs :
    *   **`ReadWriteOnce` (RWO) :** Le volume peut être monté en lecture-écriture par un seul nœud.
    *   **`ReadOnlyMany` (ROX) :** Le volume peut être monté en lecture seule par plusieurs nœuds.
    *   **`ReadWriteMany` (RWX) :** Le volume peut être monté en lecture-écriture par plusieurs nœuds.
    *   **`ReadWriteOncePod` (RWOP) :** Le volume peut être monté en lecture-écriture par un seul Pod à la fois dans tout le cluster.
*   Un PVC demande un ou plusieurs modes d'accès. Par exemple, `accessModes: ["ReadWriteOnce", "ReadOnlyMany"]` signifie que le PVC peut être satisfait par un PV offrant soit RWO, soit ROX (ou les deux). Cependant, une fois lié, un seul mode d'accès est choisi pour le montage (généralement le premier mode compatible trouvé).

### `resources.requests.storage`

*   Spécifie la quantité minimale de stockage requise.
*   Exemple : `resources: { requests: { storage: 3Gi } }` (demande 3 Gibibytes).
*   Le PVC ne sera lié qu'à un PV qui a au moins cette capacité. Kubernetes ne permet pas de lier un PVC de 10Gi à un PV de 5Gi.

### `storageClassName`

*   Le nom de la [StorageClass](./storageclasses.md) demandée pour ce PVC.
*   **Comportement :**
    *   **Si `storageClassName` est spécifié :**
        *   Le PVC ne peut être lié qu'à un PV ayant la même `storageClassName`.
        *   Si aucun PV statique correspondant n'est trouvé, la `StorageClass` nommée ici est utilisée pour tenter de provisionner dynamiquement un nouveau PV. Si la `StorageClass` n'existe pas ou ne peut pas provisionner, le PVC restera en attente (`Pending`).
    *   **Si `storageClassName` est omis ou mis à `""` (chaîne vide) :**
        *   **Provisionnement dynamique :** Si une `StorageClass` par défaut est configurée dans le cluster, Kubernetes peut l'utiliser pour provisionner dynamiquement un PV pour ce PVC. Le comportement exact (si une StorageClass par défaut est utilisée ou non lorsque `storageClassName` est omis) peut dépendre de la configuration du cluster (par exemple, des `Admission Controllers`).
        *   **Liaison statique :** Le PVC ne peut être lié qu'à des PVs qui n'ont pas non plus de `storageClassName` (c'est-à-dire `storageClassName: ""`).
*   Pour désactiver explicitement le provisionnement dynamique pour un PVC, vous pouvez le lier à une `StorageClass` qui n'a pas de provisionneur, ou mettre `storageClassName: ""` (et vous assurer qu'il n'y a pas de StorageClass par défaut ou qu'elle ne s'applique pas).

### `volumeName` (Pour la Liaison Statique)

*   Un PVC peut demander à être lié à un PV spécifique en fournissant le nom de ce PV dans le champ `spec.volumeName`.
*   **Utilisation :** Cela contourne le processus de correspondance normal (basé sur la taille, les modes d'accès, et la `StorageClass`) et le provisionnement dynamique.
*   **Conditions :** Le PV nommé doit exister, être `Available`, et satisfaire aux exigences du PVC (taille, modes d'accès).
*   **Précaution :** L'utilisation de `volumeName` crée un couplage fort entre le PVC et un PV spécifique, ce qui rend la configuration moins portable et flexible. À utiliser avec discernement, principalement lorsque vous avez besoin de réutiliser un PV existant spécifique.

### `selector` (Pour une Liaison Statique plus Affinée)

*   Un PVC peut utiliser un `selector` pour affiner l'ensemble des PVs statiques auxquels il peut être lié. Le `selector` est une requête de labels.
*   **Fonctionnement :** Seuls les PVs qui ont les labels correspondants au `selector` (et qui satisfont aussi aux autres exigences comme la taille et les modes d'accès) seront considérés pour la liaison.
*   **Exemple :**
    ```yaml
    selector:
      matchLabels:
        environment: production
        disktype: ssd
    ```
    Ce PVC ne se lierait qu'à des PVs ayant les labels `environment=production` ET `disktype=ssd`.
*   C'est une alternative à `volumeName` pour une liaison statique plus flexible. Le `selector` n'est pas utilisé si le provisionnement dynamique est déclenché.

### `status` (Phase)

*   Indique l'état actuel du PVC. Ce champ est géré par Kubernetes.
*   Phases possibles :
    *   **`Pending` :** Le PVC a été créé mais n'est pas encore lié à un PV. Cela peut se produire si aucun PV correspondant n'est trouvé, si un PV correspondant est trouvé mais pas encore disponible, ou si le provisionnement dynamique est en cours.
    *   **`Bound` :** Le PVC est lié avec succès à un PV. Le PVC peut maintenant être utilisé par un Pod.
    *   **`Lost` :** Le PVC était lié à un PV, mais la liaison a été perdue (par exemple, si le PV a été supprimé manuellement ou est devenu inaccessible). Les données pourraient être perdues.

---

## Exemple de PVC

Voici un exemple de manifeste pour un PersistentVolumeClaim :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-pvc
  namespace: my-app-namespace # Les PVCs sont à portée de namespace
spec:
  accessModes:
    - ReadWriteOnce # Demande un accès en lecture-écriture par un seul nœud
  resources:
    requests:
      storage: 2Gi # Demande 2 Gibibytes de stockage
  # ----- Champs Optionnels -----
  # storageClassName: "fast-ssd" 
  # Si vous voulez utiliser une StorageClass spécifique pour le provisionnement dynamique
  # ou pour lier à des PVs de cette classe.

  # volumeName: "my-specific-pv"
  # Pour lier ce PVC à un PV nommé "my-specific-pv".
  # Utiliser cela désactive le provisionnement dynamique pour ce PVC.

  # selector:
  #   matchLabels:
  #     environment: production
  #     backup-policy: daily
  # Pour filtrer les PVs statiques basés sur leurs labels.
```

---

## Comment les Pods Utilisent les PVCs

Les Pods accèdent au stockage persistant en montant un volume qui référence un PVC.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod-with-storage
  namespace: my-app-namespace # Doit être le même namespace que le PVC
spec:
  containers:
    - name: my-container
      image: nginx
      ports:
        - containerPort: 80
      volumeMounts: # Monte le volume dans le système de fichiers du conteneur
        - name: my-storage-volume # Nom du volume défini ci-dessous
          mountPath: /usr/share/nginx/html # Chemin de montage à l'intérieur du conteneur
  volumes: # Définit les volumes disponibles pour ce Pod
    - name: my-storage-volume # Nom arbitraire pour ce volume dans le contexte du Pod
      persistentVolumeClaim:
        claimName: my-app-pvc # Nom du PVC à utiliser (doit exister dans le même namespace)
```

**Explication :**
1.  La section `spec.volumes` du Pod définit un volume nommé `my-storage-volume`.
2.  Ce volume est de type `persistentVolumeClaim` et pointe vers le PVC nommé `my-app-pvc`.
3.  La section `spec.containers[].volumeMounts` monte ensuite ce volume (`my-storage-volume`) à un chemin spécifique (`/usr/share/nginx/html`) à l'intérieur du conteneur.
4.  L'application s'exécutant dans `my-container` peut alors lire et écrire des données dans `/usr/share/nginx/html`, et ces données seront stockées sur le PersistentVolume lié à `my-app-pvc`.

---

## Processus de Liaison (Binding)

Lorsqu'un PVC est créé, Kubernetes lance un processus de contrôle pour trouver un PV approprié et le lier au PVC. Ce processus prend en compte plusieurs facteurs :

1.  **`volumeName` :** Si `spec.volumeName` est spécifié dans le PVC, Kubernetes tente uniquement de lier ce PVC au PV portant ce nom. Si le PV n'existe pas ou ne correspond pas aux autres critères (taille, modes d'accès), le PVC reste `Pending`.
2.  **`storageClassName` :**
    *   Si une `storageClassName` est spécifiée dans le PVC :
        *   **Liaison Statique :** Kubernetes recherche d'abord un PV `Available` ayant la même `storageClassName` et qui satisfait aux demandes de taille et de modes d'accès du PVC (et au `selector` du PVC, s'il est défini).
        *   **Provisionnement Dynamique :** Si aucun PV statique approprié n'est trouvé et que la `StorageClass` spécifiée a un provisionneur configuré, le provisionneur est appelé pour créer un nouveau PV. Ce nouveau PV sera alors lié au PVC.
    *   Si aucune `storageClassName` n'est spécifiée dans le PVC (ou si elle est `""`) :
        *   **Liaison Statique :** Kubernetes recherche un PV `Available` qui n'a pas non plus de `storageClassName` et qui satisfait aux demandes de taille et de modes d'accès (et au `selector` du PVC, s'il est défini).
        *   **Provisionnement Dynamique (via StorageClass par défaut) :** Si une `StorageClass` est marquée comme "default" dans le cluster, elle peut être utilisée pour le provisionnement dynamique même si le PVC ne la spécifie pas. Si aucune StorageClass par défaut n'existe ou si elle n'est pas configurée pour s'appliquer dans ce cas, et qu'aucun PV statique ne correspond, le PVC restera `Pending`.
3.  **`accessModes` :** Le PV doit supporter au moins l'un des modes d'accès demandés par le PVC.
4.  **`resources.requests.storage` :** La capacité du PV doit être supérieure ou égale à la capacité demandée par le PVC.
5.  **`selector` :** Si un `selector` est défini dans le PVC, le PV doit avoir des labels qui correspondent à ce sélecteur (ceci s'applique principalement à la liaison statique).

Une fois qu'un PV approprié est trouvé (ou provisionné), il est lié au PVC, et le statut des deux passe à `Bound`. Un PV ne peut être lié qu'à un seul PVC à la fois (relation un-à-un).

---

> Next: [StorageClasses](./storageclasses.md)

> [cheat sheet](../useful.md)
