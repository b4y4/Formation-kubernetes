# Jobs Kubernetes

Les Jobs sont des objets Kubernetes qui créent un ou plusieurs Pods et s'assurent qu'un nombre spécifié d'entre eux se terminent avec succès. Ils sont conçus pour exécuter des tâches finies, par opposition aux processus continus gérés par des Deployments, StatefulSets ou DaemonSets.

## Introduction aux Jobs

### Que sont-ils ?

Un **Job** Kubernetes est un contrôleur qui supervise l'exécution de tâches batch. Il lance des Pods et surveille leur achèvement. Lorsqu'un nombre prédéfini de Pods a terminé sa tâche avec succès (c'est-à-dire, sorti avec un code 0), le Job lui-même est considéré comme achevé.

### Objectif

L'objectif principal des Jobs est d'exécuter des opérations ponctuelles ou par lots qui ont un début et une fin clairs. Contrairement aux applications de longue durée (comme les serveurs web) qui sont censées fonctionner en continu, les Jobs sont faits pour des tâches qui s'arrêtent une fois leur travail accompli.

*   **Tâches Finies vs. Processus Continus :**
    *   **Deployments/StatefulSets/DaemonSets :** Gèrent des Pods qui sont censés s'exécuter indéfiniment (par exemple, un serveur web, une base de données, un agent de nœud). Si un Pod s'arrête, il est redémarré.
    *   **Jobs :** Gèrent des Pods qui exécutent une tâche spécifique et s'arrêtent. Le Job ne redémarre pas indéfiniment un Pod qui a réussi sa tâche ; il s'assure simplement que la tâche est accomplie le nombre de fois requis.

## Cas d'Usage

Les Jobs sont adaptés pour une variété de tâches batch et d'opérations ponctuelles :

*   **Scripts de Migration de Base de Données :** Exécuter un script qui met à jour un schéma de base de données ou migre des données avant le déploiement d'une nouvelle version d'application.
*   **Opérations de Sauvegarde :** Lancer une tâche pour sauvegarder des données d'une base de données ou d'un volume persistant vers un stockage externe.
*   **Tâches d'Envoi d'E-mails en Lot :** Envoyer des newsletters ou des notifications par e-mail à une liste d'utilisateurs.
*   **Traitement de Données :** Exécuter des calculs intensifs, des tâches d'analyse de données, ou des conversions de format sur un ensemble de données, qui se terminent une fois le traitement achevé.
*   **Nettoyage de Ressources :** Exécuter une tâche pour nettoyer des ressources temporaires ou des fichiers anciens.
*   **Tests d'Intégration :** Exécuter une suite de tests qui s'arrête après avoir donné un résultat.

## Comment Fonctionnent les Jobs

Le fonctionnement d'un Job est relativement simple :

1.  **Création de Pods :** Le contrôleur de Job crée un ou plusieurs Pods basés sur le `spec.template` fourni dans la définition du Job. Ce template de Pod contient la spécification du conteneur qui exécutera la tâche réelle (par exemple, l'image, la commande à exécuter).
2.  **Suivi des Achèvements :** Le Job surveille les Pods qu'il a créés. Un Pod est considéré comme ayant réussi s'il se termine avec un code de sortie 0.
3.  **Achèvement du Job :** Le Job est considéré comme complet lorsque le nombre souhaité de Pods (défini par `spec.completions`) s'est terminé avec succès. Une fois le Job terminé, il ne crée plus de Pods.

## Types de Jobs

Il existe principalement deux types de Jobs, basés sur la manière dont les Pods sont exécutés :

### 1. Jobs Non-Parallèles

*   **Description :** C'est le type de Job le plus simple. Un seul Pod est créé. Le Job est considéré comme terminé dès que ce Pod unique se termine avec succès.
*   **Configuration :** C'est le comportement par défaut si `spec.completions` et `spec.parallelism` ne sont pas spécifiés, ou s'ils sont tous les deux explicitement mis à 1.
*   **Cas d'usage :** Une tâche simple qui n'a pas besoin d'être parallélisée, comme un script de migration unique.

### 2. Jobs Parallèles avec un Nombre Fixe d'Achèvements

*   **Description :** Ce type de Job exécute plusieurs Pods, et le Job est terminé lorsque un nombre spécifié (`spec.completions`) de Pods se sont terminés avec succès.
*   **Configuration :**
    *   `spec.completions` : Nombre total de Pods qui doivent se terminer avec succès pour que le Job soit considéré comme achevé.
    *   `spec.parallelism` : Nombre maximum de Pods que le Job peut exécuter simultanément. Les Pods sont créés en parallèle jusqu'à cette limite.
*   **Exemple de fonctionnement :**
    *   Si `completions: 10` et `parallelism: 2` :
        *   Le Job démarre 2 Pods.
        *   Si un Pod réussit, le compteur d'achèvements s'incrémente. Le Job démarre un nouveau Pod pour maintenir le parallélisme à 2 (tant que le nombre total d'achèvements n'est pas atteint et qu'il y a des Pods en cours ou en attente de démarrage).
        *   Le Job est terminé lorsque 10 Pods ont réussi.
*   **Cas d'usage :** Traiter une file de N éléments où chaque Pod prend un élément, ou exécuter une simulation M fois.

### (Moins Courant) Jobs Parallèles avec une File de Travail

*   **Description :** Dans ce mode, les Pods sont généralement configurés pour communiquer avec une file de messages externe (comme RabbitMQ, Kafka, Redis) pour coordonner le travail. Chaque Pod prend une unité de travail de la file, la traite, et se termine. Le Job lui-même n'est pas directement conscient de la file de travail ; il s'assure simplement qu'un certain nombre de Pods (`spec.parallelism`) sont en cours d'exécution jusqu'à ce qu'un critère d'achèvement soit rempli (souvent géré par un Pod coordinateur ou lorsque la file est vide, ce qui est une logique externe au Job Kubernetes).
*   **Configuration :** `spec.parallelism` est supérieur à 0, et `spec.completions` n'est pas défini ou est égal à `spec.parallelism`. Le Job est considéré comme complet lorsqu'au moins un Pod a réussi et que tous les Pods se sont terminés.
*   **Note :** Ce pattern est souvent mieux géré avec des Jobs à nombre fixe d'achèvements où chaque Pod gère sa propre logique de file de travail, ou avec des outils d'orchestration de workflows plus avancés. Pour la plupart des cas d'usage de "file de travail", on utilise un Job avec `completions` où chaque "complétion" représente le traitement d'un élément de la file.

## Exemple de Job YAML

Voici un exemple simple de Job qui utilise une image Perl standard pour calculer les décimales de Pi.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calculator
spec:
  template: # Modèle pour les Pods que le Job va créer
    spec:
      containers:
      - name: pi
        image: perl # Image standard contenant l'interpréteur Perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"] # Calcule 2000 décimales de Pi
      restartPolicy: Never # Politique de redémarrage pour les conteneurs dans le Pod
  # --- Options pour la gestion du Job ---
  # completions: 1 # (Par défaut si non spécifié pour un Job non parallèle)
  # parallelism: 1 # (Par défaut si non spécifié pour un Job non parallèle)
  
  # --- Options pour Jobs parallèles ---
  # completions: 5 # Le Pod (ou la tâche) doit réussir 5 fois.
  # parallelism: 2 # Exécuter jusqu'à 2 Pods en parallèle.
  
  backoffLimit: 4  # Nombre de tentatives avant de marquer le Job comme échoué (par défaut 6).
                   # Le compteur est réinitialisé si un Pod réussit.
```

### Explication des Champs Clés

*   **`spec.template` :** C'est la partie la plus importante, car elle définit le Pod qui exécutera la tâche. Elle est identique à la section `template` d'un Deployment ou d'un StatefulSet.
    *   **`spec.template.spec.restartPolicy` :** Définit la politique de redémarrage pour les conteneurs *au sein* du Pod créé par le Job. Pour les Jobs, les valeurs autorisées sont :
        *   `Never` : Si un conteneur échoue, le Pod entier est marqué comme échoué. Le contrôleur de Job décidera alors s'il doit créer un nouveau Pod (en fonction de `backoffLimit`). C'est une politique courante pour les tâches batch où un échec de conteneur signifie un échec de la tentative de tâche.
        *   `OnFailure` : Si un conteneur échoue, il peut être redémarré au sein du même Pod. Le Pod lui-même ne sera marqué comme échoué que si les redémarrages du conteneur continuent d'échouer.
        *   `Always` : **Non autorisé** pour les Jobs, car cela impliquerait un redémarrage continu même après une exécution réussie, ce qui est contraire à la nature d'une tâche finie.
*   **`spec.completions` :** (Optionnel, défaut à 1) Le nombre de Pods qui doivent se terminer avec succès.
*   **`spec.parallelism` :** (Optionnel, défaut à 1) Le nombre maximum de Pods qui peuvent s'exécuter en parallèle.
*   **`spec.backoffLimit` :** (Optionnel, défaut à 6) Le nombre de fois qu'un Job va réessayer un Pod (en créant un nouveau Pod) si le Pod précédent échoue, avant de marquer le Job entier comme échoué. Le délai entre les tentatives augmente exponentiellement (10s, 20s, 40s, ... plafonné à 6 minutes).

## Gestion des Échecs de Pods et Conteneurs

*   **Échec de Conteneur :**
    *   Si `restartPolicy: OnFailure` est utilisé, le kubelet sur le nœud essaiera de redémarrer le conteneur défaillant dans le même Pod.
    *   Si `restartPolicy: Never` est utilisé, ou si les redémarrages `OnFailure` échouent de manière répétée, le Pod entier est marqué comme échoué.
*   **Échec de Pod :**
    *   Lorsque le Job détecte qu'un Pod a échoué (par exemple, un conteneur s'est terminé avec un code de sortie non nul avec `restartPolicy: Never`, ou le Pod a dépassé son propre `activeDeadlineSeconds`), il respecte le `backoffLimit`.
    *   Si le `backoffLimit` n'est pas atteint, le Job crée un nouveau Pod pour remplacer celui qui a échoué.
    *   Si le `backoffLimit` est atteint, le Job est marqué comme échoué et ne crée plus de Pods.

## Statut du Job

Vous pouvez surveiller l'état d'un Job en utilisant `kubectl` :

*   **Lister les Jobs :**
    ```bash
    kubectl get jobs
    # Ou avec plus de détails : kubectl get jobs -o wide
    ```
    La sortie montre le nombre de complétions (par exemple, `COMPLETIONS   DURATION   AGE` -> `1/1           12s        60s`).
*   **Décrire un Job :**
    ```bash
    kubectl describe job <nom-du-job>
    # Exemple: kubectl describe job pi-calculator
    ```
    Cette commande fournit des informations détaillées, y compris :
    *   Le nombre de Pods actifs, réussis (`Succeeded`), et échoués (`Failed`).
    *   Les événements récents liés au Job (création de Pods, échecs, etc.).
*   **Voir les Pods créés par un Job :**
    Les Pods créés par un Job ont généralement des labels qui permettent de les retrouver. Le nom du Job est souvent utilisé comme label (par exemple, `job-name: pi-calculator`).
    ```bash
    kubectl get pods -l job-name=pi-calculator
    ```
*   **Voir les Logs d'un Pod de Job :**
    ```bash
    kubectl logs <nom-du-pod-du-job>
    ```

## Nettoyage des Jobs Terminés

Par défaut, les Jobs et leurs Pods (même ceux qui ont réussi ou échoué) ne sont pas automatiquement supprimés du cluster après leur achèvement. Cela permet aux utilisateurs d'inspecter leur statut final, leurs logs, et leurs événements.

Cependant, conserver indéfiniment des Jobs terminés et leurs Pods peut consommer des ressources (surtout des entrées dans etcd).

### Suppression Manuelle

Vous pouvez supprimer manuellement un Job et ses Pods associés :
```bash
kubectl delete job <nom-du-job>
# Exemple: kubectl delete job pi-calculator
```
La suppression du Job entraîne la suppression des Pods qu'il a créés.

### Nettoyage Automatique avec `ttlSecondsAfterFinished`

Kubernetes offre un mécanisme pour nettoyer automatiquement les Jobs terminés (qu'ils aient réussi ou échoué) après un certain délai. Cela se fait en utilisant le champ `spec.ttlSecondsAfterFinished`.

*   **Fonctionnement :** Vous spécifiez un nombre de secondes. Une fois que le Job est terminé, le contrôleur TTL (Time-To-Live) attendra ce délai avant de supprimer le Job et ses Pods.
*   **Disponibilité :** Cette fonctionnalité est devenue stable (GA) dans Kubernetes 1.23. Elle était en bêta depuis la version 1.21. Assurez-vous que votre version de Kubernetes la supporte.

**Exemple avec `ttlSecondsAfterFinished` :**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-with-ttl
spec:
  ttlSecondsAfterFinished: 100 # Le Job et ses Pods seront supprimés 100 secondes après la fin du Job
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(100)"]
      restartPolicy: Never
  backoffLimit: 2
```
Dans cet exemple, 100 secondes après que le Job `pi-with-ttl` se termine (soit par succès, soit par échec après avoir atteint son `backoffLimit`), il sera automatiquement supprimé, ainsi que les Pods qu'il a créés.

---

> Next: [CronJobs](./cronjobs.md)

> [cheat sheet](../useful.md)
