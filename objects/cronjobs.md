# CronJobs Kubernetes

Les CronJobs sont des objets de l'API Kubernetes qui permettent de gérer l'exécution de Jobs de manière récurrente, basée sur une planification temporelle, similaire à l'utilitaire `cron` sous Linux.

## Introduction aux CronJobs

### Que sont-ils ?

Un **CronJob** crée des objets Job à des moments spécifiés par une chaîne de format `cron`. Essentiellement, il s'agit d'un gestionnaire de tâches planifiées au sein de Kubernetes. Le CronJob lui-même ne fait qu'orchestrer la création des Jobs ; ce sont ces Jobs qui ensuite créent et gèrent les Pods pour exécuter les tâches effectives.

### Objectif

L'objectif principal des CronJobs est d'automatiser l'exécution de tâches batch récurrentes. Cela peut inclure :

*   Des tâches de maintenance automatisées.
*   La génération et l'envoi de rapports planifiés.
*   Des sauvegardes périodiques de données.
*   Des synchronisations de données à intervalles réguliers.

## Cas d'Usage

Les CronJobs sont utiles pour une variété de scénarios où une tâche doit être exécutée de manière répétée :

*   **Planification de Sauvegardes Nocturnes :** Lancer un Job chaque nuit pour sauvegarder une base de données ou des volumes persistants.
*   **Envoi de Rapports Quotidiens/Hebdomadaires :** Générer un rapport sur l'état d'une application ou des statistiques d'utilisation et l'envoyer par e-mail tous les jours à une heure précise.
*   **Déclenchement de Synchronisations de Données :** Exécuter un Job toutes les quelques heures pour synchroniser des données entre différents systèmes.
*   **Tâches de Nettoyage Périodiques :** Lancer un Job pour nettoyer des fichiers temporaires, des anciennes entrées de base de données, ou d'autres ressources obsolètes à intervalles réguliers.
*   **Vérifications de Santé Programmées :** Exécuter des scripts de vérification de santé approfondis sur des systèmes à des moments de faible charge.

## Syntaxe du Schedule Cron

La planification (`schedule`) d'un CronJob utilise le format `cron` standard, qui est une chaîne de cinq champs (ou parfois six, si les secondes sont incluses, bien que Kubernetes utilise principalement cinq champs) séparés par des espaces.

**Format : `Minute Heure JourDuMois Mois JourDeLaSemaine`**

*   **Minute :** `0` - `59`
*   **Heure :** `0` - `23`
*   **JourDuMois :** `1` - `31`
*   **Mois :** `1` - `12` (ou noms : JAN, FEB, MAR, APR, MAY, JUN, JUL, AUG, SEP, OCT, NOV, DEC)
*   **JourDeLaSemaine :** `0` - `6` (Dimanche à Samedi ; certaines implémentations utilisent `7` pour Dimanche. Kubernetes utilise `0` pour Dimanche et `6` pour Samedi)

Caractères spéciaux :
*   `*` (astérisque) : Correspond à toutes les valeurs possibles pour le champ.
*   `/` (barre oblique) : Utilisé pour spécifier des pas. Par exemple, `*/15` dans le champ des minutes signifie "toutes les 15 minutes".
*   `-` (tiret) : Utilisé pour définir des plages. Par exemple, `1-5` dans le champ JourDeLaSemaine signifie "du lundi au vendredi".
*   `,` (virgule) : Utilisé pour lister plusieurs valeurs. Par exemple, `0,30` dans le champ des minutes signifie "à la minute 0 et à la minute 30".

**Exemples de Planification :**

*   `0 * * * *` : S'exécute toutes les heures, à la minute 0 (par exemple, 1:00, 2:00, 3:00).
*   `*/15 * * * *` : S'exécute toutes les 15 minutes (par exemple, 0:00, 0:15, 0:30, 0:45, 1:00, etc.).
*   `0 0 * * 0` : S'exécute tous les dimanches à minuit (00:00).
*   `0 8 1 * *` : S'exécute à 8:00 AM le premier jour de chaque mois.
*   `30 22 * * 1-5` : S'exécute à 22:30 (10:30 PM) du lundi au vendredi.

**Fuseau Horaire :** Le fuseau horaire utilisé pour interpréter la planification cron est celui du gestionnaire de contrôleurs Kubernetes (kube-controller-manager). Soyez conscient de cela si votre cluster et vos utilisateurs sont dans des fuseaux horaires différents. Depuis Kubernetes v1.27 (bêta dans v1.25), un champ `timeZone` peut être spécifié dans `spec` du CronJob pour utiliser un fuseau horaire spécifique.

## Comment Fonctionnent les CronJobs

1.  **Planification :** Le contrôleur CronJob vérifie toutes les minutes si l'un de ses CronJobs doit être déclenché en fonction de son `spec.schedule`.
2.  **Création de Job :** Lorsqu'une planification est atteinte, le CronJob crée un objet Job basé sur son `spec.jobTemplate`. Ce `jobTemplate` contient toutes les spécifications d'un Job standard (y compris le template de Pod pour exécuter la tâche).
3.  **Exécution du Job :** Le Job nouvellement créé prend alors le relais et exécute la tâche en créant un ou plusieurs Pods, comme décrit dans la section sur les [Jobs](./jobs.md).
4.  **Historique :** Le CronJob conserve un historique des Jobs récents qu'il a créés (le nombre est configurable).

Le CronJob lui-même ne lance pas directement de Pods. Il délègue cette responsabilité au Job qu'il crée à chaque exécution planifiée.

## Exemple de CronJob YAML

Voici un exemple de CronJob qui simule la génération d'un rapport quotidien.

```yaml
apiVersion: batch/v1 # Utilisez batch/v1 pour Kubernetes 1.21+
# Pour les versions plus anciennes (avant 1.21), utilisez batch/v1beta1
kind: CronJob
metadata:
  name: scheduled-report-generator
spec:
  schedule: "0 2 * * *" # S'exécute à 2h00 du matin tous les jours
  jobTemplate: # Définit le Job qui sera créé
    spec:
      # --- Spécification du Job ---
      # Optionnel: si le job peut prendre beaucoup de temps
      # activeDeadlineSeconds: 600 # Le Job doit se terminer en 10 minutes
      # backoffLimit: 3 # Nombre de tentatives pour le Job
      template: # Modèle pour les Pods créés par le Job
        spec:
          containers:
          - name: report-generator
            image: my-report-tool:latest # Remplacez par votre image
            args:
            - "--generate-daily-summary"
            # Ajoutez ici la configuration de votre conteneur (ports, volumes, etc.)
          restartPolicy: OnFailure # Politique de redémarrage pour les conteneurs dans le Pod du Job
  # --- Options spécifiques au CronJob ---
  concurrencyPolicy: Allow # Politique de concurrence (Allow, Forbid, Replace)
  suspend: false # Mettre à true pour désactiver la planification de nouveaux Jobs
  successfulJobsHistoryLimit: 3 # Conserver l'historique des 3 derniers Jobs réussis (défaut: 3)
  failedJobsHistoryLimit: 1     # Conserver l'historique du dernier Job échoué (défaut: 1)
  # startingDeadlineSeconds: 100 # Optionnel: voir section ci-dessous
```

### Explication des Champs Clés du `spec` CronJob

*   **`schedule` :** (Obligatoire) La chaîne de planification au format cron, comme décrit ci-dessus.
*   **`jobTemplate` :** (Obligatoire) Contient la spécification complète d'un Job (`spec` d'un objet Job). C'est ce Job qui sera créé à chaque exécution planifiée.
    *   `jobTemplate.spec.template` : Définit le Pod qui effectuera le travail.
    *   `jobTemplate.spec.activeDeadlineSeconds` : (Optionnel) Durée maximale pendant laquelle le Job peut s'exécuter.
    *   `jobTemplate.spec.backoffLimit` : (Optionnel) Nombre de tentatives pour le Job avant d'être marqué comme échoué.
*   **`concurrencyPolicy` :** (Optionnel) Spécifie comment traiter les exécutions concurrentes d'un Job si une exécution précédente n'est pas encore terminée.
    *   `Allow` (par défaut) : Permet à plusieurs Jobs créés par ce CronJob de s'exécuter simultanément.
    *   `Forbid` : Empêche le démarrage d'un nouveau Job si une exécution précédente du Job n'est pas encore terminée. Le nouveau Job est ignoré.
    *   `Replace` : Si une exécution précédente du Job est toujours en cours, le nouveau Job la remplace (c'est-à-dire que l'ancien Job est annulé et supprimé, et le nouveau démarre).
*   **`suspend` :** (Optionnel, défaut à `false`) Si mis à `true`, toutes les exécutions futures du CronJob sont désactivées. Les Jobs déjà démarrés ne sont pas affectés. Utile pour désactiver temporairement une tâche sans supprimer le CronJob.
*   **`successfulJobsHistoryLimit` :** (Optionnel, défaut à `3`) Le nombre de Jobs terminés avec succès à conserver dans l'historique.
*   **`failedJobsHistoryLimit` :** (Optionnel, défaut à `1`) Le nombre de Jobs terminés en échec à conserver dans l'historique.
    Mettre ces limites à `0` signifie ne conserver aucun historique pour le type de Job correspondant.

## Deadline de Démarrage (`startingDeadlineSeconds`)

*   **`spec.startingDeadlineSeconds` :** (Optionnel) Ce champ spécifie la durée maximale en secondes après l'heure planifiée pendant laquelle un Job peut encore être démarré s'il a manqué sa planification.
*   **Scénario :** Si le contrôleur CronJob est en panne ou surchargé, un Job peut manquer son heure de déclenchement.
*   **Comportement :** Si `startingDeadlineSeconds` est défini (par exemple, à `100` secondes), le Job sera quand même lancé s'il peut l'être dans les 100 secondes suivant l'heure manquée. S'il ne peut pas être lancé dans ce délai, l'exécution est considérée comme manquée.
*   Si ce champ n'est pas défini, les Jobs manqués ne sont pas relancés (ou plutôt, ils sont considérés comme manqués immédiatement s'ils ne peuvent pas être créés à l'heure exacte).

## Statut du CronJob

Vous pouvez inspecter les CronJobs et les Jobs qu'ils créent :

*   **Lister les CronJobs :**
    ```bash
    kubectl get cronjobs
    # Alias: kubectl get cj
    ```
    La sortie typique inclut `NAME`, `SCHEDULE`, `SUSPEND`, `ACTIVE`, `LAST SCHEDULE`, `AGE`.
    *   `ACTIVE` : Nombre de Jobs actuellement en cours d'exécution créés par ce CronJob.
    *   `LAST SCHEDULE` : Heure de la dernière exécution planifiée.
*   **Décrire un CronJob :**
    ```bash
    kubectl describe cronjob <nom-du-cronjob>
    # Exemple: kubectl describe cronjob scheduled-report-generator
    ```
    Cette commande affiche des détails, y compris la planification, la politique de concurrence, les limites d'historique, les événements récents (comme la création de Jobs), et une liste des Jobs actifs et récemment terminés.
*   **Voir les Jobs créés par un CronJob :**
    Les Jobs créés par un CronJob portent généralement un label qui les lie au CronJob parent. Le nom du CronJob est souvent utilisé dans le nom du Job créé (par exemple, `scheduled-report-generator-1672531200`).
    Vous pouvez lister tous les Jobs et filtrer par label si vous en avez configuré un, ou simplement observer les noms.
    ```bash
    kubectl get jobs
    ```
    Pour voir quels Jobs sont gérés par un CronJob spécifique, vous pouvez regarder les événements dans la sortie de `kubectl describe cronjob <nom-du-cronjob>`.

---

> Next: [Gestion de la Configuration](./configApp.md)

> [cheat sheet](../useful.md)
