# Contextes de Sécurité (Security Contexts) Kubernetes

Les contextes de sécurité sont des champs dans les spécifications des Pods et des conteneurs qui permettent de définir des paramètres de privilèges et de contrôle d'accès. Ils sont essentiels pour renforcer la sécurité des charges de travail en limitant ce que les Pods et leurs conteneurs peuvent faire.

## Introduction aux Contextes de Sécurité

### Que sont-ils ?

Un **Security Context** (contexte de sécurité) définit les paramètres de privilège et de contrôle d'accès pour un Pod ou un Conteneur. Cela inclut, par exemple, l'identifiant utilisateur (UID) et de groupe (GID) avec lequel le processus s'exécute, les capacités Linux à accorder ou à retirer, et si le conteneur peut s'exécuter en mode privilégié.

### Objectif

L'objectif principal des contextes de sécurité est de renforcer la sécurité en appliquant le **principe du moindre privilège**. En configurant des contextes de sécurité restrictifs, vous pouvez réduire la surface d'attaque de vos applications et limiter l'impact potentiel d'une compromission d'un conteneur.

Ils permettent de contrôler finement les aspects de sécurité qui ne sont pas gérés par d'autres mécanismes comme le RBAC (qui contrôle l'accès à l'API Kubernetes) ou les NetworkPolicies (qui contrôlent le trafic réseau).

---

## Niveaux des Contextes de Sécurité

Les contextes de sécurité peuvent être spécifiés à deux niveaux :

1.  **Au Niveau du Pod (`spec.securityContext`) :**
    *   Ces paramètres s'appliquent à **tous les conteneurs** au sein du Pod.
    *   Ils définissent des valeurs par défaut pour les conteneurs du Pod.
    *   Exemples : `runAsUser`, `runAsGroup`, `fsGroup`, `runAsNonRoot`.

2.  **Au Niveau du Conteneur (`spec.containers[].securityContext`) :**
    *   Ces paramètres sont spécifiques à un **conteneur individuel** au sein d'un Pod.
    *   Si un paramètre est défini à la fois au niveau du Pod et au niveau du conteneur, la valeur spécifiée au niveau du **conteneur prévaut** pour ce conteneur.
    *   Exemples : `runAsUser`, `capabilities`, `privileged`, `readOnlyRootFilesystem`.

Cette hiérarchie permet de définir des politiques de sécurité de base pour un Pod entier tout en permettant des ajustements fins pour des conteneurs spécifiques si nécessaire.

---

## Champs Clés du Contexte de Sécurité (Niveau Pod)

Ces champs sont définis dans `spec.securityContext` d'un Pod.

*   **`runAsUser: <UID>` / `runAsGroup: <GID>` :**
    *   Spécifie l'UID (User ID) et/ou le GID (Group ID) avec lequel le processus principal (entrypoint) de **tous les conteneurs** du Pod doit s'exécuter.
    *   Si non spécifié, l'UID/GID utilisé est celui défini dans l'image du conteneur (souvent root, UID 0).
    *   Il est fortement recommandé de définir `runAsUser` avec un UID non nul pour éviter de s'exécuter en tant que root.
*   **`runAsNonRoot: true` :**
    *   Si mis à `true`, Kubelet validera avant de lancer un conteneur que l'image n'est pas configurée pour s'exécuter en tant que root (UID 0). Si l'image tente de s'exécuter en tant que root, le Pod échouera au démarrage.
    *   Ce champ est une mesure de sécurité importante pour s'assurer que les conteneurs ne s'exécutent pas avec des privilèges root.
*   **`fsGroup: <GID>` :**
    *   Spécifie un GID supplémentaire (supplemental group ID) qui s'applique à tous les conteneurs du Pod.
    *   Ce GID est propriétaire de tous les fichiers créés dans les volumes montés par le Pod (comme `emptyDir` ou les volumes persistants).
    *   Utile pour permettre à plusieurs conteneurs (s'exécutant potentiellement avec des UIDs différents mais partageant ce GID) de lire et écrire sur des volumes partagés.
*   **`fsGroupChangePolicy: "OnRootMismatch" | "Always"` :**
    *   Définit quand et comment la propriété et les permissions du `fsGroup` sont appliquées aux volumes.
    *   `Always` : Kubelet change toujours récursivement la propriété et les permissions du volume pour correspondre au `fsGroup` lors du montage. Cela peut être lent pour les grands volumes.
    *   `OnRootMismatch` (par défaut) : Kubelet ne change la propriété et les permissions que si le répertoire racine du volume n'a pas déjà la bonne propriété et les bonnes permissions. Cela peut accélérer le montage des volumes.
*   **Modules de Sécurité Linux (Avancé) :**
    *   **`seLinuxOptions` :** Configure le contexte SELinux pour les conteneurs du Pod. Nécessite que SELinux soit activé sur les nœuds.
    *   **`seccompProfile` :** Permet de restreindre les appels système (syscalls) qu'un conteneur peut effectuer.
        *   `type: RuntimeDefault` : Applique le profil seccomp par défaut fourni par le runtime de conteneur (par exemple, Docker, containerd). C'est une bonne pratique de sécurité.
        *   `type: Unconfined` : Aucun profil seccomp n'est appliqué (potentiellement dangereux).
        *   `type: Localhost, localhostProfile: <nom-profil.json>` : Utilise un profil seccomp personnalisé stocké sur le nœud.
    *   **`appArmorProfile` :** Configure les profils AppArmor pour les conteneurs du Pod. Nécessite qu'AppArmor soit activé et configuré sur les nœuds.

---

## Champs Clés du Contexte de Sécurité (Niveau Conteneur)

Ces champs sont définis dans `spec.containers[].securityContext` d'un conteneur spécifique. Ils peuvent surcharger les paramètres définis au niveau du Pod pour ce conteneur.

*   **`runAsUser: <UID>` / `runAsGroup: <GID>` / `runAsNonRoot: true` :**
    *   Mêmes significations qu'au niveau du Pod, mais s'appliquent uniquement à ce conteneur spécifique. Ils surchargent les valeurs du `securityContext` du Pod.
*   **`privileged: true | false` :**
    *   Si `true`, le conteneur s'exécute en mode privilégié. Cela désactive la plupart des mécanismes de sécurité des conteneurs et donne au conteneur un accès presque complet à tous les périphériques et capacités du nœud hôte.
    *   **À utiliser avec une extrême prudence et uniquement si absolument nécessaire.** C'est une configuration très dangereuse qui expose le nœud hôte à des risques importants. Généralement non recommandé.
*   **`allowPrivilegeEscalation: true | false` :**
    *   Contrôle si un processus à l'intérieur du conteneur peut obtenir plus de privilèges que son processus parent.
    *   Il est fortement recommandé de le mettre à `false` pour empêcher les processus d'escalader leurs privilèges (par exemple, via des binaires SUID ou SGID). `false` n'empêche pas le mode `privileged: true`.
*   **`readOnlyRootFilesystem: true | false` :**
    *   Si `true`, le système de fichiers racine du conteneur est monté en lecture seule.
    *   C'est une excellente pratique de sécurité car cela empêche l'application (ou un attaquant) de modifier les fichiers du système d'exploitation du conteneur, y compris les binaires et les configurations.
    *   L'application doit être conçue pour écrire ses données temporaires ou persistantes uniquement sur des volumes désignés (par exemple, `emptyDir` ou `persistentVolumeClaim`) qui sont montés en lecture-écriture.
*   **`capabilities` :**
    *   Permet de gérer finement les capacités Linux. Les capacités Linux divisent les privilèges root traditionnels en unités plus petites et distinctes.
    *   **`add: [<capability_name>]` :** Ajoute des capacités spécifiques.
    *   **`drop: [<capability_name>]` :** Retire des capacités spécifiques.
    *   Il est recommandé de retirer toutes les capacités (`drop: ["ALL"]`) puis d'ajouter uniquement celles qui sont strictement nécessaires.
    *   Exemple :
        ```yaml
        capabilities:
          drop:
          - ALL # Retire toutes les capacités par défaut
          add:
          - NET_BIND_SERVICE # Permet au conteneur de lier des ports inférieurs à 1024
                             # sans avoir besoin d'être root (si runAsUser est non-root).
          # - SYS_TIME # Permet de modifier l'heure système (généralement non nécessaire)
          # - MKNOD # Permet de créer des fichiers de périphériques (généralement non nécessaire)
        ```
*   **`seLinuxOptions` / `seccompProfile` / `appArmorProfile` :**
    *   Mêmes significations qu'au niveau du Pod, mais permettent une configuration plus fine pour chaque conteneur individuel.

---

## Exemple de Pod avec Contextes de Sécurité

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod-example
spec:
  securityContext: # Contexte de Sécurité au niveau du Pod
    runAsUser: 1000 # Tous les conteneurs s'exécuteront en tant qu'UID 1000 par défaut
    runAsGroup: 3000 # Tous les conteneurs s'exécuteront en tant que GID 3000 par défaut
    fsGroup: 2000    # Les volumes seront accessibles par le groupe GID 2000
    runAsNonRoot: true # Valide que les conteneurs ne s'exécutent pas en tant que root
    seccompProfile: # Applique le profil seccomp par défaut du runtime
      type: RuntimeDefault
  containers:
  - name: my-secure-container
    image: my-app-image:latest # Remplacez par votre image
    # Ports, variables d'environnement, etc.
    securityContext: # Contexte de Sécurité au niveau du Conteneur
      # runAsUser: 1001 # Pourrait surcharger runAsUser du Pod pour CE conteneur spécifique
      # runAsGroup: 3001 # Pourrait surcharger runAsGroup du Pod
      allowPrivilegeEscalation: false # Empêche l'escalade de privilèges
      readOnlyRootFilesystem: true    # Le système de fichiers racine est en lecture seule
      capabilities:
        drop:
        - ALL # Retire toutes les capacités
        # add: # Ajoutez uniquement les capacités strictement nécessaires
        # - NET_BIND_SERVICE
    volumeMounts: # Exemple de montage de volume pour les données temporaires ou persistantes
    - name: app-data
      mountPath: /app/data # L'application peut écrire ici si readOnlyRootFilesystem est true
  volumes:
  - name: app-data
    emptyDir: {} # Ou un persistentVolumeClaim
```

---

## Interaction avec les Pod Security Standards (PSS) / Pod Security Admission (PSA)

Kubernetes a évolué dans la manière dont les politiques de sécurité des Pods sont appliquées à l'échelle du cluster :

*   **PodSecurityPolicies (PSP) :** (Déprécié à partir de Kubernetes 1.21 et supprimé en 1.25) Était un contrôleur d'admission qui validait les spécifications des Pods par rapport à des politiques définies.
*   **Pod Security Admission (PSA) :** C'est le remplaçant intégré de PSP. C'est un contrôleur d'admission qui applique les **Pod Security Standards** (PSS).
    *   Les **Pod Security Standards** sont des ensembles de politiques prédéfinies :
        *   `Privileged` : Non sécurisé, essentiellement sans restrictions.
        *   `Baseline` : Minimalement restrictif, bloque les escalades de privilèges connues.
        *   `Restricted` : Très restrictif, suit les meilleures pratiques de durcissement actuelles.
    *   PSA peut être configuré au niveau du namespace pour `enforce` (appliquer), `audit` (journaliser les violations), ou `warn` (avertir des violations) un standard spécifique.
    *   Les **Contextes de Sécurité** sont les mécanismes que vous utilisez dans la définition de vos Pods et conteneurs pour vous assurer qu'ils respectent les exigences du standard PSS appliqué à leur namespace. Par exemple, le standard "Restricted" exige généralement :
        *   `runAsNonRoot: true`
        *   `seccompProfile: { type: RuntimeDefault }` ou un profil plus strict.
        *   `capabilities: { drop: ["ALL"] }`
        *   `allowPrivilegeEscalation: false`

Configurer correctement les `securityContext` de vos Pods et conteneurs est donc essentiel pour satisfaire aux exigences des Pod Security Standards et pour sécuriser vos charges de travail.

---

## Bonnes Pratiques

*   **Exécuter en tant qu'Utilisateur Non-Root :**
    *   Utilisez `securityContext.runAsNonRoot: true` au niveau du Pod.
    *   Spécifiez `securityContext.runAsUser: <UID_non_nul>` (par exemple, `1000`) et `securityContext.runAsGroup: <GID_non_nul>` (par exemple, `1000`). Assurez-vous que votre image de conteneur est construite pour s'exécuter avec cet UID/GID ou qu'elle peut le faire.
*   **Système de Fichiers Racine en Lecture Seule :**
    *   Utilisez `securityContext.readOnlyRootFilesystem: true` pour chaque conteneur.
    *   Montez des volumes (`emptyDir` ou `persistentVolumeClaim`) pour les chemins où l'application a besoin d'écrire.
*   **Empêcher l'Escalade de Privilèges :**
    *   Utilisez `securityContext.allowPrivilegeEscalation: false` pour chaque conteneur.
*   **Gérer les Capacités Linux :**
    *   Retirez toutes les capacités par défaut : `securityContext.capabilities: { drop: ["ALL"] }`.
    *   Ajoutez ensuite uniquement les capacités minimales requises : `add: ["CAP_NEEDED_1", "CAP_NEEDED_2"]`.
*   **Utiliser les Profils Seccomp :**
    *   Appliquez le profil seccomp par défaut du runtime : `securityContext.seccompProfile: { type: RuntimeDefault }`.
    *   Pour une sécurité renforcée, envisagez des profils seccomp personnalisés qui ne listent que les appels système autorisés.
*   **Éviter le Mode Privilégié :** N'utilisez `securityContext.privileged: true` sous aucun prétexte, sauf si c'est une exigence absolue et que vous comprenez pleinement les risques (par exemple, pour certains pilotes de périphériques de bas niveau).
*   **Appliquer au Niveau du Pod et Surcharger si Nécessaire :** Définissez des contextes de sécurité restrictifs au niveau du Pod (`spec.securityContext`) comme base, et ne les assouplissez au niveau du conteneur (`spec.containers[].securityContext`) que pour les conteneurs qui en ont un besoin justifié.

---

> Next: [Cheat Sheet](../useful.md)

> [cheat sheet](../useful.md)
