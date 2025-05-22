# Formation/Notes Kubernetes

Bienvenue dans cette formation Kubernetes ! Ce document sert de point d'entrée principal et de table des matières pour naviguer à travers les différents modules d'apprentissage.

## Table des Matières

1.  **Introduction à Kubernetes**
    *   [Architecture de Kubernetes](./introduction/architecture.md)
    *   [Contexte et Configuration (`kubectl`)](./introduction/context.md)

2.  **Concepts et Objets Fondamentaux de Kubernetes**
    *   [Pods](./objects/pods.md)
    *   [Namespaces (Espaces de Noms)](./objects/namespace.md)
    *   [Labels et Sélecteurs](./objects/labels-selectors.md)
    *   [Annotations](./objects/annotations.md)
    *   [Services](./objects/services.md)
    *   [Deployments (Déploiements)](./objects/deployement.md)
    *   [ReplicaSets](./objects/replicatSet.md)

3.  **Gestion des Charges de Travail Avancées**
    *   [StatefulSets](./objects/statefulsets.md)
    *   [DaemonSets](./objects/daemonsets.md)
    *   [Jobs (Tâches Batch)](./objects/jobs.md)
    *   [CronJobs (Tâches Planifiées)](./objects/cronjobs.md)

4.  **Gestion de la Configuration des Applications**
    *   [ConfigMaps et Secrets](./objects/configApp.md)

5.  **Réseautage dans Kubernetes**
    *   [Ingress](./objects/ingress.md)
    *   [NetworkPolicies (Politiques Réseau)](./objects/networkpolicies.md)

6.  **Stockage dans Kubernetes**
    *   [Vue d'ensemble du Stockage](./objects/storage-overview.md)
    *   [PersistentVolumes (PV)](./objects/persistentvolumes.md)
    *   [PersistentVolumeClaims (PVC)](./objects/persistentvolumeclaims.md)
    *   [StorageClasses](./objects/storageclasses.md)

7.  **Sécurité dans Kubernetes**
    *   [Vue d'ensemble de la Sécurité](./security/security-overview.md)
    *   [Role-Based Access Control (RBAC)](./security/rbac.md)
    *   [Contextes de Sécurité (Security Contexts)](./security/security-contexts.md)

8.  **Autres Ressources Utiles**
    *   [Monitoring (Surveillance)](./divers/monitoring.md)
    *   [Aide-Mémoire `kubectl` (Cheat Sheet)](./useful.md)

---

```python
>>> import this
The Zen of Python, by Tim Peters

Beautiful is better than ugly.
Explicit is better than implicit.
Simple is better than complex.
Complex is better than complicated.
Flat is better than nested.
Sparse is better than dense.
Readability counts.
Special cases aren't special enough to break the rules.
Although practicality beats purity.
Errors should never pass silently.
Unless explicitly silenced.
In the face of ambiguity, refuse the temptation to guess.
There should be one-- and preferably only one --obvious way to do it.
Although that way may not be obvious at first unless you're Dutch.
Now is better than never.
Although never is often better than *right* now.
If the implementation is hard to explain, it's a bad idea.
If the implementation is easy to explain, it may be a good idea.
Namespaces are one honking great idea -- let's do more of those!
>>>
```

Enjoy!
