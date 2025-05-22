# DaemonSets Kubernetes

Les DaemonSets sont une fonctionnalité de Kubernetes qui garantit qu'une copie d'un Pod s'exécute sur tous les nœuds (ou un sous-ensemble de nœuds) d'un cluster. Ils sont essentiels pour déployer des démons (daemons) à l'échelle du cluster qui effectuent des tâches de maintenance ou de support sur chaque nœud.

## Introduction aux DaemonSets

### Que sont-ils ?

Un **DaemonSet** assure qu'une instance d'un Pod spécifique est programmée sur chaque nœud éligible du cluster. Dès qu'un nœud rejoint le cluster et correspond aux critères du DaemonSet (par exemple, via un `nodeSelector`), le contrôleur DaemonSet y déploie automatiquement le Pod. Inversement, si un nœud quitte le cluster ou ne correspond plus aux critères, le Pod géré par le DaemonSet est supprimé de ce nœud.

### Pourquoi sont-ils nécessaires ?

Les DaemonSets sont utilisés pour des tâches qui doivent s'exécuter sur chaque nœud (ou sur un ensemble spécifique de nœuds) afin de fournir des services de niveau système ou d'infrastructure. Ils sont idéaux pour les agents ou les processus qui doivent être présents localement sur chaque machine du cluster.

### Caractéristiques Clés

*   **Un Pod par Nœud Correspondant :** Un DaemonSet garantit qu'au plus un Pod qu'il gère s'exécute sur chaque nœud qui satisfait à ses contraintes de planification.
*   **Planification Automatique sur les Nouveaux Nœuds :** Lorsqu'un nouveau nœud est ajouté au cluster et qu'il correspond aux sélecteurs du DaemonSet, un Pod est automatiquement créé sur ce nœud.
*   **Suppression Automatique des Pods :** Si un nœud est retiré du cluster, le Pod du DaemonSet est automatiquement supprimé. De même, si les labels d'un nœud changent et qu'il ne correspond plus au `nodeSelector` du DaemonSet, le Pod est retiré.
*   **Pas de `replicas` :** Contrairement aux Deployments ou StatefulSets, un DaemonSet n'a pas de champ `spec.replicas`. Le nombre de Pods est dynamiquement déterminé par le nombre de nœuds éligibles.

## Cas d'Usage

Les DaemonSets sont couramment utilisés pour :

*   **Agents de Collecte de Logs :**
    *   Exemples : Fluentd, Logstash, Filebeat. Ces agents collectent les logs des applications s'exécutant sur le nœud et les envoient à un système de logging centralisé.
*   **Agents de Monitoring de Nœud :**
    *   Exemples : Prometheus Node Exporter, Datadog Agent, New Relic Agent. Ces agents collectent des métriques sur l'état du nœud (CPU, mémoire, disque, réseau) et les exposent à un système de monitoring.
*   **Démons de Stockage de Cluster :**
    *   Exemples : GlusterFS, Ceph. Ces systèmes de stockage distribué peuvent nécessiter un agent s'exécutant sur chaque nœud pour fournir l'accès au stockage.
*   **Plugins Réseau (CNI - Container Network Interface) :**
    *   Exemples : Calico node, Cilium, Weave Net. Beaucoup de plugins réseau déploient un agent sur chaque nœud pour configurer le réseau des Pods et gérer les politiques réseau.
*   **Proxys de Nœud :**
    *   Par exemple, `kube-proxy` lui-même est souvent exécuté en tant que DaemonSet pour gérer les règles de routage réseau sur chaque nœud.

## Comment Fonctionnent les DaemonSets

*   **`spec.selector` :** Comme les autres contrôleurs (Deployments, StatefulSets), un DaemonSet utilise `spec.selector.matchLabels` pour identifier les Pods qu'il doit gérer. Ce label doit correspondre à ceux définis dans `spec.template.metadata.labels`.
*   **`spec.template` :** Définit le modèle du Pod qui sera déployé sur chaque nœud. Cela inclut l'image du conteneur, les volumes, les ports, etc.
*   **Planification sur les Nœuds :** Le contrôleur DaemonSet surveille les nœuds du cluster. Pour chaque nœud, il vérifie s'il existe déjà un Pod géré par le DaemonSet. Si ce n'est pas le cas, et si le nœud est éligible, il crée le Pod à partir du `spec.template`.

### Sélecteur de Nœud (`spec.template.spec.nodeSelector`)

Vous pouvez restreindre les nœuds sur lesquels les Pods du DaemonSet seront déployés en utilisant un `nodeSelector` dans le `spec.template.spec` du Pod. Seuls les nœuds ayant tous les labels spécifiés dans le `nodeSelector` exécuteront le Pod du DaemonSet.

```yaml
# ... (partie metadata et spec.selector du DaemonSet)
      template:
        metadata:
          labels:
            app: mon-agent-special
        spec:
          nodeSelector:
            disktype: ssd # Le Pod ne sera déployé que sur les nœuds avec le label 'disktype=ssd'
            region: "europe-west1"
# ... (partie containers)
```

### Tolérances (`spec.template.spec.tolerations`)

Les Pods de DaemonSet peuvent avoir besoin de s'exécuter sur des nœuds "taintés" (marqués pour repousser les Pods qui n'ont pas de tolérance explicite pour cette "taint"). Par exemple, les nœuds maîtres (control plane) ont souvent des taints. Si votre DaemonSet doit s'exécuter sur ces nœuds (par exemple, un agent de sécurité), vous devez ajouter les tolérances appropriées au template du Pod.

```yaml
# ... (partie metadata et spec.selector du DaemonSet)
      template:
        metadata:
          labels:
            app: mon-agent-critique
        spec:
          tolerations:
          - key: "node-role.kubernetes.io/master" # Tolère le taint typique des nœuds maîtres
            effect: "NoSchedule"
          - key: "special-purpose"
            operator: "Exists" # Tolère n'importe quel taint avec la clé "special-purpose"
# ... (partie containers)
```
**Attention :** Déployer des DaemonSets sur les nœuds maîtres doit être fait avec prudence pour ne pas impacter la stabilité du control plane.

## Exemple de DaemonSet YAML

Voici un exemple de DaemonSet qui pourrait simuler un agent de collecte de logs simple (comme Fluentd).

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-logging-agent
  labels:
    app: fluentd-logging
spec:
  selector:
    matchLabels:
      app: fluentd-logging # Doit correspondre à .spec.template.metadata.labels
  template:
    metadata:
      labels:
        app: fluentd-logging # Doit correspondre à .spec.selector.matchLabels
    spec:
      # Optionnel : Restreindre aux nœuds avec un label spécifique
      # nodeSelector:
      #   kubernetes.io/os: linux
      tolerations:
      # Cette tolérance est souvent nécessaire pour les agents de logging/monitoring
      # qui doivent accéder aux logs/métriques de tous les composants du nœud.
      - operator: "Exists" 
        effect: "NoSchedule" # Permet de s'exécuter même si le nœud a des taints NoSchedule
      - operator: "Exists"
        effect: "NoExecute"  # Permet de s'exécuter même si le nœud a des taints NoExecute (avec précaution)
      containers:
      - name: fluentd-agent
        image: fluent/fluentd:v1.14-debian-1 # Une image Fluentd générique
        # Configuration typique pour un agent de logs :
        # - Monter les répertoires de logs du nœud hôte.
        # - Configurer Fluentd pour lire ces logs et les envoyer ailleurs.
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log         # Accès aux logs généraux du système
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers # Accès aux logs des conteneurs Docker
          readOnly: true
      volumes:
      # Utilisation de hostPath pour accéder aux fichiers/répertoires du nœud hôte.
      # ATTENTION : hostPath peut présenter des risques de sécurité et lier étroitement
      # le Pod à la configuration du nœud. À utiliser avec discernement.
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      terminationGracePeriodSeconds: 30 # Temps pour que le Pod s'arrête proprement
```

**Explication du `hostPath` :**
*   Un volume `hostPath` monte un fichier ou un répertoire du système de fichiers du nœud hôte directement dans le Pod.
*   **Cas d'usage pour DaemonSets :** C'est très courant pour les agents de logging (pour lire `/var/log`) ou de monitoring.
*   **Implications et Risques :**
    *   **Sécurité :** Si un Pod `hostPath` est compromis, il peut potentiellement accéder ou modifier des fichiers sensibles sur le nœud hôte. Utilisez `readOnly: true` autant que possible pour les montages.
    *   **Portabilité :** La configuration du Pod devient dépendante de la structure du système de fichiers du nœud. Si le chemin `/var/log` n'existe pas sur un nœud, le Pod échouera.
    *   **Privilèges :** L'accès à certains chemins peut nécessiter des privilèges élevés pour le Pod (par exemple, via un `securityContext`).

## Mise à Jour des DaemonSets

Les DaemonSets supportent deux stratégies de mise à jour, configurées via `spec.updateStrategy.type` :

*   **`RollingUpdate` (Par Défaut) :**
    *   Lorsque vous modifiez le `spec.template` du DaemonSet (par exemple, l'image du conteneur), les Pods sont mis à jour de manière progressive, un nœud à la fois.
    *   L'ancien Pod sur un nœud est supprimé, et le nouveau Pod est créé. Le DaemonSet attend que le nouveau Pod soit `Running` et `Ready` avant de passer au nœud suivant.
    *   Vous pouvez affiner le comportement avec `spec.updateStrategy.rollingUpdate.maxUnavailable` (nombre ou pourcentage de Pods qui peuvent être indisponibles pendant la mise à jour, par défaut 1) et `maxSurge` (nombre ou pourcentage de Pods qui peuvent être créés au-delà du nombre désiré, non applicable aux DaemonSets car ils ne peuvent pas "surge" au-delà du nombre de nœuds). Pour les DaemonSets, `maxSurge` est ignoré et `maxUnavailable` contrôle le nombre de nœuds mis à jour en parallèle.
*   **`OnDelete` :**
    *   Avec cette stratégie, le contrôleur DaemonSet ne met pas automatiquement à jour les Pods lorsque le `spec.template` change.
    *   Pour appliquer la nouvelle configuration, vous devez supprimer manuellement les Pods existants sur chaque nœud. Le DaemonSet les recréera alors avec la nouvelle version. Cela vous donne un contrôle total sur le moment de la mise à jour de chaque Pod/nœud.

## Communication avec les Pods d'un DaemonSet

La manière d'interagir avec les Pods d'un DaemonSet dépend de la nature du démon :

*   **Aucun Service Externe Requis (Cas le Plus Courant) :**
    *   Souvent, les agents DaemonSet (comme les collecteurs de logs ou les moniteurs de nœuds) n'ont pas besoin d'être contactés *depuis l'extérieur* du nœud. Ils s'exécutent sur le nœud, collectent des informations localement, et les poussent vers un service centralisé (par exemple, un cluster Elasticsearch, une base de données Prometheus).
*   **`hostPort` :**
    *   Si le démon sur le nœud doit exposer un port directement sur l'adresse IP du nœud, vous pouvez utiliser `spec.template.spec.containers[].ports[].hostPort`.
    *   **Avantage :** Le service est accessible via `IP_DU_NOEUD:hostPort`.
    *   **Inconvénients :**
        *   **Conflits de Ports :** Vous devez vous assurer que le `hostPort` n'est pas déjà utilisé sur le nœud par un autre processus, sinon le Pod du DaemonSet ne pourra pas démarrer (ou un seul des Pods utilisant ce port sur le nœud pourra démarrer).
        *   **Sécurité :** Exposer des ports directement sur l'IP du nœud peut augmenter la surface d'attaque.
*   **Service Headless :**
    *   Si vous avez besoin de découvrir les adresses IP des Pods du DaemonSet pour une communication directe (par exemple, un autre service a besoin de parler à l'agent spécifique sur chaque nœud), vous pouvez créer un Service "headless" (avec `clusterIP: None`) qui sélectionne les Pods du DaemonSet via leurs labels. Cela créera des enregistrements DNS pour chaque Pod.
*   **Service avec ClusterIP (Moins Courant pour les DaemonSets) :**
    *   Si vous avez un cas d'usage où vous voulez un point d'entrée unique (ClusterIP) qui répartit la charge vers *un* des Pods du DaemonSet (par exemple, pour obtenir l'état d'un agent aléatoire), vous pourriez utiliser un Service standard. Cependant, cela va à l'encontre du pattern typique d'un DaemonSet où chaque Pod est spécifique à son nœud.

En général, la communication avec un DaemonSet est moins axée sur l'exposition de services entrants et plus sur le travail que le démon effectue localement sur le nœud.

---

> Next: [Jobs](./jobs.md)

> [cheat sheet](../useful.md)
