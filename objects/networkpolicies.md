# NetworkPolicies Kubernetes (Politiques Réseau)

Les NetworkPolicies sont des ressources Kubernetes qui contrôlent le flux de trafic réseau au niveau de l'adresse IP ou du port (couche OSI 3 ou 4) entre les Pods d'un cluster. Elles constituent un outil essentiel pour la sécurisation et la segmentation du réseau au sein de Kubernetes.

## Introduction aux NetworkPolicies

### Que sont-elles ?

Une **NetworkPolicy** est un objet Kubernetes qui définit comment des groupes de Pods sont autorisés à communiquer entre eux et avec d'autres points de terminaison réseau. Les NetworkPolicies utilisent des labels pour sélectionner des Pods et définissent des règles qui spécifient quel trafic est autorisé.

### Pourquoi sont-elles nécessaires ?

*   **Sécurité :** Par défaut, toute communication est autorisée entre tous les Pods au sein d'un cluster Kubernetes. Les NetworkPolicies permettent de restreindre cette communication pour renforcer la sécurité.
*   **Segmentation du Réseau :** Elles permettent d'isoler des applications ou des environnements au sein d'un même namespace ou entre différents namespaces. Par exemple, empêcher une application frontend de communiquer directement avec une base de données de production, sauf via un service backend autorisé.
*   **Posture "Default Deny" :** Vous pouvez implémenter une politique de "refus par défaut" où tout trafic est bloqué, puis n'autoriser explicitement que les communications nécessaires.
*   **Conformité :** Dans de nombreux contextes réglementés, la segmentation du réseau et le contrôle du trafic sont des exigences.

### Implémentation par un Plugin Réseau

Les NetworkPolicies sont implémentées par le **plugin réseau CNI (Container Network Interface)** configuré dans votre cluster. Tous les plugins CNI ne supportent pas les NetworkPolicies.

*   **Exemples de plugins supportant les NetworkPolicies :** Calico, Cilium, Weave Net, Kube-router, Antrea.
*   **Vérification :** Assurez-vous que votre cluster Kubernetes utilise un plugin réseau qui les prend en charge avant de tenter de les utiliser. Si le plugin ne les supporte pas, la création d'une ressource NetworkPolicy n'aura aucun effet.

## Prérequis

*   Un cluster Kubernetes fonctionnel.
*   Un plugin réseau CNI qui supporte les NetworkPolicies doit être installé et configuré dans le cluster.

## Comment Fonctionnent les NetworkPolicies

*   **Portée au Namespace :** Les NetworkPolicies sont des objets à portée de namespace. Une politique définie dans un namespace ne s'applique qu'aux Pods de ce namespace.
*   **Comportement par Défaut (Aucune Politique) :** Si aucun NetworkPolicy dans un namespace ne sélectionne un Pod particulier, alors tout trafic entrant (ingress) et sortant (egress) est autorisé pour ce Pod. C'est la politique "allow-all" par défaut de Kubernetes.
*   **Isolation des Pods :** Dès qu'au moins un NetworkPolicy sélectionne un Pod (pour l'ingress ou l'egress), ce Pod devient "isolé". Cela signifie que :
    *   Pour le trafic **ingress**, tout trafic entrant est refusé, sauf s'il est explicitement autorisé par une règle `ingress` dans une politique qui sélectionne ce Pod.
    *   Pour le trafic **egress**, tout trafic sortant est refusé, sauf s'il est explicitement autorisé par une règle `egress` dans une politique qui sélectionne ce Pod.
*   **Politiques Additives :** Les règles des NetworkPolicies sont additives. Si plusieurs politiques s'appliquent à un même Pod, l'union de leurs règles `ingress` et `egress` est appliquée. Le trafic est autorisé s'il correspond à *au moins une* des règles applicables.

## Sélection des Pods (`podSelector`)

Chaque NetworkPolicy doit spécifier les Pods auxquels elle s'applique à l'aide d'un `podSelector`. Ce sélecteur utilise les labels des Pods.

*   **`spec.podSelector` :** Définit le groupe de Pods dans le namespace actuel auquel cette politique s'applique.
    *   Exemple : Un `podSelector` avec `matchLabels: { app: backend }` appliquera la politique à tous les Pods ayant le label `app=backend` dans le namespace de la politique.
*   **`podSelector: {}` (Sélecteur Vide) :** Un sélecteur de pod vide (`podSelector: {}`) sélectionne **tous** les Pods dans le namespace. C'est souvent utilisé pour créer des politiques de "default deny".

## Types de Politiques (`policyTypes`)

Le champ `spec.policyTypes` (optionnel) d'un NetworkPolicy indique les types de trafic auxquels la politique s'applique :

*   **`Ingress` :** La politique contient des règles pour le trafic entrant vers les Pods sélectionnés par `podSelector`.
*   **`Egress` :** La politique contient des règles pour le trafic sortant depuis les Pods sélectionnés par `podSelector`.

**Comportement de `policyTypes` :**
*   Si `policyTypes` est omis et que la politique contient des règles `ingress`, elle est traitée comme si `policyTypes: ["Ingress"]` était spécifié.
*   Si `policyTypes` est omis et que la politique contient des règles `egress`, elle est traitée comme si `policyTypes: ["Egress"]` était spécifié.
*   Si `policyTypes` est omis et que la politique contient des règles `ingress` ET `egress`, elle est traitée comme si `policyTypes: ["Ingress", "Egress"]` était spécifié.
*   **Important pour le "Default Deny" :** Si `policyTypes` est spécifié (par exemple, `policyTypes: ["Ingress"]`) mais que la section correspondante (`ingress` ou `egress`) est absente ou vide (pas de règles), cela signifie que ce type de trafic est refusé pour les Pods sélectionnés. Par exemple, si `policyTypes: ["Ingress"]` est défini mais qu'il n'y a pas de section `ingress` (ou une section `ingress: []` vide), tout le trafic entrant est refusé pour les Pods sélectionnés.

## Règles de Politique (`ingress` et `egress`)

Les sections `ingress` et `egress` définissent les règles qui autorisent le trafic. Chaque section est une liste de règles. Si une section (par exemple, `ingress`) est omise ou vide, aucun trafic de ce type n'est autorisé pour les Pods isolés par cette politique (sauf si explicitement autorisé par une autre politique).

Chaque règle de trafic (`ingress` ou `egress`) peut spécifier :

*   **`from` (pour `ingress`) ou `to` (pour `egress`) :** Définit les sources (pour ingress) ou les destinations (pour egress) autorisées. C'est une liste d'éléments, et le trafic est autorisé s'il correspond à *au moins un* de ces éléments. On peut spécifier :
    *   **`podSelector` :** Sélectionne d'autres Pods (généralement dans le même namespace que la NetworkPolicy).
        ```yaml
        # Exemple dans une règle 'from' (ingress)
        - podSelector:
            matchLabels:
              role: frontend
        ```
    *   **`namespaceSelector` :** Sélectionne des Pods dans d'autres namespaces (basé sur les labels des namespaces). Souvent combiné avec un `podSelector` pour affiner la sélection des Pods dans ces namespaces.
        ```yaml
        # Exemple dans une règle 'to' (egress)
        - namespaceSelector:
            matchLabels:
              project: my-database-project
          # Optionnel: podSelector pour les pods dans les namespaces sélectionnés
          # podSelector:
          #   matchLabels:
          #     app: database
        ```
    *   **`ipBlock` :** Spécifie des plages d'adresses IP en format CIDR. Utile pour autoriser le trafic vers/depuis des adresses IP externes au cluster (par exemple, Internet, des serveurs on-premise).
        ```yaml
        # Exemple dans une règle 'to' (egress)
        - ipBlock:
            cidr: 192.168.5.0/24
            except: # Optionnel: exclure des IPs de ce bloc
              - 192.168.5.10
        ```
    *   **Si `from` ou `to` est omis ou vide (`[]`) dans une règle, cela ne correspond à aucune source/destination.** Pour autoriser depuis/vers toutes les sources/destinations (par exemple, pour un "allow all" spécifique à un port après un "default deny"), il faut utiliser une règle avec un sélecteur vide, comme `from: [{}]` (voir exemples).

*   **`ports` :** Définit les ports et protocoles autorisés. C'est une liste d'éléments.
    *   Chaque élément peut spécifier `protocol` (TCP, UDP, SCTP ; défaut à TCP) et `port` (numéro de port ou nom de port si défini dans les Pods).
    *   **Si `ports` est omis dans une règle**, cela signifie que la règle s'applique à **tous les ports** pour les sources/destinations spécifiées par `from`/`to`.

## Cas d'Usage et Exemples

### 1. Default Deny All (Ingress et Egress)

Cette politique sélectionne tous les Pods dans le namespace `my-namespace` et, comme aucune règle `ingress` ou `egress` n'est définie, elle bloque tout le trafic entrant et sortant pour ces Pods.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: my-namespace
spec:
  podSelector: {} # Sélectionne tous les Pods dans le namespace 'my-namespace'
  policyTypes:    # Spécifie les types de trafic affectés par cette politique
  - Ingress
  - Egress
# Aucune section 'ingress' ou 'egress' n'est définie, donc :
# - Tout trafic entrant est REFUSÉ.
# - Tout trafic sortant est REFUSÉ.
```
**Explication de `policyTypes` ici :**
*   En incluant `Ingress` et `Egress` dans `policyTypes`, nous indiquons que cette politique gère ces deux types de flux.
*   L'absence de sections `ingress:` et `egress:` signifie qu'il n'y a aucune règle d'autorisation pour ces flux. Par conséquent, pour les Pods sélectionnés (tous dans le namespace), tout trafic entrant et sortant est refusé.

### 2. Autoriser l'Ingress depuis des Pods Spécifiques (Même Namespace)

Permet aux Pods "frontend" de communiquer avec les Pods "backend" sur un port spécifique.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: my-namespace
spec:
  podSelector: # Appliquer aux Pods backend
    matchLabels:
      app: backend
  policyTypes:
  - Ingress # Cette politique ne concerne que le trafic entrant
  ingress:
  - from: # Autoriser le trafic PROVENANT de :
    - podSelector: # Pods ayant le label app=frontend
        matchLabels:
          app: frontend
    ports: # UNIQUEMENT sur ces ports :
    - protocol: TCP
      port: 8080 # Port d'écoute du backend
```

### 3. Autoriser l'Egress vers des Pods Spécifiques dans un Autre Namespace

Permet aux Pods `backend-app` dans `backend-ns` de communiquer avec les Pods `mydb` dans les namespaces ayant le label `team=database-team` sur le port 5432.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-egress-to-database
  namespace: backend-ns # Cette politique s'applique aux Pods dans 'backend-ns'
spec:
  podSelector: # Appliquer aux Pods backend-app
    matchLabels:
      app: backend-app
  policyTypes:
  - Egress # Cette politique ne concerne que le trafic sortant
  egress:
  - to: # Autoriser le trafic VERS :
    - namespaceSelector: # Namespaces ayant le label team=database-team
        matchLabels:
          team: database-team
      podSelector: # ET Pods dans ces namespaces ayant le label app=mydb
        matchLabels:
          app: mydb
    ports: # UNIQUEMENT sur ces ports :
    - protocol: TCP
      port: 5432 # Port PostgreSQL
```

### 4. Autoriser l'Egress vers Internet (CIDR Spécifique)

Permet aux Pods `my-app-needs-internet` d'envoyer du trafic UDP sur le port 53 à l'adresse IP `8.8.8.8` (DNS Google).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-external-api
  namespace: my-namespace
spec:
  podSelector:
    matchLabels:
      app: my-app-needs-internet
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 8.8.8.8/32 # Autorise le trafic vers cette IP spécifique
        # Pour autoriser tout Internet (non recommandé sans autre filtrage) :
        # cidr: 0.0.0.0/0
        # except:
        #  - 10.0.0.0/8 # Exclure les IPs internes
        #  - 192.168.0.0/16
        #  - 172.16.0.0/12
    ports: # Optionnel : restreindre à des ports spécifiques
    - protocol: UDP
      port: 53
```

### 5. Autoriser Tout Ingress / Tout Egress (pour un Pod déjà isolé)

Si un Pod est déjà isolé (par exemple, par une politique "default-deny"), vous pouvez créer une autre politique pour ré-autoriser tout le trafic entrant ou sortant pour ce Pod spécifique.

**Autoriser Tout Ingress :**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress-to-myapp
  namespace: my-namespace
spec:
  podSelector: # Appliquer aux Pods my-app
    matchLabels:
      app: my-app
  policyTypes:
  - Ingress
  ingress:
  - {} # Une liste 'from' vide avec une règle vide {} signifie "autoriser depuis toutes les sources"
       # sur tous les ports (car 'ports' est omis).
```

**Autoriser Tout Egress :**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-egress-from-myapp
  namespace: my-namespace
spec:
  podSelector: # Appliquer aux Pods my-app
    matchLabels:
      app: my-app
  policyTypes:
  - Egress
  egress:
  - {} # Une liste 'to' vide avec une règle vide {} signifie "autoriser vers toutes les destinations"
       # sur tous les ports (car 'ports' est omis).
```
**Note :** Une règle `ingress: [{}]` ou `egress: [{}]` (une liste contenant un objet vide) est la manière correcte de spécifier "toutes sources" ou "toutes destinations". Une section `ingress:` vide ou omise signifie "aucun trafic entrant autorisé".

## Conseils pour Rédiger des Politiques

*   **Commencer par un "Default Deny" :** Si la sécurité est une priorité, commencez par une politique qui refuse tout trafic entrant et sortant pour tous les Pods du namespace, puis ajoutez des politiques pour autoriser sélectivement le trafic nécessaire.
*   **Construire les Politiques Incrémentalement :** Ajoutez des règles petit à petit et testez-les pour éviter de bloquer involontairement du trafic légitime.
*   **Utiliser les Labels Efficacement :** Des labels clairs et cohérents pour les Pods et les Namespaces sont essentiels pour écrire des NetworkPolicies précises et maintenables.
*   **Tester Soigneusement :** Après avoir appliqué une politique, testez la connectivité entre les Pods concernés et avec les services externes pour vous assurer qu'elle fonctionne comme prévu. Des outils comme `netcat` ou `curl` depuis l'intérieur des Pods peuvent être utiles.
*   **Attention aux DNS :** Si vous restreignez l'egress, n'oubliez pas d'autoriser le trafic DNS (généralement UDP port 53) vers les serveurs DNS du cluster (souvent `kube-dns` ou CoreDNS) ou des serveurs DNS externes si nécessaire. Une politique egress trop restrictive peut empêcher les Pods de résoudre les noms d'hôtes.

---

> Next: [Stockage dans Kubernetes](./storage-overview.md)

> [cheat sheet](../useful.md)
