## Background

Topology manager in Kubernetes is used to implement the NUMA node affinity so that the performance/latency sensitive applications can have a better performance and low latency. 
 
Kubernetes topology manager can support 4 policies:
- none (default)
- best-effort
- restricted
- single-numa-node

The policy can be set by kubelet flag: --topology-manager-policy, In this document we will cover ‘single-numa-node’ only. 

In Kubernetes there are 2 topology manager scopes, either option can be selected at a time of the kubelet startup, with --topology-manager-scope flag.
- container (default)
- pod

In combination with ‘--topology-manager-scope=pod’ and ‘--topology-manager-policy=single-numa-node’, a pod should be rejected when a single NUMA node cannot meet the Pod resource requirement (i.g. A Pod needs more than 1 NUMA node).

In OCP 4.7.x, Topology Manager related settings (mainly kubelet flags) are supposed to be managed by PAO(Performance Addon Operator), unfortunately ‘topology-manager-scope’ is not exposed by PAO now. As a work around we need to create a KubeletConfig CR to combine the kubelet configuration generated by PAO and the ‘topologyManagerScope: pod’ settings so that the correct configuration can be provisioned to kubelet. 

## Environment
### Scripts
- Cluster setup:  https://github.com/borball/openshift-ansible-kvm 
- PAO: https://github.com/borball/k8s-tm-test/blob/master/pao-enabled.yaml 
- Node label: https://github.com/borball/k8s-tm-test/blob/master/node-label.yaml 
- Performance Profile: https://github.com/borball/k8s-tm-test/blob/master/performance-profile.yaml 

### Cluster
A OCP 4.7.20 cluster has been provisioned, the work nodes have the NUMA settings enabled, each NUMA node has 4vCPU available. In total each worker node has 8vCPU.
worker-0 has been labelled as ‘ht’ and considered as a ‘ht’ node while worker-1 is the standard node.

``` 
NAME       STATUS   ROLES             AGE   VERSION
master-0   Ready    master            18h   v1.20.0+01c9f3f
master-1   Ready    master            18h   v1.20.0+01c9f3f
master-2   Ready    master            18h   v1.20.0+01c9f3f
worker-0   Ready    ht,worker         18h   v1.20.0+01c9f3f
worker-1   Ready    standard,worker      18h   v1.20.0+01c9f3f
```

```
kni@bzhai-hive ~ $ ssh core@worker-0 lscpu 
Architecture:        x86_64
CPU op-mode(s):      32-bit, 64-bit
Byte Order:          Little Endian
CPU(s):              8
On-line CPU(s) list: 0-7
Thread(s) per core:  1
Core(s) per socket:  1
Socket(s):           8
NUMA node(s):        2
Vendor ID:           GenuineIntel
CPU family:          6
Model:               63
Model name:          Intel(R) Xeon(R) CPU E5-2630 v3 @ 2.40GHz
Stepping:            2
CPU MHz:             2394.454
BogoMIPS:            4788.90
Virtualization:      VT-x
Hypervisor vendor:   KVM
Virtualization type: full
L1d cache:           32K
L1i cache:           32K
L2 cache:            256K
L3 cache:            20480K
NUMA node0 CPU(s):   0-3
NUMA node1 CPU(s):   4-7

kni@bzhai-hive ~ $ ssh core@worker-1 lscpu  
Architecture:        x86_64
CPU op-mode(s):      32-bit, 64-bit
Byte Order:          Little Endian
CPU(s):              8
On-line CPU(s) list: 0-7
Thread(s) per core:  1
Core(s) per socket:  1
Socket(s):           8
NUMA node(s):        2
Vendor ID:           GenuineIntel
CPU family:          6
Model:               63
Model name:          Intel(R) Xeon(R) CPU E5-2630 v3 @ 2.40GHz
Stepping:            2
CPU MHz:             2394.454
BogoMIPS:            4788.90
Virtualization:      VT-x
Hypervisor vendor:   KVM
Virtualization type: full
L1d cache:           32K
L1i cache:           32K
L2 cache:            256K
L3 cache:            20480K
NUMA node0 CPU(s):   0-3
NUMA node1 CPU(s):   4-7
```

### PAO
Performance Addon Operator is enabled on the cluster. 

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
 name: openshift-performance-addon-operator
 labels:
   openshift.io/run-level: "1"
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
 name: openshift-performance-addon-operator
 namespace: openshift-performance-addon-operator
spec:
 targetNamespaces:
   - openshift-performance-addon-operator
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
 name: openshift-performance-addon-operator-subscription
 namespace: openshift-operators
spec:
 channel: "4.7"
 name: performance-addon-operator
 source: redhat-operators
 sourceNamespace: openshift-marketplace
```

### Performance Profile
Performance Profile ‘performance-single-numa-node’ is created and applied on worker-0 node with node selector node-role.kubernetes.io/ht, in this performance profile, the topologyPolicy is set as ‘single-numa-node’.

```yaml
---
apiVersion: performance.openshift.io/v2
kind: PerformanceProfile
metadata:
 name: performance-single-numa-node
spec:
 machineConfigPoolSelector:
   machineconfiguration.openshift.io/role: ht
 cpu:
   isolated: "1-3,5-7"
   reserved: "0,4"
 numa:
   topologyPolicy: single-numa-node
 nodeSelector:
   node-role.kubernetes.io/ht: ""

```
 
Once worker-0 is restarted, dump the kubelet configuration, 

```shell
oc get kubeletconfig performance-single-numa-node -o yaml > /tmp/kubeletconfig.yaml
```

```yaml
---
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
 name: performance-single-numa-node
spec:
 kubeletConfig:
   apiVersion: kubelet.config.k8s.io/v1beta1
   authentication:
     anonymous: {}
     webhook:
       cacheTTL: 0s
     x509: {}
   authorization:
     webhook:
       cacheAuthorizedTTL: 0s
       cacheUnauthorizedTTL: 0s
   cpuManagerPolicy: static
   cpuManagerReconcilePeriod: 5s
   evictionPressureTransitionPeriod: 0s
   fileCheckFrequency: 0s
   httpCheckFrequency: 0s
   imageMinimumGCAge: 0s
   kind: KubeletConfiguration
   kubeReserved:
     cpu: 1000m
     memory: 500Mi
   nodeStatusReportFrequency: 0s
   nodeStatusUpdateFrequency: 0s
   reservedSystemCPUs: 0,4
   runtimeRequestTimeout: 0s
   streamingConnectionIdleTimeout: 0s
   syncFrequency: 0s
   systemReserved:
     cpu: 1000m
     memory: 500Mi
   topologyManagerPolicy: single-numa-node
   volumeStatsAggPeriod: 0s
 machineConfigPoolSelector:
   matchLabels:
     machineconfiguration.openshift.io/role: ht

```

Add topologyManagerScope: pod in the yaml file above and rename the kubelet config name, final file is showing as below:

```yaml
---
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
 name: custom-config
spec:
 kubeletConfig:
   apiVersion: kubelet.config.k8s.io/v1beta1
   authentication:
     anonymous: {}
     webhook:
       cacheTTL: 0s
     x509: {}
   authorization:
     webhook:
       cacheAuthorizedTTL: 0s
       cacheUnauthorizedTTL: 0s
   cpuManagerPolicy: static
   cpuManagerReconcilePeriod: 5s
   evictionPressureTransitionPeriod: 0s
   fileCheckFrequency: 0s
   httpCheckFrequency: 0s
   imageMinimumGCAge: 0s
   kind: KubeletConfiguration
   kubeReserved:
     cpu: 1000m
     memory: 500Mi
   nodeStatusReportFrequency: 0s
   nodeStatusUpdateFrequency: 0s
   reservedSystemCPUs: 0,4
   runtimeRequestTimeout: 0s
   streamingConnectionIdleTimeout: 0s
   syncFrequency: 0s
   systemReserved:
     cpu: 1000m
     memory: 500Mi
   topologyManagerPolicy: single-numa-node
   topologyManagerScope: pod
   volumeStatsAggPeriod: 0s
 machineConfigPoolSelector:
   matchLabels:
     machineconfiguration.openshift.io/role: ht

```

Apply the kubelet config and worker-0 will reboot, then perform the test below.

## Test 
### topology-manager-scope=pod
Remember that worker-0 has ‘performance-single-numa-node’ enabled but worker-1 does not. 

#### Scenario 1 & 2

Description:

2 containers in the Pod running on the node which has ‘performance-single-numa-node’ enabled shall be pinned on the same NUMA node.

Script: 

https://github.com/borball/k8s-tm-test/blob/master/test-case-1%262.yaml 

Steps:

Create a deployment with 2 replicas, inside the deployment a Pod with 2 containers tm-1 and tm-2, tm-1 needs 1 vCPU and tm-2 needs 2vCPU.  The pods were scheduled on worker-0 and worker-1. 

```
kni@bzhai-hive ~/github/k8s-tm-test/deploy/test (master) $ oc get pods -o wide -n tm-test              
NAME                               READY   STATUS    RESTARTS   AGE     IP             NODE       NOMINATED NODE   READINESS GATES
guaranteed-pods-84895fc895-mqrmj   2/2     Running   0          4h23m   10.128.3.117   worker-1   <none>        <none>
guaranteed-pods-84895fc895-s8gnh   2/2     Running   0          4h23m   10.131.0.17    worker-0   <none>        <none>

```

Expected 1: 

2 containers in Pod running on worker-0 shall be pinned to the same NUMA node.

```
kni@bzhai-hive ~/github/k8s-tm-test/deploy/test (master) $ oc exec -it -n tm-test -c tm-1 guaranteed-pods-84895fc895-s8gnh -- cat /sys/fs/cgroup/cpuset/cpuset.cpus                                  
1
kni@bzhai-hive ~/github/k8s-tm-test/deploy/test (master) $ oc exec -it -n tm-test -c tm-2 guaranteed-pods-84895fc895-s8gnh -- cat /sys/fs/cgroup/cpuset/cpuset.cpus 
2-3
```

Expected 2:

2 containers in Pod running on worker-1 can be running on multiple NUMA nodes.

```
kni@bzhai-hive ~/github/k8s-tm-test/deploy/test (master) $ oc exec -it -n tm-test -c tm-1 guaranteed-pods-84895fc895-mqrmj -- cat /sys/fs/cgroup/cpuset/cpuset.cpus
0-7
kni@bzhai-hive ~/github/k8s-tm-test/deploy/test (master) $ oc exec -it -n tm-test -c tm-2 guaranteed-pods-84895fc895-mqrmj -- cat /sys/fs/cgroup/cpuset/cpuset.cpus 
0-7
```

#### Scenario 3

Description:

Not possible to schedule a Pod which requires numbers of CPU exceeding a single NUMA node’s capacity.

Script: 

https://github.com/borball/k8s-tm-test/blob/master/test-case-3.yaml 

Steps:

Create a deployment with 2 replicas, inside the deployment a Pod with 2 containers tm-1 and tm-2, tm-1 needs 2 vCPU and tm-2 needs 3 vCPU. 

Expected:

One Pod replica shall be scheduled on worker-1 node, but the other replica shall not be scheduled to worker-0 node, according to kubernetes document, it shall trigger a TopologyAffinityError error.


```
A replica was scheduled on worker-1 node. 
The other replica got TopologyAffinityError error.
guaranteed-pods-f66889c4c-slh5v   2/2     Running                 0          5m51s   10.128.3.222   worker-1   <none>           <none>
guaranteed-pods-f66889c4c-sr86k   0/2     TopologyAffinityError   0          4m38s   <none>         worker-0   <none>           <none>
guaranteed-pods-f66889c4c-srn5q   0/2     TopologyAffinityError   0          3m7s    <none>         worker-0   <none>           <none>
guaranteed-pods-f66889c4c-srsn7   0/2     TopologyAffinityError   0          5m25s   <none>         worker-0   <none>           <none>
guaranteed-pods-f66889c4c-st6ng   0/2     TopologyAffinityError   0          4m15s   <none>         worker-0   <none>           <none>

```

 
### topology-manager-scope=container

Revert the --topology-manager-scope to default value ‘container’ first. Keep ‘performance-single-numa-node’ enabled on worker-0 but not worker-1. 

#### Scenario 4

Description:

2 containers in the Pod running on the node which has ‘performance-single-numa-node’ enabled can be running on multiple NUMA nodes.


Script: 

https://github.com/borball/k8s-tm-test/blob/master/test-case-4.yaml 

Steps:

Create a deployment with 2 replicas, inside the deployment a Pod with 2 containers tm-1 and tm-2, tm-1 needs 2 vCPU and tm-2 needs 3vCPU. 

Expected:

The 2 replicas shall be scheduled on 2 worker nodes. Compared with Scenario 3, it can be scheduled. 
The 2 containers in the pod can be pinned to different NUMA nodes.

```
kni@bzhai-hive ~/github/k8s-tm-test/deploy/test (master) $ oc get pods -n tm-test  -o wide
NAME                              READY   STATUS    RESTARTS   AGE   IP             NODE       NOMINATED NODE   READINESS GATES
guaranteed-pods-f66889c4c-qv9rw   2/2     Running   0          9s    10.131.0.6     worker-0   <none>           <none>
guaranteed-pods-f66889c4c-slh5v   2/2     Running   0          40m   10.128.3.222   worker-1   <none>           <none>
```

Containers in pod running on worker-0, were pinned on different NUMA nodes:

```
kni@bzhai-hive ~/github/k8s-tm-test/deploy/test (master) $ oc exec -it -n tm-test -c tm-1 guaranteed-pods-f66889c4c-qv9rw -- cat /sys/fs/cgroup/cpuset/cpuset.cpus
1-2
kni@bzhai-hive ~/github/k8s-tm-test/deploy/test (master) $ oc exec -it -n tm-test -c tm-2 guaranteed-pods-f66889c4c-qv9rw -- cat /sys/fs/cgroup/cpuset/cpuset.cpus 
5-7
```

Containers in pod running on worker-1, were not pinned on any specific NUMA node:

```shell
kni@bzhai-hive ~/github/k8s-tm-test/deploy/test (master) $ oc exec -it -n tm-test -c tm-1 guaranteed-pods-f66889c4c-slh5v -- cat /sys/fs/cgroup/cpuset/cpuset.cpus                                 
0-7
kni@bzhai-hive ~/github/k8s-tm-test/deploy/test (master) $ oc exec -it -n tm-test -c tm-2 guaranteed-pods-f66889c4c-slh5v -- cat /sys/fs/cgroup/cpuset/cpuset.cpus 
0-7
```


#### Scenario 5
Description:

Pod cannot be scheduled to the node which has ‘performance-single-numa-node’ enabled if any of the containers inside the pod requires that the number of vCPU exceeds the single NUMA node’s capacity.

Script: 

https://github.com/borball/k8s-tm-test/blob/master/test-case-5.yaml 

Steps:

Create a deployment with 2 replicas, inside the deployment a Pod with 2 containers tm-1 and tm-2, tm-1 needs 1 vCPU and tm-2 needs 5vCPU. 

Expected:

One replica shall be scheduled on worker-1 node. 
No replica can be scheduled to worker-0 node due to TopologyAffinityError.

```
One replica was scheduled to worker-1 node.
guaranteed-pods-785b596cbc-gktx5   2/2     Running                 0          14s   10.128.3.235   worker-1   <none>           <none>
```

The other replica on worker-0 got TopologyAffinityError

```
guaranteed-pods-785b596cbc-gltpz   0/2     TopologyAffinityError   0          6s    <none>         worker-0   <none>           <none>
guaranteed-pods-785b596cbc-gw7rq   0/2     TopologyAffinityError   0          10s   <none>         worker-0   <none>           <none>
guaranteed-pods-785b596cbc-gzgmm   0/2     TopologyAffinityError   0          8s    <none>         worker-0   <none>           <none>
guaranteed-pods-785b596cbc-h6msz   0/2     TopologyAffinityError   0          10s   <none>         worker-0   <none>           <none>
guaranteed-pods-785b596cbc-h9mpd   0/2     TopologyAffinityError   0          12s   <none>         worker-0   <none>           <none>
```

## References

- [1]: Topology Manager: https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/ 
- [2]: openshift-ansible-kvm: https://github.com/borball/openshift-ansible-kvm 
- [3]: k8s-tm-test: https://github.com/borball/k8s-tm-test  



