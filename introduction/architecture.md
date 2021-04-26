## Introduction
Kubernetes est un projet Open Source créé par Google en 2015. Il permet d’automatiser le déploiement et la gestion d’applications multi-container à l’échelle. Il s’agit d’un système permettant d’exécuter et de coordonner des applications containerisées sur un cluster de machines. en utilisant des méthodes de prédictibilité, de scalabilité et de haute disponibilité.

## Architecture de kubernetes
Le terme « cluster » désigne un déploiement fonctionnel de Kubernetes. Un cluster Kubernetes comprend deux principaux composants : **le control plan** et **les workers nodes**.
![](../images/arch.png)

### Control plan
Commençons par le centre névralgique de notre cluster Kubernetes : le plan de contrôle. Il contient les composants Kubernetes qui contrôlent le cluster, ainsi que des données sur l'état et la configuration du cluster. Ces principaux composants de Kubernetes s'assurent que suffisamment de conteneurs peuvent fonctionner avec les ressources nécessaires. 

Le plan de contrôle est en contact permanent avec vos machines de calcul. Vous avez configuré votre cluster pour qu'il fonctionne d'une certaine manière. Le plan de contrôle s'assure que votre configuration est respectée.

* **kube-apiserver**
Vous avez besoin d'interagir avec votre cluster Kubernetes ? C'est à cela que sert l'API. L'API de Kubernetes est la partie frontale du plan de contrôle. Elle prend en charge les demandes internes et externes. Le serveur d'API détermine si une demande est valide ou non et la traite, le cas échéant. Vous pouvez accéder à l'API avec des appels REST, à l'aide de l'interface en ligne de commande kubectl ou d'autres outils en ligne de commande tels que kubeadm.

* **kube-scheduler**
Votre cluster est-il en bonne santé ? Est-il possible d'intégrer de nouveaux conteneurs si besoin ? Ce sont les questions auxquelles doit répondre le planificateur Kubernetes.
En plus de l'intégrité du cluster, le planificateur doit prendre en compte les besoins en ressources (par exemple, processeur ou mémoire) d'un pod. Il planifie ensuite l'attribution du pod au nœud de calcul adéquat.

* **kube-controller-manager**
Les contrôleurs assurent l'exécution du cluster, tandis que le gestionnaire de contrôleur Kubernetes regroupe plusieurs fonctions de contrôleur. Un contrôleur se réfère au planificateur pour s'assurer qu'un nombre suffisant de pods est exécuté. Si un pod est défaillant, un autre contrôleur le remarque et réagit. Un contrôleur connecte les services aux pods afin que les demandes soient acheminées jusqu'aux points de terminaison appropriés. D'autres contrôleurs permettent de créer des comptes et des jetons d'accès aux API.

* **etcd**
« etcd » est une base de données clé-valeur qui comprend les données de configuration et les informations sur l'état du cluster. Distribuée et résistante aux pannes, la base de données etcd constitue la référence unique concernant votre cluster.
