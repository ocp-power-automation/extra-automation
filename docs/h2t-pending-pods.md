# How to Troubleshoot Pending Pods
=======================================================================

A pod stuck in pending state cannot be scheduled onto a node until the problem is identified and resolved.
It could be caused due to a handful of reasons ranging from resource issues to the rules applied to the pods and nodes.
For this, a few commands could be executed to narrow down the possibilities and focus on what could have been the exact issue causing it.
A few initial diagnostic commands mentioned below could be a starting step to help troubleshoot the pods stuck in pending state.

## Initial diagnosis:

* `oc adm top nodes`  
This command will show how CPU and Memory are tight across nodes in the cluster.
If you see high percentage number(s), the node(s) is/are running out of that resource, and possibly preventing new pods from being scheduled and starting.
Check reason 1 below.
* `oc describe node node-name`  
This is a good follow-up command to above, and will provide detailed information of a specific node. Checking the information under 'Allocated resources' or 'Conditions' could help in understanding if the problem is inclining towards insufficient resources(memory or CPU) on the node and in determining the resources to be allocated to the pod accordingly
* `oc get pods -A | grep Pending`  
This command will show pods that are in Pending state in the cluster.
If you see only a handful, you are in luck and proceed with below oc describe pod command for those to find out more details and which subsection of this document to go.
If you see many more Pending pods, it may be quicker to suspect more systemic issues, than individual pod issues, although it may still come down to it.
* `oc describe pod pod-name`  
The 'Events' section of the output would provide the current status of the pod and also the reason for it being in that state.
If it is stuck in pending state, based on the reason, that particular issue can be resolved by following the steps below

## 1. Reason: Insufficient resources on the nodes available
--------------------------------------------------------------
            Error ==>  `0/5 nodes are available: 1 Insufficient cpu`

* Pod went to pending state if you  don't have enough resources: You may have exhausted the supply of CPU or Memory in your cluster.

### How can this be resolved?
Viewing the CPU and memory usage statistics on the nodes could help in better allocation of a pod to the desired node. This can be checked by `oc adm top nodes` 
Note: <cluster-reader> permission is required to view the usage statistics
Here are some example command lines that extract just the necessary information.    

`oc get nodes -o yaml | egrep '\sname:|cpu:|memory:'`

```
name: master-0.202-8880.example.com
      cpu: 7500m
      memory: 16101184Ki
      cpu: "8"
      memory: 16715584Ki
```

`oc get nodes -o json | jq '.items[] | {name: .metadata.name, cap: .status.capacity'}`

```
{
  "name": "master-0.202-8880.example.com",
  "cap": {
    "cpu": "8",
    "ephemeral-storage": "209290220Ki",
    "hugepages-16Gi": "0",
    "hugepages-16Mi": "0",
    "memory": "16715584Ki",
    "pods": "250"
  } }
```

To view the statistics of a node based labels - <oc adm top node --selector=' '>  (Operators used: =, != and ==). Note: <cluster-reader> permission is required to view the usage statistics. To delete all pods which are completed/failed  and free the resources use the below command.
For a current namespace

`$(oc get pods | grep Error | awk '{print $1}'); do oc delete pod --grace-period=1 ${pod}; done` 

For all namespaces

`$(oc get pods --all-namespaces | grep Error | awk '{print $1}'); do oc delete pod --grace period=1 ${pod}; done`

## 2. Reason: Node and pod affinity rules applied to the pod 
---------------------------------------------------------------
            Error ==> 1 node(s) didn't match pod affinity/anti-affinity
            
* If you do not carefully configure pod affinity, node affinity and pod anti-affinity with equal-priority pods, the pod might not be scheduled and ends up in pending/waiting state
* Depending on your pod priority and preemption settings, the scheduler might not be able to find an appropriate node for a pod without violating affinity requirements.

### How can this be resolved?
Pod affinity, pod anti-affinity and node affinity allow you to constrain which nodes your pod is eligible to be scheduled on based on the key/value labels on other pods.
Here is a sample configuration to help understand the application of affinity rules to nodes and pods
Sample pod configuration with a node selector rule :
```
    spec:
      nodeSelector:
        beta.kubernetes.io/os: linux
        node-role.kubernetes.io/worker: ''
        type: user-node
```

Sample pod configuration with a node affinity preferred rule :
``` 
       spec:
         affinity:
            nodeAffinity: 
               preferredDuringSchedulingIgnoredDuringExecution: 
               - weight: 1 
                  preference:
                matchExpressions:
                 - key: e2e-az-EastWest 
                    operator: In 
                    values:
                    - e2e-az-East 
                    - e2e-az-West     
```
Using node selectors and node affinity in the same pod configuration, note the following:

* If you configure both nodeSelector and nodeAffinity, both conditions must be satisfied for the pod to be scheduled onto a candidate node.

* If you specify multiple nodeSelectorTerms associated with nodeAffinity types, then the pod can be scheduled onto a node if one of the nodeSelectorTerms is satisfied.

* If you specify multiple matchExpressions associated with nodeSelectorTerms, then the pod can be scheduled onto a node only if all matchExpressions are satisfied.

## 3. Reason: Applying taints and tolerations
-------------------------------------------------
            Error ==> 1 node(s) had taints that the pod didn't tolerate
            
* When a taint is applied to a node, it repels the scheduling of a pod on that node unless it matches the taint with its tolerations.

### How can this be resolved?

Ensure that the tolerations are applied to the pods such that they match the taints of the nodes to end up being scheduled on the desired one. 

The taints applied to a node can be checked through ` oc describe node node-name | grep -i taint`. The pod toleration in the respective yaml file can be verified to check if the pod can be scheduled on that particular node.
   
Example : 
If a particular node has the following taints:

` oc describe node node-1 | grep -i taint `
``` 
Taints: node.alpha.kubernetes.io/not-ready:NoExecute
```
The pod with the appropriate tolerations would be schedules on node-1 such as:

```
tolerations:
- key: "node.alpha.kubernetes.io/not-ready" 
operator: "Exists" 
effect: "NoExecute"
```
## 4. Reason: Issues with Container engine on the host
---------------------------------------------------------
            Error ==> failed to pull image

* The specific image being used by the pod might not be available in the repository it is being pulled from. 

* As an example

` Normal   Pulling  10s (x4 over 94s)  kubelet, worker-1.prashanth-3d14.redhat.com  Pulling image "hello-openshift"  Warning  Failed   9s (x4 over 93s)   kubelet, worker-1.prashanth-3d14.redhat.com  Failed to pull image "hello-openshift": rpc error: code = Unknown desc = Error reading manifest latest in docker.io/hello-openshift/hello: errors `

### How can this be resolved?

The following can be checked:

* Make sure that you have the name of the image correct.              
* Does the image exist in the repository being pulled from?    
* Have you checked if the image is pushed to the repository?          
* Run a manual podman/docker pull image on your machine to see if the image can be pulled.

## 5. Reason: PersistentVolumeClaims
---------------------------------------
            Error ==> pod has unbound immediate PersistentVolumeClaims
            
* Pod can get stuck in pending state because the pod may have unbound immediate PersistentVolumeClaims. Can be checked with command - `oc describe pod podname` and  `oc get pvc -n namespace` shows status of pvc in unbound state.

* As an example.
`Warning  FailedScheduling  22s (x2 over 22s)  default-scheduler  pod has unbound immediate PersistentVolumeClaims`    

### How can this be resolved?

The OCP cluster inspects the persistent volume claim to find the bound volume and mounts that volume for a Pod.  For those persistent volumes that support multiple access modes, you must specify which mode applies when you use the claim as a volume in a Pod, in other words the 'accesModes' mentioned in the persistent volume claim of a given pod must match any existing persistent volume for the pod to be scheduled.

The claims remain unbound indefinitely if a matching volume does not exist or cannot be created with any available provisioner servicing a storage class.
 
Consider the following PV to be existing on the cluster.

Persitent volume specification:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0001 
spec:
  capacity:
    storage: 5Gi 
  accessModes:
    - ReadWriteOnce 
  persistentVolumeReclaimPolicy: Retain
```
For a pod to be scheduled on a given node in the cluster, the PVC should consist of the following specifications. 

Persistent Volume Claim specifications for a pod:

```
kind: PersistentVolumeClaim
metadata:
  name: nfs-claim1
spec:
  accessModes:
    - ReadWriteOnce 
  resources:
    requests:
      storage: 5Gi
```

## 6. Reason: Using host port
--------------------------------
                Error ==> node(s) didn't have free' ports for the requested pod ports

* When a pod uses a 'hostPort' for any of its containers, there are a limited number of places that the pod can be scheduled. The following error would be faced.

`Events:  Type     Reason            Age                From               Message  ----     ------            ----               ----               -------  Warning  FailedScheduling  6s (x25 over 34s)  default-scheduler  0/9 nodes are available: 4 node(s) didn't have free ports for the requested pod ports, 7 node(s) didn't match node selector.`

### How can this be resolved?
The below command outputs the container ports in all namespaces.

`oc get po --all-namespaces -o=jsonpath="{range .items[*]}{.spec.nodeName}{'\t'}{.spec.hostNetwork}{'\t'}{.spec.hostNetwork}{'\t'}{.spec.containers..containerPort}{'\n'}{end}"`

Using the above command, the occupied ports for the exisiting containers on all the nodes can be checked and if the port being requested is already occupied, the hostPort for the particular container can be modified in the requested pod configuration to use another port.