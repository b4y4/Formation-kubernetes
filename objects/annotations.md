# Annotations Kubernetes

Les annotations sont un mécanisme Kubernetes permettant d'attacher des métadonnées arbitraires non identifiantes aux objets. Comme les labels, ce sont des paires clé-valeur, mais elles servent des objectifs différents.

## Définition

*   **Paires Clé-Valeur :** Les annotations sont stockées sous forme de paires clé-valeur, où les clés et les valeurs sont des chaînes de caractères.
*   **Métadonnées Non Identifiantes :** Contrairement aux labels, les annotations ne sont pas utilisées pour identifier et sélectionner des objets. Kubernetes ne se base pas sur les annotations pour regrouper ou filtrer des ressources.
*   **Informations Supplémentaires :** Elles permettent d'ajouter des informations riches et souvent plus volumineuses qui peuvent être utiles aux humains ou à des outils externes pour la gestion, l'audit, ou l'intégration.

### Différence Principale avec les Labels

*   **Labels :** Utilisés pour **l'organisation et la sélection** d'objets. Ils sont un critère de filtrage pour les sélecteurs (par exemple, un Service sélectionne des Pods via leurs labels). Les clés et valeurs des labels sont généralement courtes et structurées.
*   **Annotations :** Utilisées pour attacher des **métadonnées descriptives ou fonctionnelles** qui ne servent pas à la sélection. Elles peuvent contenir des informations plus complexes, plus longues, et moins structurées que les labels.

## Objectif et Cas d'Usage

Les annotations sont polyvalentes et peuvent servir à de nombreux usages :

*   **Informations de Build et de Release :**
    *   Stockage du SHA de commit Git, du numéro de version, du timestamp de build, du hash de l'image Docker, ou d'un lien vers le pipeline CI/CD qui a généré la ressource.
    *   Exemple : `build-timestamp: "2023-10-26T14:30:00Z"`
*   **Coordonnées et Informations de Contact :**
    *   Nom de l'équipe responsable, adresse e-mail, numéro de téléphone d'astreinte, ou canal Slack pour des alertes ou des questions.
    *   Exemple : `contact/on-call: "ops-team@example.com"`
*   **Pointeurs vers des Systèmes Externes :**
    *   Liens vers des tableaux de bord de logging (Kibana, Grafana Loki), de monitoring (Prometheus, Grafana), ou d'analytique.
    *   Exemple : `monitoring/dashboard: "https://grafana.example.com/d/abcdef123/my-app-dashboard"`
*   **Instructions pour des Outils Client ou Contrôleurs :**
    *   Configuration pour des contrôleurs personnalisés, des outils GitOps (comme ArgoCD ou Flux), des gestionnaires de certificats (comme cert-manager), ou des outils de maillage de services (service mesh).
    *   Exemple (pour cert-manager) : `cert-manager.io/cluster-issuer: "letsencrypt-prod"`
    *   Exemple (pour Prometheus) : `prometheus.io/scrape: "true"` et `prometheus.io/port: "8080"`
*   **Descriptions Détaillées :**
    *   Fournir une description longue de l'objet, de sa fonction, ou de son comportement, qui serait trop volumineuse pour un label.
    *   Exemple : `description: "Ce pod exécute le serveur d'application principal qui gère les transactions clients."`
*   **Champs Gérés par des Outils Déclaratifs :**
    *   Des outils peuvent utiliser des annotations pour stocker l'état de la configuration appliquée, pour la détection de dérive ou pour des opérations de "three-way merge".

## Syntaxe

*   Les annotations sont des paires clé-valeur, tout comme les labels.
*   Les clés et les valeurs sont des chaînes de caractères.
*   **Taille et Complexité :** Les annotations peuvent contenir des données beaucoup plus volumineuses et complexes que les labels. La taille totale des annotations d'un objet peut aller jusqu'à 256KiB.
*   **Jeu de Caractères pour les Clés :** Les clés d'annotation suivent les mêmes règles que les clés de label :
    *   Un préfixe optionnel (nom de domaine DNS, max 253 caractères, suivi d'un `/`). Les préfixes `kubernetes.io/` et `k8s.io/` sont réservés.
    *   Le nom de la clé (partie après le `/` ou la clé entière si pas de préfixe) doit comporter au maximum 63 caractères, commencer et finir par un alphanumérique, et peut contenir des tirets (`-`), des underscores (`_`), et des points (`.`).

## Exemple de Définition d'Annotations dans un Pod

Voici un exemple de manifeste de Pod illustrant l'utilisation de diverses annotations :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod-annote
  labels:
    app: mon-app
  annotations:
    # Informations de build et de version
    buildVersion: "1.3.5-alpha"
    gitCommit: "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8g9h0"
    dockerImageHash: "sha256:12345abcdef..."
    # Description et documentation
    description: "Ce pod exécute le serveur d'application principal pour le module de traitement des commandes."
    documentationURL: "https://wiki.example.com/apps/commande-processor"
    # Informations de contact
    contact/email: "dev-equipe-commandes@example.com"
    contact/slack-channel: "#equipe-commandes"
    # Instructions pour des outils (par exemple, Prometheus)
    prometheus.io/scrape: "true"
    prometheus.io/port: "9102"
    # Informations pour un outil de déploiement personnalisé
    customdeploy.tool.io/strategy: "blue-green"
spec:
  containers:
  - name: mon-app-conteneur
    image: mon-app-image:1.3.5-alpha # L'image pourrait correspondre à buildVersion
    ports:
    - containerPort: 8080
```

## Gestion des Annotations avec `kubectl`

*   **Afficher les annotations (souvent visible avec `describe`) :**
    ```bash
    kubectl describe pod mon-pod-annote
    ```
    (Les annotations sont généralement listées dans la sortie de `describe`.)

*   **Afficher la configuration YAML complète (y compris toutes les annotations) :**
    ```bash
    kubectl get pod mon-pod-annote -o yaml
    ```

*   **Ajouter ou Mettre à jour une annotation :**
    Si l'annotation n'existe pas, elle est ajoutée. Si elle existe, sa valeur est mise à jour.
    ```bash
    kubectl annotate pod mon-pod-existant nouvelle-annotation="valeur exemple"
    kubectl annotate pod mon-pod-existant description="Nouvelle description pour ce pod." --overwrite
    ```
    Utilisez `--overwrite` pour modifier une annotation existante. Sans cela, `kubectl annotate` refusera de modifier une annotation déjà présente pour éviter les écrasements accidentels.

*   **Supprimer une annotation :**
    Ajoutez un tiret (`-`) à la fin du nom de l'annotation.
    ```bash
    kubectl annotate pod mon-pod-existant description-
    ```

*   **Annoter plusieurs ressources :**
    Vous pouvez annoter plusieurs ressources en même temps.
    ```bash
    kubectl annotate pods --all owner="equipe-infra"
    kubectl annotate pods -l app=nginx contact/email="nginx-admins@example.com" --overwrite
    ```

## Quand Utiliser les Annotations vs. les Labels ?

Le choix entre un label et une annotation dépend de l'utilisation prévue de la métadonnée :

*   **Utilisez les LABELS pour :**
    *   Les attributs qui **identifient** et **distinguent** les objets.
    *   Les métadonnées qui seront utilisées par les **sélecteurs** (par exemple, pour les Services, Deployments, ReplicaSets, Jobs, etc.).
    *   Organiser les objets pour des requêtes `kubectl` (par exemple, `kubectl get pods -l environment=production`).
    *   Regrouper des objets pour des opérations en masse.

*   **Utilisez les ANNOTATIONS pour :**
    *   Les métadonnées **non identifiantes** et **descriptives**.
    *   Les informations qui ne sont pas destinées à être utilisées dans les sélecteurs.
    *   Les données utiles pour les **humains** (descriptions, contacts, liens vers la documentation) ou pour des **outils externes** et des **contrôleurs personnalisés** (configurations spécifiques, informations de build, pointeurs vers des dashboards).
    *   Les métadonnées potentiellement volumineuses ou complexes qui ne respecteraient pas les contraintes de taille ou de complexité des labels.

En résumé : si vous avez besoin de *sélectionner* des objets sur la base de cette métadonnée, utilisez un label. Sinon, une annotation est probablement plus appropriée.

---

> Next: [StatefulSets](./statefulsets.md)

> [cheat sheet](../useful.md)
