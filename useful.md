## Check the K8S cluster health

```bash
curl -k 'https://localhost:6443/readyz?verbose&exclude=etcd'
or
kubectl get --raw='/readyz?verbose'
```

Output

```bash
[+]ping ok
[+]log ok
[+]etcd ok
[+]informer-sync ok
[+]poststarthook/start-kube-apiserver-admission-initializer ok
[+]poststarthook/generic-apiserver-start-informers ok
[+]poststarthook/priority-and-fairness-config-consumer ok
[+]poststarthook/priority-and-fairness-filter ok
[+]poststarthook/start-apiextensions-informers ok
[+]poststarthook/start-apiextensions-controllers ok
[+]poststarthook/crd-informer-synced ok
[+]poststarthook/bootstrap-controller ok
[+]poststarthook/rbac/bootstrap-roles ok
[+]poststarthook/scheduling/bootstrap-system-priority-classes ok
[+]poststarthook/priority-and-fairness-config-producer ok
[+]poststarthook/start-cluster-authentication-info-controller ok
[+]poststarthook/aggregator-reload-proxy-client-cert ok
[+]poststarthook/start-kube-aggregator-informers ok
[+]poststarthook/apiservice-registration-controller ok
[+]poststarthook/apiservice-status-available-controller ok
[+]poststarthook/kube-apiserver-autoregistration ok
[+]autoregister-completion ok
[+]poststarthook/apiservice-openapi-controller ok
[+]shutdown ok
readyz check passed
```

## Get a simple diagnostic for the cluster

```bash
kubectl get componentstatuses
```

Output

```bash
NAME                  STATUS    MESSAGE             ERROR
scheduler             Healthy   ok
controller-manager    Healthy   ok
etcd-0                Healthy   {"health": "true"}
```

## Listing Kubernetes Worker Nodes

```bash
kubectl get nodes
```

Output

```bash
NAME        STATUS        AGE   VERSION
kubernetes  Ready,master  45d   v1.12.1
node-1      Ready         45d   v1.12.1
node-2      Ready         45d   v1.12.1
node-3      Ready         45d   v1.12.1
```

## Get more information about a node
```bash
kubectl describe nodes <nom-du-noeud>
# Alias: kubectl describe no <nom-du-noeud>
```
*Affiche des informations détaillées sur le nœud.*

---
## Gestion des Ressources de Base et Organisation

### Namespaces (Espaces de Noms)
Les Namespaces permettent de partitionner les ressources du cluster.

```bash
# Lister tous les namespaces
kubectl get namespaces
# Alias: kubectl get ns

# Décrire un namespace spécifique
kubectl describe namespace <nom-namespace>
# Alias: kubectl describe ns <nom-namespace>

# Créer un nouveau namespace
kubectl create namespace <nom-namespace>
# Alias: kubectl create ns <nom-namespace>

# Supprimer un namespace (Attention: supprime toutes les ressources qu'il contient!)
kubectl delete namespace <nom-namespace>
# Alias: kubectl delete ns <nom-namespace>

# Changer le contexte kubectl actuel pour utiliser un namespace par défaut
kubectl config set-context $(kubectl config current-context) --namespace=<nom-namespace>
```

### Labels et Sélecteurs
Les Labels sont des paires clé-valeur attachées aux objets, utilisées par les Sélecteurs pour les filtrer.

```bash
# Afficher les pods avec leurs labels
kubectl get pods --show-labels

# Lister les pods avec un label spécifique (égalité)
kubectl get pods -l environment=production
# Alias: kubectl get po -l environment=production

# Lister les pods avec plusieurs labels (ET logique)
kubectl get pods -l 'app=nginx,environment=dev'

# Lister les pods avec un label dont la valeur est dans un ensemble (IN)
kubectl get pods -l 'environment in (production, qa)'

# Lister les pods qui ont un label spécifique (existence)
kubectl get pods -l 'exists(tier)'
# Lister les pods qui n'ont PAS un label spécifique
kubectl get pods -l '!exists(deprecated-label)' # Note: la syntaxe peut varier selon le shell

# Ajouter un label à un pod
kubectl label pods <nom-pod> nouvelle-version=v1.2.3

# Modifier un label existant sur un pod (nécessite --overwrite)
kubectl label pods <nom-pod> environnement=staging --overwrite

# Supprimer un label d'un pod (en ajoutant un tiret à la fin du nom du label)
kubectl label pods <nom-pod> ancienne-version-
```

### Annotations
Les Annotations permettent d'attacher des métadonnées non identifiantes aux objets.

```bash
# Ajouter/Mettre à jour une annotation sur un pod
kubectl annotate pod <nom-pod> description="Ceci est mon pod de test"
# Si l'annotation existe, utiliser --overwrite pour la modifier
kubectl annotate pod <nom-pod> description="Nouvelle description" --overwrite

# Supprimer une annotation d'un pod (en ajoutant un tiret à la fin du nom de l'annotation)
kubectl annotate pod <nom-pod> description-

# Les annotations sont visibles avec describe ou en format YAML/JSON
kubectl describe pod <nom-pod>
kubectl get pod <nom-pod> -o yaml
```

---
## Gestion des Pods et Déploiements (Workloads)

### Pods
Commandes de base pour interagir avec les Pods.
```bash
# Lister tous les pods dans le namespace actuel
kubectl get pods
# Alias: kubectl get po

# Lister les pods dans tous les namespaces
kubectl get pods --all-namespaces
# Alias: kubectl get po -A

# Obtenir les détails d'un pod (configuration, statut, événements)
kubectl describe pod <nom-pod> -n <nom-namespace> # Remplacer <nom-namespace> si pas default
# Alias: kubectl describe po <nom-pod> -n <nom-namespace>

# Afficher les logs d'un pod
kubectl logs <nom-pod> -n <nom-namespace>
# Pour suivre les logs en temps réel:
kubectl logs -f <nom-pod> -n <nom-namespace>
# Pour voir les logs d'un conteneur spécifique dans un pod multi-conteneurs:
kubectl logs <nom-pod> -c <nom-conteneur> -n <nom-namespace>
# Pour voir les logs des pods précédents (si redémarrage)
kubectl logs --previous <nom-pod> -n <nom-namespace>


# Exécuter une commande dans un pod
kubectl exec <nom-pod> -n <nom-namespace> -- <commande> <args...>
# Exemple: kubectl exec my-pod -n default -- ls /app
# Pour obtenir un shell interactif:
kubectl exec -it <nom-pod> -n <nom-namespace> -- /bin/sh # ou /bin/bash

# Copier des fichiers depuis/vers un pod
kubectl cp <nom-pod>:<chemin-source-pod> <chemin-destination-local> -n <nom-namespace>
kubectl cp <chemin-source-local> <nom-pod>:<chemin-destination-pod> -n <nom-namespace>

# Supprimer un pod (sera souvent recréé par un Deployment, ReplicaSet, etc.)
kubectl delete pod <nom-pod> -n <nom-namespace>
```

### Deployments
Gèrent le déploiement et la mise à jour des applications sans état.
```bash
# Lister les deployments
kubectl get deployments -n <nom-namespace>
# Alias: kubectl get deploy -n <nom-namespace>

# Décrire un deployment
kubectl describe deployment <nom-deployment> -n <nom-namespace>
# Alias: kubectl describe deploy <nom-deployment> -n <nom-namespace>

# Mettre à l'échelle un deployment (changer le nombre de réplicas)
kubectl scale deployment <nom-deployment> --replicas=5 -n <nom-namespace>

# Voir l'état d'un déploiement (rolling update)
kubectl rollout status deployment/<nom-deployment> -n <nom-namespace>

# Voir l'historique des révisions d'un deployment
kubectl rollout history deployment/<nom-deployment> -n <nom-namespace>

# Revenir à une révision précédente
kubectl rollout undo deployment/<nom-deployment> --to-revision=<numero-revision> -n <nom-namespace>

# Redémarrer un déploiement (provoque une mise à jour progressive avec la même spec)
kubectl rollout restart deployment/<nom-deployment> -n <nom-namespace>

# Supprimer un deployment (supprime aussi les ReplicaSets et Pods associés)
kubectl delete deployment <nom-deployment> -n <nom-namespace>
```

### StatefulSets
Pour les applications avec état nécessitant des identités stables et un stockage persistant.
```bash
# Lister les StatefulSets
kubectl get statefulsets -n <nom-namespace>
# Alias: kubectl get sts -n <nom-namespace>

# Décrire un StatefulSet
kubectl describe statefulset <nom-sts> -n <nom-namespace>
# Alias: kubectl describe sts <nom-sts> -n <nom-namespace>

# Mettre à l'échelle un StatefulSet
kubectl scale statefulset <nom-sts> --replicas=<nombre> -n <nom-namespace>

# Voir l'état du déploiement d'un StatefulSet
kubectl rollout status statefulset/<nom-sts> -n <nom-namespace>

# Supprimer un StatefulSet (Attention: les PVCs peuvent ne pas être supprimés, selon la reclaimPolicy)
kubectl delete statefulset <nom-sts> -n <nom-namespace>
```

### DaemonSets
Assurent qu'une copie d'un Pod s'exécute sur tous les nœuds (ou un sous-ensemble).
```bash
# Lister les DaemonSets
kubectl get daemonsets -n <nom-namespace>
# Alias: kubectl get ds -n <nom-namespace>

# Décrire un DaemonSet
kubectl describe daemonset <nom-ds> -n <nom-namespace>
# Alias: kubectl describe ds <nom-ds> -n <nom-namespace>

# Voir l'état du déploiement d'un DaemonSet
kubectl rollout status daemonset/<nom-ds> -n <nom-namespace>

# Supprimer un DaemonSet
kubectl delete daemonset <nom-ds> -n <nom-namespace>
```

### Jobs
Pour exécuter des tâches batch qui se terminent.
```bash
# Lister les Jobs
kubectl get jobs -n <nom-namespace>

# Décrire un Job (utile pour voir les Pods créés et leur statut)
kubectl describe job <nom-job> -n <nom-namespace>

# Afficher les logs d'un Pod créé par un Job (le nom du Pod est souvent <nom-job>-<hash>)
# D'abord, trouver le nom du Pod: kubectl get pods -l job-name=<nom-job> -n <nom-namespace>
kubectl logs <nom-pod-du-job> -n <nom-namespace>

# Attendre qu'un Job se termine
kubectl wait --for=condition=complete job/<nom-job> -n <nom-namespace> --timeout=300s

# Supprimer un Job (et les Pods qu'il a créés)
kubectl delete job <nom-job> -n <nom-namespace>
```

### CronJobs
Pour planifier l'exécution récurrente de Jobs.
```bash
# Lister les CronJobs
kubectl get cronjobs -n <nom-namespace>
# Alias: kubectl get cj -n <nom-namespace>

# Décrire un CronJob (voir la planification, le template de Job, les Jobs récents)
kubectl describe cronjob <nom-cj> -n <nom-namespace>
# Alias: kubectl describe cj <nom-cj> -n <nom-namespace>

# Surveiller les Jobs créés par les CronJobs (utile pour le débogage)
kubectl get jobs --watch -n <nom-namespace>

# Déclencher manuellement un Job à partir d'un template de CronJob
kubectl create job <nom-job-manuel> --from=cronjob/<nom-cj> -n <nom-namespace>

# Suspendre/Reprendre un CronJob
kubectl patch cronjob <nom-cj> -p '{"spec": {"suspend": true}}' -n <nom-namespace>
kubectl patch cronjob <nom-cj> -p '{"spec": {"suspend": false}}' -n <nom-namespace>

# Supprimer un CronJob (ne supprime pas les Jobs actifs ou terminés qu'il a créés)
kubectl delete cronjob <nom-cj> -n <nom-namespace>
```

---
## Configuration des Applications

### ConfigMaps
Pour stocker des données de configuration non sensibles.
```bash
# Lister les ConfigMaps
kubectl get configmaps -n <nom-namespace>
# Alias: kubectl get cm -n <nom-namespace>

# Décrire un ConfigMap
kubectl describe configmap <nom-cm> -n <nom-namespace>
# Alias: kubectl describe cm <nom-cm> -n <nom-namespace>

# Créer un ConfigMap à partir de littéraux ou de fichiers
kubectl create configmap <nom-cm> \
  --from-literal=cle1=valeur1 \
  --from-literal=cle2=valeur2 \
  --from-file=chemin/vers/fichier.conf \
  -n <nom-namespace>

# Modifier un ConfigMap (par exemple, en l'ouvrant dans l'éditeur par défaut)
kubectl edit configmap <nom-cm> -n <nom-namespace>

# Supprimer un ConfigMap
kubectl delete configmap <nom-cm> -n <nom-namespace>
```

### Secrets
Pour stocker des données sensibles (mots de passe, tokens, clés).
```bash
# Lister les Secrets
kubectl get secrets -n <nom-namespace>

# Décrire un Secret (ne montre PAS les valeurs décodées)
kubectl describe secret <nom-secret> -n <nom-namespace>

# Créer un Secret générique à partir de littéraux ou de fichiers
kubectl create secret generic <nom-secret> \
  --from-literal=utilisateur=admin \
  --from-literal=motdepasse='S!cr3t' \
  --from-file=chemin/vers/cert.pem \
  -n <nom-namespace>

# Créer un Secret pour un registre Docker privé
kubectl create secret docker-registry <nom-secret-registre> \
  --docker-server=<serveur-registre> \
  --docker-username=<utilisateur> \
  --docker-password=<mot-de-passe> \
  --docker-email=<email> \
  -n <nom-namespace>

# Afficher une valeur de Secret (décodée)
# ATTENTION: Affiche des informations sensibles en clair dans votre terminal!
echo $(kubectl get secret <nom-secret> -n <nom-namespace> -o jsonpath='{.data.<cle-du-secret>}' | base64 --decode)
# Exemple: echo $(kubectl get secret my-db-secret -n default -o jsonpath='{.data.password}' | base64 --decode)

# Supprimer un Secret
kubectl delete secret <nom-secret> -n <nom-namespace>
```

---
## Réseautage (Networking)

### Services
Exposent des applications s'exécutant sur un ensemble de Pods.
```bash
# Lister les services
kubectl get services -n <nom-namespace>
# Alias: kubectl get svc -n <nom-namespace>

# Décrire un service
kubectl describe service <nom-service> -n <nom-namespace>
# Alias: kubectl describe svc <nom-service> -n <nom-namespace>

# Exposer un Deployment avec un Service (type ClusterIP par défaut)
kubectl expose deployment <nom-deployment> --port=<port-externe> --target-port=<port-conteneur> -n <nom-namespace>
# Pour un type NodePort: kubectl expose deployment <nom-deployment> --type=NodePort --port=80 --target-port=8080

# Supprimer un service
kubectl delete service <nom-service> -n <nom-namespace>
```

### Ingress
Gère l'accès externe aux services HTTP/HTTPS du cluster.
```bash
# Lister les Ingress
kubectl get ingress -n <nom-namespace>
# Alias: kubectl get ing -n <nom-namespace>

# Décrire un Ingress
kubectl describe ingress <nom-ingress> -n <nom-namespace>
# Alias: kubectl describe ing <nom-ingress> -n <nom-namespace>

# Supprimer un Ingress
kubectl delete ingress <nom-ingress> -n <nom-namespace>
```

### NetworkPolicies (Politiques Réseau)
Contrôlent le flux de trafic entre les Pods.
```bash
# Lister les NetworkPolicies
kubectl get networkpolicies -n <nom-namespace>
# Alias: kubectl get netpol -n <nom-namespace>

# Décrire une NetworkPolicy
kubectl describe networkpolicy <nom-netpol> -n <nom-namespace>
# Alias: kubectl describe netpol <nom-netpol> -n <nom-namespace>

# Supprimer une NetworkPolicy
kubectl delete networkpolicy <nom-netpol> -n <nom-namespace>
```

---
## Stockage (Storage)

### PersistentVolumes (PVs)
Représentent des morceaux de stockage dans le cluster.
```bash
# Lister les PersistentVolumes
kubectl get persistentvolumes
# Alias: kubectl get pv

# Décrire un PersistentVolume
kubectl describe persistentvolume <nom-pv>
# Alias: kubectl describe pv <nom-pv>

# Supprimer un PersistentVolume (si la politique de récupération le permet ou pour nettoyage manuel)
kubectl delete persistentvolume <nom-pv>
```

### PersistentVolumeClaims (PVCs)
Demandes de stockage par les utilisateurs/applications.
```bash
# Lister les PersistentVolumeClaims
kubectl get persistentvolumeclaims -n <nom-namespace>
# Alias: kubectl get pvc -n <nom-namespace>

# Décrire un PersistentVolumeClaim
kubectl describe persistentvolumeclaim <nom-pvc> -n <nom-namespace>
# Alias: kubectl describe pvc <nom-pvc> -n <nom-namespace>

# Supprimer un PersistentVolumeClaim (peut déclencher la politique de récupération du PV)
kubectl delete persistentvolumeclaim <nom-pvc> -n <nom-namespace>
```

### StorageClasses
Définissent les "classes" de stockage disponibles.
```bash
# Lister les StorageClasses
kubectl get storageclasses
# Alias: kubectl get sc

# Décrire une StorageClass
kubectl describe storageclass <nom-sc>
# Alias: kubectl describe sc <nom-sc>

# Marquer une StorageClass comme défaut (remplacer <nom-sc> par le nom de la classe)
kubectl patch storageclass <nom-sc> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Retirer le statut par défaut d'une StorageClass
kubectl patch storageclass <nom-sc> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

---
## Sécurité et Contrôle d'Accès (RBAC)

### ServiceAccounts (Comptes de Service)
Identités pour les processus s'exécutant dans les Pods.
```bash
# Lister les ServiceAccounts dans un namespace
kubectl get serviceaccounts -n <nom-namespace>
# Alias: kubectl get sa -n <nom-namespace>

# Créer un ServiceAccount
kubectl create serviceaccount <nom-sa> -n <nom-namespace>

# Décrire un ServiceAccount
kubectl describe serviceaccount <nom-sa> -n <nom-namespace>
# Alias: kubectl describe sa <nom-sa> -n <nom-namespace>

# Supprimer un ServiceAccount
kubectl delete serviceaccount <nom-sa> -n <nom-namespace>
```

### RBAC (Roles, ClusterRoles, RoleBindings, ClusterRoleBindings)
Gestion des permissions basées sur les rôles.
```bash
# Lister les Roles (namespacés)
kubectl get roles -n <nom-namespace>
# Lister tous les Roles dans tous les namespaces (si droits suffisants)
kubectl get roles --all-namespaces

# Lister les ClusterRoles (globaux au cluster)
kubectl get clusterroles

# Lister les RoleBindings (namespacés)
kubectl get rolebindings -n <nom-namespace>
# Lister tous les RoleBindings dans tous les namespaces
kubectl get rolebindings --all-namespaces

# Lister les ClusterRoleBindings (globaux au cluster)
kubectl get clusterrolebindings

# Décrire un Role
kubectl describe role <nom-role> -n <nom-namespace>

# Décrire un ClusterRole
kubectl describe clusterrole <nom-clusterrole>

# Vérifier si une action est autorisée pour l'utilisateur courant
kubectl auth can-i <verbe> <ressource> --namespace <nom-namespace>
# Exemple: kubectl auth can-i list pods -n default

# Vérifier pour un utilisateur spécifique
kubectl auth can-i <verbe> <ressource> --as <nom-utilisateur> --namespace <nom-namespace>
# Exemple: kubectl auth can-i create deployments --as jane.doe@example.com -n web-dev

# Vérifier pour un ServiceAccount
kubectl auth can-i <verbe> <ressource> --as system:serviceaccount:<nom-namespace>:<nom-sa> --namespace <nom-namespace>
# Exemple: kubectl auth can-i get secrets --as system:serviceaccount:kube-system:replicaset-controller -n kube-system

# (Si le plugin kubectl-whoami est installé) Voir l'identité actuelle utilisée par kubectl
# kubectl whoami
# Sinon, pour voir le nom de l'utilisateur dans le contexte actuel:
kubectl config view -o jsonpath='{.contexts[?(@.name=="'$(kubectl config current-context)'")].context.user}'
```

---
## Utilitaires Généraux `kubectl`

### Changer le contexte actuel (si plusieurs clusters configurés)
```bash
# Lister les contextes disponibles
kubectl config get-contexts
# Afficher le contexte actuel
kubectl config current-context
# Changer de contexte
kubectl config use-context <nom-du-contexte>
```

### Appliquer une configuration YAML
```bash
kubectl apply -f <nom-fichier.yaml_ou_url>
# Pour appliquer tous les fichiers YAML dans un répertoire:
kubectl apply -f <nom-repertoire>/
```

### Supprimer des ressources à partir d'un fichier YAML
```bash
kubectl delete -f <nom-fichier.yaml>
```

### Éditer une ressource en direct
Ouvre l'éditeur par défaut (défini par $KUBE_EDITOR ou $EDITOR) pour modifier une ressource.
```bash
kubectl edit <type-ressource>/<nom-ressource> -n <nom-namespace>
# Exemple: kubectl edit deployment/my-deployment -n default
```

### Obtenir la version du client et du serveur
```bash
kubectl version
```

### Obtenir la configuration de l'API pour une ressource
Explique les champs d'une ressource.
```bash
kubectl explain <type-ressource>
# Exemple: kubectl explain pod
# Pour des champs spécifiques:
kubectl explain pod.spec.containers
```

---
*Fin de l'aide-mémoire.*