# How to set up Custom Grafana

OpenShift comes with Grafana by default, however, it is read-only, and thus you can't do custom queries and create custom dashboard for your particular needs.
For clusters on x86, you can refer to [this Red Hat blog](https://www.redhat.com/en/blog/custom-grafana-dashboards-red-hat-openshift-container-platform-4).
Sadly, there is no images available (at this time) for ppc64le, and thus the blog _doesn't work_.
This page describes how to overcome that for an OCP 4.x clusters running on Power/ppc64le, by leveraging operator images that IBM Cloud Pak utilizes, verified with OCP 4.7.4 in April, 2021.

**Disclaimer**: IBM does _not_ provide support for the public images that are used outside Cloud Pak context.

## Prerequisites

* Healthy OCP 4.x cluster with no cluster operators degraded or not available
* CRI-O on all cluster nodes can pull images from Internet (ie. [cluster-wide proxy](https://docs.openshift.com/container-platform/4.7/networking/enable-cluster-wide-proxy.html))

## Instructions

This is a boiled-down version that would get you a customizable Grafana instance running on your cluster with 
All this is based on, if you are curious or need more details, the IBM Knowledge Center articles at the bottom;

1. `oc create -f cs-catsrc.yaml` (below "YAML" section)
   <details>
   <summary>YAML</summary>
   <pre class="text">
    apiVersion: operators.coreos.com/v1alpha1
    kind: CatalogSource
    metadata:
      name: opencloud-operators
      namespace: openshift-marketplace
    spec:
      displayName: IBMCS Operators
      publisher: IBM
      sourceType: grpc
      image: docker.io/ibmcom/ibm-common-service-catalog:latest
      updateStrategy:
        registryPoll:
          interval: 45m
   </pre>

    (This was taken from "2. Installing the IBM Cloud Pak fundational services operator" from [this page](https://www.ibm.com/support/knowledgecenter/SSHKN6/installer/3.x.x/install_operator.html).)
   </details>

1. make sure `oc -n openshift-marketplace get pod | grep opencloud-operators` shows "Running"
1. `oc create -f cs-subs.yaml` (below "YAML" section)
   <details>
     <summary>YAML</summary>
     <pre class="text">
      apiVersion: v1
      kind: Namespace
      metadata:
        name: common-service
      ---
      apiVersion: operators.coreos.com/v1alpha2
      kind: OperatorGroup
      metadata:
        name: operatorgroup
        namespace: common-service
      spec:
        targetNamespaces:
        - common-service
      ---
      apiVersion: operators.coreos.com/v1alpha1
      kind: Subscription
      metadata:
        name: ibm-common-service-operator
        namespace: common-service
      spec:
        channel: v3
        installPlanApproval: Automatic
        name: ibm-common-service-operator
        source: opencloud-operators
        sourceNamespace: openshift-marketplace"
    </pre>

     (This was compiled to do the "Common Services" operator on OperatorHub GUI as intructed in "2. Installing the IBM Cloud Pak fundational services operator" from [this page](https://www.ibm.com/support/knowledgecenter/SSHKN6/installer/3.x.x/install_operator.html). You can definitely do the manual install on Web Console, if you so prefer.)
   </details>

1. `oc edit commonservices common-service -n ibm-common-services` and replace `spec:` stanza with contents below;
   <details>
     <summary>YAML snippet</summary>
     <pre class="text">
      spec:
        manualManagement: true
        services:
        - name: ibm-monitoring-grafana-operator
          spec:
            grafana:
              dashboardConfig:
                resources:
                  limits:
                    cpu: 70m
                    memory: 170Mi
                  requests:
                    cpu: 25m
                    memory: 140Mi
              datasourceConfig:
                type: openshift
              grafanaConfig:
                resources:
                  limits:
                    cpu: 150m
                    memory: 230Mi
                  requests:
                    cpu: 25m
                    memory: 200Mi
              isHub: false
              routerConfig:
                resources:
                  limits:
                    cpu: 70m
                    memory: 80Mi
                  requests:
                    cpu: 25m
                    memory: 65Mi
        size: medium
     </pre>
     (This was culled and put together from [this page](https://www.ibm.com/support/knowledgecenter/SSHKN6/installer/3.x.x/custom_resource.html) for Grafana.)
   </details>

1. `oc apply -f cs-grafana.yaml` (below "YAML" section)
   <details>
   <summary>YAML</summary>

     <pre class="text">
     apiVersion: operator.ibm.com/v1alpha1
     kind: OperandRequest
     metadata:
       name: common-service
       namespace: ibm-common-services
     spec:
       requests:
         - operands:
             - name: ibm-monitoring-grafana-operator
           registry: common-service
     </pre>
     (This is what's needed for Grafana, based on all the stuff described in [this page](https://www.ibm.com/support/knowledgecenter/SSHKN6/monitoring/1.x.x/monitoring_service.html#install_monitsrv).)
   </details>

1. give it some time (half an hour?) to get all the necessary pods up and running: `oc get pod -n ibm-common-services`

## References

* [IBM Cloud Pak foundational services documentation top page](https://www.ibm.com/support/knowledgecenter/SSHKN6/installer/3.x.x/install_operator.html)
* [Monitoring Service section of the same doc site](https://www.ibm.com/docs/en/cpfs?topic=monitoring-1xx-operator)