# Introduction à la Sécurité Kubernetes

La sécurité est un aspect fondamental de tout déploiement Kubernetes. Elle vise à protéger les applications, les données qu'elles traitent, ainsi que le cluster Kubernetes lui-même contre les menaces, les accès non autorisés et les vulnérabilités.

Kubernetes, de par sa nature distribuée et sa complexité, présente une surface d'attaque potentielle qui nécessite une approche de sécurité multi-couches. Cette approche est souvent décrite par le modèle des **"4 C" de la Sécurité Cloud Native** : Cloud (ou Infrastructure d'entreprise), Cluster, Conteneur, et Code.

Ce tutoriel se concentrera principalement sur les aspects de sécurité liés au **Cluster** et au **Conteneur** qui peuvent être configurés et gérés au sein de Kubernetes.

Au cours de cette section sur la sécurité, nous aborderons les sujets suivants :
*   **RBAC (Role-Based Access Control) :** Contrôler qui peut faire quoi sur les ressources du cluster.
*   **Security Contexts (Contextes de Sécurité) :** Définir les privilèges et les permissions des Pods et des conteneurs.
*   **Pod Security Admission (ou anciennement PodSecurityPolicies) :** Appliquer des standards de sécurité au niveau du cluster pour les Pods.
*   Nous rappellerons également l'importance de la gestion sécurisée des **Secrets**.

---

## Les 4 C de la Sécurité Cloud Native (Brève Mention)

Comprendre le modèle des "4 C" aide à situer les différentes responsabilités en matière de sécurité :

1.  **Cloud / Infrastructure (Cloud/Cluster Infrastructure) :**
    *   Concerne la sécurité de l'infrastructure sous-jacente sur laquelle Kubernetes est déployé. Cela inclut la sécurité physique des datacenters, la sécurité des réseaux virtuels, la sécurisation des machines virtuelles (VMs) ou des serveurs bare-metal, et la protection du système d'exploitation hôte.
    *   **Responsabilité :** Généralement celle du fournisseur de cloud (pour les services managés Kubernetes comme GKE, EKS, AKS) ou de l'équipe d'infrastructure (pour les clusters auto-hébergés).

2.  **Cluster :**
    *   Concerne la sécurité des composants du cluster Kubernetes lui-même et de sa configuration.
    *   Cela inclut la sécurisation de l'API Server (accès, authentification, autorisation), la protection d'etcd (la base de données du cluster), la sécurisation des kubelets sur chaque nœud, la configuration des politiques réseau, et la gestion des identités et des accès (RBAC).
    *   **Responsabilité :** Partagée entre l'administrateur du cluster et les utilisateurs qui déploient des configurations.

3.  **Conteneur (Container) :**
    *   Concerne la sécurité du moteur d'exécution des conteneurs (par exemple, Docker, containerd) et, de manière cruciale, la sécurité des images de conteneurs.
    *   Cela inclut l'utilisation d'images de base sécurisées et minimales, l'analyse des images pour détecter les vulnérabilités (CVEs), la configuration des conteneurs pour qu'ils s'exécutent avec le moins de privilèges possible (par exemple, en tant qu'utilisateur non-root), et la sécurisation des registres d'images.
    *   **Responsabilité :** Principalement celle des développeurs et des équipes DevOps qui construisent et déploient les images.

4.  **Code :**
    *   Concerne la sécurité de l'application elle-même qui s'exécute à l'intérieur des conteneurs.
    *   Cela inclut les pratiques de codage sécurisé, la gestion des dépendances (analyse des vulnérabilités dans les bibliothèques tierces), la protection contre les attaques courantes (par exemple, injection SQL, XSS), et la gestion des secrets applicatifs.
    *   **Responsabilité :** Principalement celle des développeurs d'applications.

Chaque couche s'appuie sur la sécurité de la couche inférieure. Une vulnérabilité dans une couche inférieure peut compromettre les couches supérieures.

---

## Concepts Clés de la Sécurité Kubernetes

Voici un aperçu des concepts de sécurité importants dans Kubernetes, qui seront détaillés dans les pages suivantes :

*   **Authentification & Autorisation :**
    *   **Authentification (Qui peut accéder ?) :** Détermine comment les utilisateurs, les administrateurs, et les composants du système (comme les Pods via les ServiceAccounts) prouvent leur identité à l'API Server de Kubernetes. Les méthodes incluent les certificats clients, les tokens (tokens de porteur, tokens de ServiceAccount, tokens OIDC), et les webhooks d'authentification.
    *   **Autorisation (Que peuvent-ils faire ?) :** Une fois authentifié, l'autorisation détermine si une identité a le droit d'effectuer une action spécifique (par exemple, créer un Pod, lister les Secrets) sur une ressource donnée. Kubernetes utilise principalement le **Role-Based Access Control (RBAC)**, qui sera notre principal point de focalisation. D'autres modes comme ABAC (Attribute-Based Access Control) ou les webhooks d'autorisation existent mais sont moins courants pour la gestion quotidienne.

*   **Pod Security Standards / Contextes de Sécurité (Security Contexts) :**
    *   Les **Contextes de Sécurité** (`securityContext`) permettent de définir des paramètres de sécurité au niveau du Pod et/ou du conteneur. Ces paramètres contrôlent les privilèges d'exécution, tels que :
        *   Exécuter en tant qu'utilisateur/groupe non-root (`runAsUser`, `runAsGroup`).
        *   Empêcher l'escalade de privilèges (`allowPrivilegeEscalation: false`).
        *   Utiliser un système de fichiers racine en lecture seule (`readOnlyRootFilesystem: true`).
        *   Définir les capacités Linux (`capabilities: { drop: ["ALL"], add: [...] }`).
    *   Les **Pod Security Standards** (qui remplacent les PodSecurityPolicies dépréciées) définissent différents niveaux de sécurité (`Privileged`, `Baseline`, `Restricted`) que les Pods doivent respecter pour être admis dans le cluster. Ils sont appliqués via le **Pod Security Admission controller**.

*   **Sécurité Réseau (Network Security) :**
    *   Comme nous l'avons vu dans la section réseau, les [NetworkPolicies](../objects/networkpolicies.md) sont cruciales pour contrôler le flux de trafic entre les Pods et avec d'autres points de terminaison réseau. Elles permettent de segmenter le réseau et d'appliquer le principe du moindre privilège au niveau du réseau.

*   **Gestion des Secrets (Secret Management) :**
    *   Bien que les [Secrets](../objects/configApp.md#secrets) aient été couverts dans la gestion de la configuration, leur aspect sécurité est primordial.
    *   Il est crucial d'activer le **chiffrement au repos (encryption at rest)** pour les Secrets dans etcd.
    *   L'accès aux Secrets doit être strictement contrôlé via RBAC.
    *   Pour des besoins avancés, des outils externes comme HashiCorp Vault ou Sealed Secrets peuvent être intégrés pour une gestion plus robuste des secrets.

*   **Sécurité des Images de Conteneurs (Image Security) :**
    *   Utiliser des images de base minimales et provenant de sources fiables.
    *   Analyser régulièrement les images pour détecter les vulnérabilités connues (CVEs) en utilisant des outils comme Trivy, Clair, ou des scanners intégrés aux registres d'images.
    *   Signer les images pour garantir leur intégrité et leur provenance.

*   **Contrôleurs d'Admission (Admission Controllers) :**
    *   Les contrôleurs d'admission sont des plugins de l'API Server qui interceptent les requêtes après leur authentification et leur autorisation, mais avant que l'objet ne soit persisté dans etcd.
    *   Ils peuvent valider (`ValidatingAdmissionWebhook`) ou modifier (`MutatingAdmissionWebhook`) les objets.
    *   Plusieurs contrôleurs d'admission sont importants pour la sécurité, notamment :
        *   `PodSecurity` (le nouveau Pod Security Admission controller).
        *   `PodSecurityPolicy` (déprécié à partir de Kubernetes 1.21 et supprimé en 1.25).
        *   `LimitRanger` (pour appliquer les limites de ressources).
        *   `ResourceQuota` (pour gérer les quotas de ressources par namespace).

---

## Principe du Moindre Privilège

Un fil conducteur à travers toutes les stratégies de sécurité Kubernetes est le **principe du moindre privilège**. Cela signifie que chaque composant, utilisateur, ou application ne doit avoir que les permissions strictement nécessaires pour accomplir sa tâche, et rien de plus.

*   **Pour les utilisateurs et les ServiceAccounts :** Définissez des rôles RBAC avec des permissions granulaires.
*   **Pour les Pods et les conteneurs :** Utilisez les `securityContext` pour limiter les capacités, exécuter en tant qu'utilisateur non-root, etc.
*   **Pour le réseau :** Utilisez les `NetworkPolicies` pour n'autoriser que les flux de communication nécessaires.

Appliquer ce principe réduit considérablement la surface d'attaque et l'impact potentiel d'une compromission.

---

> Next: [Role-Based Access Control (RBAC)](./rbac.md)

> [cheat sheet](../useful.md)
