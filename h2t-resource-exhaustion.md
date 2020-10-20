#  Resource-Exhaustion
##  Introduction
As a developer or an Ops, you occasionally get into this situation where pods or API calls fail and resume working seemingly randomly.
You might notice there are a lots of "Evicted" and/or "Pending" pods, or high restart counts on pods, or your `oc` commands take longer to respond or even fail sometimes, and everything seems to point to the system is busy, under heavy load too much stuff on it.
This document explains how to verify, diagnose, and offer what could be done about this "resource exhaustion" 
situation on an OpenShift cluster.
## How to verify this situation
Resource exhaustion is the condition that happens when the resources required to execute an action are entirely or nearly expended, preventing that action from occurring.The most common outcome of resource exhaustion is denial of service.
`oc adm top nodes` is one way to see how tight CPU and Memory resources are for the cluster nodes.
#### Example output below :
You will get to know about resource exhaustion condition using this command
```
$ oc adm top nodes
 NAME           CPU(cores)          CPU%               MEMORY(bytes)                MEMORY%\n
 node-1            297m              29%                4263Mi                        55%    
 node-0            55m               5%                 1201Mi                        15%    
 infra-1           85m               8%                 1319Mi                        17%    
 infra-0           182m              18%                2524Mi                        32%    
 master-0          178m              8%                 2584Mi                        16%    
```
Other indications that one could suspect of this situation are failing pods, especially in "Evicted" state.
Use `oc get pods -A | egrep -v 'Completed|Running'` and see what and how many are failing. 
Sometimes, "Pending" state could be attributed to resource exhaustion, as those pods are having hard time finding a free node to run on.

## Basics of scheduling pods
When you specify a Pod, you can optionally specify how much of each resource a Container needs. 
The most common resources to specify are CPU and memory . 
CPU is specified in units of Kubernetes CPUs, Memory is specified in units of bytes. 
When you specify the resource request for Containers in a Pod, the scheduler uses this information to decide which node to place the Pod on. 

When you specify a resource limit for a Container, the kubelet enforces those limits so that the running container is not allowed to use more of that resource than the limit you set. 
The kubelet also reserves at least the request amount of that system resource specifically for that container to use.

For example, if you set a memory request of 256 MiB for a container, and that container is in a Pod scheduled to a Node with 8GiB of memory and no other Pods, then the container can try to use more RAM.

If you set a memory limit of 4GiB for that Container, the kubelet (and container runtime) enforce the limit.
The runtime prevents the container from using more than the configured resource limit.

For example when a process in the container tries to consume more than the allowed amount of memory, the system kernel terminates the process that attempted the allocation, with an out of memory (OOM) error.
In order to monitor this, you always have to look at the use of memory compared to the limit.
Percentage of the node memory used by a pod is usually a bad indicator as it gives no indication on how close to the limit the memory usage is. 
In Kubernetes, limits are applied to containers, not pods, so monitor the memory usage of a container vs. the limit of that container.
<img width="1416" alt="Import dashboard from file or Grafana com" 
src="https://478h5m1yrfsa3bbe262u7muv-wpengine.netdna-ssl.com/wp-content/uploads/image3-1.png">
## Pod Eviction
Pods could be evicted as part of kubelet's attempt to keep the node from going down completely. See sample messages below. 
Example : 
```
 Reason :  Evicted
 Warning : Evicted    125m  kubelet, worker-0.shivani4-9998.ocp-44.com  
 The node had condition:        [DiskPressure].
```
```
 Reason :    Evicted
 Warning : Evicted  126m  kubelet, worker-0.shivani4-9998.ocp-44.com  
 The node was low on resource:   ephemeral-storage.
 Container certified-operators was using 44156Ki,which exceeds its request of 0.
```
Kubernetes has a way of determining which pods to kill and evict in what order and trying to mitigate similar resource tightness in the future. 
It's built around the idea of garbage collection and QoS (Quality of Service) classes.
See these pages for more details;<br/><https://sysdig.com/blog/kubernetes-pod-evicted/>
<https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/>

## Signs of nodes running out of memory
The kubelet can support the ability to trigger eviction decisions on the signals described below.
The value of each signal is described in the description column based on the kubelet summary API.

| Eviction Signal  | Description |
| ------------- | ------------- |
| memory.available   | memory.available := node.status.capacity[memory] - node.stats.memory.workingSet |
| nodefs.available  | nodefs.available |
| nodefs.inodesFree   | nodefs.inodesFree := node.stats.fs.inodesFree  |
| imagefs.available  | imagefs.available := node.stats.runtime.imagefs.available  |
| imagefs.inodesFree | magefs.inodesFree := node.stats.runtime.imagefs.inodesFree  |

This link explains further details about eviction signals and how to configure out of resource handling with kubelet.
<https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/#eviction-signals>

## When does oc describe node show MemoryPressure=true?
oc describe node show MemoryPressure=true when Kubelet has insufficient memory.
When a node in a Kubernetes cluster is running out of memory or disk, it activates a flag signaling that it is under pressure.
This blocks any new allocation in the node and starts the eviction process.At that moment, kubelet starts to reclaim resources, killing containers and declaring pods as failed until the resource usage is under the eviction threshold again.

## Ephemeral storage
"Ephemeral" means that there is no long-term guarantee about durability.
Kubernetes supports two ways to configure local ephemeral storage on a node:

- Single file system 
- Two file system

#### Setting requests and limits for local ephemeral storage
You can use ephemeral-storage for managing local ephemeral storage. Each Container of a Pod can specify one or more of the following:
   1. spec.containers[].resources.limits.ephemeral-storage
   2. spec.containers[].resources.requests.ephemeral-storage
   
To know more in detail about ephemeral storage refer link provided below<br/>
<https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#:~:text=Ephemeral><br/>
The kubelet supports different ways to measure Pod storage use:
1. Periodic Scanning :<br/>
The kubelet performs regular, schedules checks that scan each emptyDir volume, container log directory, and writeable container layer.The scan measures how much space is used.

2. Filesystem project quota :<br/>
Project quotas are an operating-system level feature for managing storage use on filesystems. With Kubernetes, you can enable project quotas for monitoring storage use. 
Make sure that the filesystem backing the emptyDir volumes, on the nodeprovides project quota support. 
For example, XFS and ext4fs offer project quotas.

## Restart count
Restart count represents the number of times the container inside a pod has been restarted, it is based on the number of dead containers that have not yet been removed. Note that this is calculated from dead containers. 
command to get restart count  :   kubectl describe pod nginx | grep -i "Restart"

## Details about resourse allocation
As mentioned earlier, a pod can specify how much system resources it needs ("request") and/or doesn't want to consume more than ("limit"). 
Also, a namespace can impose defaults on all pods running in it if they don't explicitly specify their resource requests/limits. 
Kubernetes has guided tasks that one can follow along to understand how these settings work:
#### CPU
To specify a CPU request and limit for a container,include following fields in the container resource manifest<br/>
Example:
```
resources:
  limits:        
    cpu: "1"     
  requests: 
    cpu: "0.5"
```
To know more in detail about allocation of cpu resources to containers and pods refer link provided below
<https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/>
#### Memory
To specify a Memory request and limit for a container, include following fields in the container resource manifest<br/>
Example:
```
resources:
   limits:       
     memory: "200Mi"    
   requests:       
     memory: "100Mi"
```
To know more in detail about allocation of memory resource to containers and pods refer link provided below
<https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/>

#### Namespace defaults
If a Container is created in a namespace that has a default memory limit, and the Container does not specify its own memory limit, then the Container is assigned the default memory limit.<br/>
To know more about configuring default memory requests and limits for a namespace refer link provided below
<https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/>

#### At node level
One can also specify how much resource Kubernetes can use and how much OS/system keeps for its 
stability. This helps avoid overcommitment and flapping. 
You can control how much of a node resource is made available for scheduling in order to allow the scheduler to fully allocate a node and to prevent evictions
This page describes how this all works in OpenShift clusters in more details:<br/> 
<https://docs.openshift.com/container-platform/4.5/nodes/nodes/nodes-nodes-resources-configuring.html> <br/>

- Overcommitment:<br/> 
  Scheduling is based on resources requested, while quota and hard limits refer to resource limits, which can 
  be set higher than requested resources.
  The difference between request and limit determines the level of overcommit .
  The Cluster ResourceOverride Operator is an admission webhook that allows you to control the level of overcommit and manage container density across all the nodes in your cluster.<br/>
  To know in detail about cluster resource override operator refer link provided below<br/>
<https://docs.openshift.com/container-platform/4.5/post_installation_configuration/node-tasks.html#nodes-cluster-overcommit-resource-requests_post-install-node-tasks><br/>

- Flapping:<br/> 
  If the node ready status has an altering behaviour , the node is flapping. 
  Flapping indicates that is kubelet is not ready.
  To know more in detail about causes of flapping , refer the link provided below<br/>
  <https://docs.sysdig.com/en/nodes-data.html>

## What to do to Mitigate
- The kubelet needs to preserve node stability when available compute resources are low. 
  This is especially important when dealing with incompressible compute resources, such as memory or disk space.If such resources are exhausted, nodes become unstable.

- The kubelet can proactively monitor for and prevent total starvation of a compute resource. 
  In those cases, the kubelet can reclaim the starved resource by proactively failing one or more Pods.
  When the kubelet fails a Pod, it terminates all of its containers and transitions its PodPhase to Failed.
  If the evicted Pod is managed by a Deployment, the Deployment will create another Pod to be scheduled 
  by Kubernetes.

- To provide more reliable scheduling and minimize node resource overcommitment, each node can reserve a portion of its resources for use by all underlying node components. 
  CPU and memory resources reserved for node components in OpenShift Container Platform are based on two node settings:
     - Kube-reserved
     - System-reserved
  
   If a flag is not set, it defaults to 0. If none of the flags are set, the allocated resource is set to the node’s capacity as it was before 
   the  introduction of allocatable resources. 
   For further reading on managing node resources see<br/>
   <https://docs.openshift.com/container-platform/4.5/nodes/nodes/nodes-nodes-resources-configuring.htm>

- The scheduler attempts to optimize the compute resource use across all nodes in your cluster. 
  It places Pods onto specific nodes, taking the Pods' compute resource requests and nodes' available capacity into consideration.
  OpenShift Container Platform administrators can control the level of overcommit and manage container density on nodes. 
  You can configure cluster-level overcommit using the Cluster Resource Override Operator.
  To know in detail about cluster resource override operator see<br/>
  <https://docs.openshift.com/container-platform/4.5/nodes/clusters/nodes-cluster-overcommit.html>


