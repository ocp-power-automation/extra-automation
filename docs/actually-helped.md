Collection of issues we faced and external pages/links that actually helped resolve them;

## 2021-03

* A compute or storage got deleted perhaps by accident or mistake, and now your `terraform.tfstate` is out of sync.
`terraform refresh` is the first thing to try, as it queries the target IaaS and updates the .tfstate file accordingly.
If for some reason that doesn't work, use this `terraform state rm` coommand that can delete resources properly in the file.
First, you run `terraform state list` and find out which module corresponds to the infrastructure resource(s) that are now gone.
After determining which one(s), you can go ahead and run `terraform state rm` with those, perhaps in quotes, one at a time.
Once done, `terraform.tfstate` file is back in sync, and other Terraform commands (eg. `refresh`, `apply`, `destroy`) should be able to run happily.  
 => <https://www.terraform.io/docs/cli/commands/state/rm.html>

##### (2021-02-24 4.7 GA)

## 2021-02

* `image-registry` operator reporting Progressing, while Available and not Degraded, after starting with "EmptyDir" storage, switching to PVC (with `nfs-provisioner`), then deleting and recreating the provisioner Pod. The `cluster` config of `image-registry` ends up with 2 storage types, which it cannot support at this time (tested/verified with 4.6.13).  
  To edit the config, use: `$ oc edit configs.imageregistry.operator.openshift.io`  
  => <https://access.redhat.com/solutions/4516391>

## 2021-01

* 4.6 node reporting `NotReady` after upgrade to 4.6.12, due to CRI-O failing to start up on one worker node (other nodes were fine).
  The error seen in `journalctl` on the troubling node was like `Jan 21 16:00:48 worker1.ocp4.example.com bash[1881]: Error: readlink /var/lib/containers/storage/overlay/l/6MDZKPV63T>`  
  Running `# systemctl daemon-reload` was necessary before the storage wipe steps in the below doc.  
  => <https://access.redhat.com/solutions/5350721> (again, see [2020-10](#2020-10))

* OCS (OpenShift Container Storage) 4.6 install worked like a champ following this document.
  => <https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage/4.6/html-single/monitoring_openshift_container_storage/index>

##### (2020-10-26 4.6 GA)

## 2020-10

* Pods failing to start on a particular node, often with `ImagePullBackOff` error, due to getting CRI-O ephemeral storage full  
  => <https://access.redhat.com/solutions/5350721>

* How to troubleshoot ingress, routes, services, what objects to check, in what order (perhaps for likeliness/quicker resolution)  
  => <https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/>

* 4.5 disconnected cluster (ie. no direct access to Internet), normal upgrade fails, requiring "opting out Insights operator" and "mirroring image repository (again)." 
  `oc adm upgrade` first checks if the cluster is currently healthy, and any problem with cluster operators (`oc get co`) blocks upgrade operation from proceeding. 
  The Insights operator would be running but degraded because it cannot report status to Red Hat due to no Internet access, hence needs "opting out."
  Then, the upgrade-to version images need to be mirrored to the internal repository, again, due to no Internet access.  
  => <https://docs.openshift.com/container-platform/4.5/support/remote_health_monitoring/opting-out-of-remote-health-reporting.html>  
  => <https://docs.openshift.com/container-platform/4.5/updating/updating-restricted-network-cluster.html>

* How to install, configure, use Node Feature Discovery operator. 
  The article was written for OCP 4.1, therefore the UI is a little different in 4.6, but it's actually easier (without having to deal with YAML) in 4.6.  
  => <https://access.redhat.com/solutions/4734811>

## 2020-09

* 4.5 on libvirt, `authentication` and `console` operators not available, and `console` pod's log indicating DNS issue for "oauth-openshift.<domain>"  
  => <https://github.com/openshift/installer/issues/1648#issuecomment-585235423>
  
## 2020-08

* node `NotReady` after reboot perhaps after shut down for a few days, with "Kubelet stopped posting node status" in `oc describe node`  
  essentially, `sudo systemctl restart kubelet` on the bad node, followed by `oc get csr | grep Pending | awk '{print $1}' | xargs oc adm certificate approve`, perhaps multiple times as a few CSRs could come in for approval sequentially.  
  => <https://docs.openshift.com/container-platform/4.1/backup_and_restore/disaster_recovery/scenario-3-expired-certs.html>
  
* random intermittent API server errors like "connection refused," "authentication required," "internal error," with `etcd` running on slow disk  
  => <https://github.com/openshift/release/blob/23074b5/ci-operator/templates/openshift/installer/cluster-launch-installer-openstack-e2e.yaml#L375-L382>

##### (2020-07-30 4.5 GA)

##### (2020-06-23 4.4 GA)

## 2020-06

* `OOMKilled` on _large_ POWER system, while same set of pods run fine on _smaller_ x86 system, with possible tinkering with `slub_max_order` kernel param  
  => <https://medium.com/ibm-cloud/fortifying-ibm-cloud-private-for-large-enterprise-power-systems-6119804f0103>

##### (2020-04-30 4.3 GA)

## 2020-02

* 4.x image registry not accessible externally by default, need `default-route` created  
  => <https://docs.openshift.com/container-platform/4.3/registry/securing-exposing-registry.html>  
  => <https://docs.openshift.com/container-platform/4.4/registry/configuring-registry-operator.html#registry-operator-default-crd_configuring-registry-operator>