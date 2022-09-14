## Check the K8S cluster health
```console
curl -k 'https://localhost:6443/readyz?verbose&exclude=etcd'
or
kubectl get --raw='/readyz?verbose'
```
Output
```console
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
```console
kubectl get componentstatuses
```
Output
```console
NAME                  STATUS    MESSAGE             ERROR
scheduler             Healthy   ok
controller-manager    Healthy   ok
etcd-0                Healthy   {"health": "true"}
```
## Listing Kubernetes Worker Nodes
```console
kubectl get nodes
```
Output
```console
NAME        STATUS        AGE   VERSION
kubernetes  Ready,master  45d   v1.12.1
node-1      Ready         45d   v1.12.1
node-2      Ready         45d   v1.12.1
node-3      Ready         45d   v1.12.1
```
## Get more information about a node
```console
kubectl describe nodes node-1
```
Output
```console
Name:     node-1
Role:
Labels:   beta.kubernetes.io/arch=arm     # running an ARM processor
          beta.kubernetes.io/os=linux     # running the Linux OS
          kubernetes.io/hostname=node-1 
          
Conditions:
  Type            Status      LastHeartbeatTime       Reason                      Message
  ----            ------      -----------------       ------                      -------
  OutOfDisk       False       Sun, 05 Feb 2017…       KubeletHasSufficientDisk    kubelet…    # show that the node has sufficient disk
  MemoryPressure  False       Sun, 05 Feb 2017…       KubeletHasSufficientMemory  kubelet…    # show that the node has sufficient memory
  DiskPressure    False       Sun, 05 Feb 2017…       KubeletHasNoDiskPressure    kubelet…
  Ready           True        Sun, 05 Feb 2017…       KubeletReady                kubelet…

Capacity:
  alpha.kubernetes.io/nvidia-gpu: 0
  cpu:                            4
  memory:                         882636Ki
  pods:                           110
Allocatable:
  alpha.kubernetes.io/nvidia-gpu: 0
  cpu:                            4
  memory:                         882636Ki
  pods:                           110
  
System Info:
  Machine ID:                     9122895d0d494e3f97dda1e8f969c85c
  System UUID:                    A7DBF2CE-DB1E-E34A-969A-3355C36A2149
  Boot ID:                        ba53d5ee-27d2-4b6a-8f19-e5f702993ec6
  Kernel Version:                 4.15.0-1037-azure
  OS Image:                       Ubuntu 16.04.5 LTS
  Operating System:               linux
  Architecture:                   amd64
  Container Runtime Version:      docker://3.0.4
  Kubelet Version:                v1.12.6
  Kube-Proxy Version:             v1.12.6
PodCIDR:                          10.244.1.0/24

Non-terminated Pods: (3 in total)
  Namespace     Name          CPU Requests    CPU Limits    Memory Requests     Memory Limits
  ---------     ----          ------------    ----------    ---------------     -------------
  kube-system   kube-dns...   260m (6%)       0 (0%)        140Mi (16%)         220Mi (25%)
  kube-system   kube-fla...   0 (0%)          0 (0%)        0 (0%)              0 (0%)
  kube-system   kube-pro...   0 (0%)          0 (0%)        0 (0%)              0 (0%)
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.
  CPU Requests  CPU Limits    Memory Requests Memory Limits
  ------------  ----------    --------------- -------------
  260m (6%)     0 (0%)        140Mi (16%)     220Mi (25%)
No events.
```
