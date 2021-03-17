# Deployment of Kruize on Openshift on Power

### Introduction

There were times when we used to have challenges in right sizing and optimizing containers which leads to these drawback - When applications were not sized appropriately in a k8s cluster, they either waste resources (oversized) or worse get terminated (undersized), either way could result in loss of revenue. Quick fix for this issue is **Kruize**. Kruize is an opensource tool which monitors application containers for resource usage. It has an analysis engine which predicts the right size for the containers that are being monitored and offers recommendations on the right CPU and Memory `request` and `limit` values. This helps IT admins to review and apply the recommendations knowing that their applications and even the cluster is optimally sized.

The install steps that [kruize](https://github.com/kruize/kruize/blob/master/docs/README.md#openshift) has generally worked on an OpenShift cluster on IBM Power Systems, but you might find below some tweaks we had to make.

> **_NOTE:_** It is recommended to run the pods that are being monitored by kruize with no requests and limits set (or with very high limits set). This helps kruize to understand what is the real usage and hence can recommend the right set of values.

 
### Steps for deploying kruize on IBM Power Systems  

**1. Clone the kruize github repository :**

`$ git clone https://github.com/kruize/kruize.git` 

`$ cd kruize`

Now make the following changes in **manifests/kruize.yaml_template**

```text
- apiVersion: extensions/v1beta1
+ apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: kruize
      labels:
        app: kruize
    spec:
      replicas: 1
 +    selector:
 +      matchLabels:
 +        app: kruize
 +        name: kruize
 +        app.kubernetes.io/name: "kruize"
      template:
        metadata:
          labels:
```

**2. Run the `deploy.sh` script to start deployment :** 

```text
$ ./deploy.sh -c openshift

Sample output:

###  Installing kruize for OpenShift


WARNING: This will create a Kruize ServiceMonitor object in the openshift-monitoring namespace
WARNING: This is currently not recommended for production

Create ServiceMonitor object and continue installation?(y/n)? yes

Info: Checking pre requisites for OpenShift...done
Info: Logging in to OpenShift cluster...
Authentication required for https://api.kruize.cp.fyre.ibm.com:6443 (openshift)
Username: kubeadmin
Password: 
Login successful.

You have access to 58 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".


Info: Setting Prometheus URL as https://prometheus-k8s-openshift-monitoring.apps.kruize.cp.fyre.ibm.com
Info: Deploying kruize yaml to OpenShift cluster

Now using project "openshift-monitoring" on server "https://api.kruize.cp.fyre.ibm.com:6443".

deployment.apps/kruize created
service/kruize configured

Info: Waiting for kruize to come up...
kruize-6cfc8f4bbf-956vq                       1/1     ContainerCreating   0          11m
kruize-f54c97d6f-78l7m                        1/1     Running             0          4s
Info: kruize deploy succeeded: Running
kruize-6cfc8f4bbf-956vq                       1/1     ContainerCreating   0          11m
kruize-f54c97d6f-78l7m                        1/1     Running             0          4s
```

> **_NOTE:_**  If kruize pod is not in _Running_ state then check the logs of the failed kruize pod by executing 
> `oc logs -f <kruize_pod_name>`
> If log says execution format error then the image would be of some other architecture(probably amd64) .
If that's the case, you can build your own power(ppc64le) image by using `build.sh` script present [here](https://github.com/kruize/kruize/blob/master/build.sh) 
But before running `./build.sh` make sure you have docker installed on your system .


**3. Once the deployment is complete, create a route in order to access the Kruize API**

```text
$ cat route.yaml

kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: kruize
  namespace: openshift-monitoring
  labels:
    app: kruize
  annotations:
    openshift.io/host.generated: 'true'
spec:
  host: kruize-openshift-monitoring.apps.kruize.cp.fyre.ibm.com
  to:
    kind: Service
    name: kruize
    weight: 100
  port:
    targetPort: http
  wildcardPolicy: None
status:
  ingress:
    - host: kruize-openshift-monitoring.apps.kruize.cp.fyre.ibm.com
      routerName: default
      wildcardPolicy: None
      routerCanonicalHostname: apps.kruize.cp.fyre.ibm.com
```
> _NOTE:_ Change the values of `host` and `routerCanonicalHostname` according to your cluster. 

* Run: `oc create -f route.yaml`

**4. Use the API to view recommendations** 

* List recommendations for all applications by doing :

$ curl http://<`host_of_your_cluster`>/recommendations?listApplications 

_Example_: 

```text
[root@kruize-inf ~]# curl http://kruize-openshift-monitoring.apps.kruize.cp.fyre.ibm.com/recommendations?listApplications

[
  {
    "application_name": "kube-state-metrics",
    "resources": {
      "requests": {
        "memory": "0.0M",
        "cpu": 0.0
      },
      "limits": {
        "memory": "0.0M",
        "cpu": 0.0
      }
    }
  },
  {
    "application_name": "node-exporter",
    "resources": {
      "requests": {
        "memory": "125.5M",
        "cpu": 0.5
      },
      "limits": {
        "memory": "150.7M",
        "cpu": 1.0
      }
    }
  },
  {
    "application_name": "kruize",
    "resources": {
      "requests": {
        "memory": "158.2M",
        "cpu": 0.5
      },
      "limits": {
        "memory": "278.1M",
        "cpu": 1.0
      }
    }
  },
  {
    "application_name": "thanos-querier",
    "resources": {
      "requests": {
        "memory": "0.0M",
        "cpu": 0.0
      },
      "limits": {
        "memory": "0.0M",
        "cpu": 0.0
      }
    }
  },
  {
    "application_name": "prometheus-operator",
    "resources": {
      "requests": {
        "memory": "133.8M",
        "cpu": 0.5
      },
      "limits": {
        "memory": "160.6M",
        "cpu": 1.0
      }
    }
  }
]

```
> _NOTE:_ This is initial output before adding any load on application, and it shows all applications that are deployed in `openshift-monitoring` namespace because Kruize is also deployed in this namespace.

* List recommendations for single application:
 
```text
$ curl http://kruize-openshift-monitoring.apps.kruize.cp.fyre.ibm.com/recommendations?application_name=kruize

[
 {
    "application_name": "kruize",
    "resources": {
      "requests": {
        "memory": "158.2M",
        "cpu": 0.5
      },
      "limits": {
        "memory": "278.1M",
        "cpu": 1.0
       }
     }
   }
]
```

**5. Label the pod to monitor your application:** 

Kruize by default monitors `openshift-monitoring` namespace. In order to monitor your application, run the following command:
```text
$ oc label pod -n <NS> <POD_NAME> app.kubernetes.io/name=<ANY_NAME>
```
NS stands for namespace name

### Examples

##### 1. To monitor deployment for an acmeair application:
 
```text
[root@kruize-inf ~]# git clone https://github.ibm.com/powercloud/acmeair.git

Cloning into 'acmeair'...
remote: Enumerating objects: 31, done.
remote: Total 31 (delta 0), reused 0 (delta 0), pack-reused 31
Unpacking objects: 100% (31/31), done.

[root@kruize-inf ~]# cd acmeair/
[root@kruize-inf ~]# git checkout powervs

Branch 'powervs' set up to track remote branch 'powervs' from 'origin'.
Switched to a new branch 'powervs'

[root@kruize-inf acmeair]# ls
acmeair.PNG  deploy.sh  openshift-manifest  README.md

[root@kruize-inf acmeair]# ./deploy.sh

Route Host=acmeair.apps.kruize.cp.fyre.ibm.com
Now using project "acme" on server "https://api.kruize.cp.fyre.ibm.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app rails-postgresql-example

to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=k8s.gcr.io/serve_hostname

Already on project "acme" on server "https://api.kruize.cp.fyre.ibm.com:6443".
Adding quay.io/pravin_dsilva/ for main service
Patching Route Host: acmeair.apps.kruize.cp.fyre.ibm.com for main service
Adding quay.io/pravin_dsilva/ for auth service
Patching Route Host: acmeair.apps.kruize.cp.fyre.ibm.com for auth service
Adding quay.io/pravin_dsilva/ for booking service
Patching Route Host: acmeair.apps.kruize.cp.fyre.ibm.com for booking service
Adding quay.io/pravin_dsilva/ for customer service
Patching Route Host: acmeair.apps.kruize.cp.fyre.ibm.com for customer service
Adding quay.io/pravin_dsilva/ for flight service
Patching Route Host: acmeair.apps.kruize.cp.fyre.ibm.com for flight service
route.route.openshift.io/acmeair-auth-route created
deployment.apps/acmeair-authservice created
service/acmeair-auth-service created
route.route.openshift.io/acmeair-booking-route created
deployment.apps/acmeair-bookingservice created
service/acmeair-booking-service created
service/acmeair-booking-db created
deployment.apps/acmeair-booking-db created
route.route.openshift.io/acmeair-customer-route created
deployment.apps/acmeair-customerservice created
service/acmeair-customer-service created
service/acmeair-customer-db created
deployment.apps/acmeair-customer-db created
route.route.openshift.io/acmeair-flight-route created
deployment.apps/acmeair-flightservice created
service/acmeair-flight-service created
service/acmeair-flight-db created
deployment.apps/acmeair-flight-db created
route.route.openshift.io/acmeair-main-route created
deployment.apps/acmeair-mainservice created
service/acmeair-main-service created
Acme Air Deployment complete. Access the application at acmeair.apps.kruize.cp.fyre.ibm.com/acmeair

[root@kruize-inf acmeair]# oc get pods
NAME                                       READY   STATUS    RESTARTS   AGE
acmeair-authservice-55648d8445-4k4r4       1/1     Running   0          57m
acmeair-booking-db-6ffc689d55-kxs9r        1/1     Running   0          57m
acmeair-bookingservice-76fc8f7f84-5vsml     1/1     Running   0          57m
acmeair-customer-db-ff6bcc8fc-qzxxp       1/1     Running   0          57m
acmeair-customerservice-77b69b8f58-wvwrb   1/1     Running   0          57m
acmeair-flight-db-6f9ff4dd6f-zczrl         1/1     Running   0          57m
acmeair-flightservice-7d47988b8b-hwqjg     1/1     Running   0          57m
acmeair-mainservice-7f7ff489d-qhss7       1/1     Running   0          57m
```

* In order to monitor your application, label the pod as mentioned in Step5:
```
[root@kruize-inf acmeair]# oc label pod -n acme acmeair-customer-db-ff6bcc8fc-qzxxp app.kubernetes.io/name=acmeair-customer-db
```
pod/acmeair-customer-db-ff6bcc8fc-qzxxp labeled

* List recommendations: 

```text
[root@kruize-inf acmeair]# curl http://kruize-openshift-monitoring.apps.kruize.cp.fyre.ibm.com/recommendations?application_name=acmeair-customer-db

[
  {
    "application_name": "acmeair-customer-db",
    "resources": {
      "requests": {
        "memory": "0.0M",
        "cpu": 0.0
      },
      "limits": {
        "memory": "0.0M",
        "cpu": 0.0
      }
    }
  }
]
```
> _NOTE:_ The above outputs are before adding any load on the application.

**After adding load:** 

 
curl http://acmeair.apps.kruize.cp.fyre.ibm.com:6443/booking/loader/load

curl http://acmeair.apps.kruize.cp.fyre.ibm.com:6443/flight/loader/load

curl http://acmeair.apps.kruize.cp.fyre.ibm.com:6443/customer/loader/load?numCustomers=100000 


**To list recommendations for single application:** 

```text
[root@kruize-inf acmeair]# curl http://kruize-openshift-monitoring.apps.kruize.cp.fyre.ibm.com/recommendations?application_name=acmeair-customer-db

[
  {
    "application_name": "acmeair-customer-db",
    "resources": {
      "requests": {
        "memory": "194.6M",
        "cpu": 0.5
      },
      "limits": {
        "memory": "233.5M",
        "cpu": 1.0
      }
    }
  }
]
```

**List of recommendations on all the applications:**  

```text
[root@kruize-inf acmeair]# curl http://kruize-openshift-monitoring.apps.kruize.cp.fyre.ibm.com/recommendations?listApplications

[
  {
    "application_name": "kube-state-metrics",
    "resources": {
      "requests": {
        "memory": "0.0M",
        "cpu": 0.0
      },
      "limits": {
        "memory": "0.0M",
        "cpu": 0.0
      }
    }
  },
  {
    "application_name": "node-exporter",
    "resources": {
      "requests": {
        "memory": "125.8M",
        "cpu": 0.5
      },
      "limits": {
        "memory": "151.4M",
        "cpu": 1.0
      }
    }
  },
  {
    "application_name": "kruize",
    "resources": {
      "requests": {
        "memory": "175.9M",
        "cpu": 0.5
      },
      "limits": {
        "memory": "278.1M",
        "cpu": 1.0
      }
    }
  },
  {
    "application_name": "acmeair-customer-db",
    "resources": {
      "requests": {
        "memory": "194.6M",
        "cpu": 0.5
      },
      "limits": {
        "memory": "233.5M",
        "cpu": 1.0
      }
    }
  },
  {
    "application_name": "thanos-querier",
    "resources": {
      "requests": {
        "memory": "0.0M",
        "cpu": 0.0
      },
      "limits": {
        "memory": "0.0M",
        "cpu": 0.0
      }
    }
  },
  {
    "application_name": "prometheus-operator",
    "resources": {
      "requests": {
        "memory": "134.1M",
        "cpu": 0.5
      },
      "limits": {
        "memory": "161.6M",
        "cpu": 1.0
      }
    }
  }
]
```
 
##### 2. To monitor **osd** pod in Openshift Container Storage: 
 
```text
[root@kruize-inf ~]# oc get pods -n openshift-storage |grep osd

NAME                                                              READY   STATUS      RESTARTS   AGE
rook-ceph-osd-0-6499496d58-pwml8                                  1/1     Running     0          12m
rook-ceph-osd-1-5bd4d44b6f-dd6nq                                  1/1     Running     22         2d15h
rook-ceph-osd-2-655f579dc7-65c7p                                  1/1     Running     0          141m
```

- Label the osd pod which you want to monitor: 

```
[root@kruize-inf ~]# oc label pod rook-ceph-osd-1-5bd4d44b6f-dd6nq -n openshift-storage app.kubernetes.io/name=rook-ceph-osd-1
```

- View Recommendations: 

**Before adding load:**  

```text
[root@kruize-inf ~]# curl http://kruize-openshift-monitoring.apps.kruize.cp.fyre.ibm.com/recommendations?application_name=rook-ceph-osd-1

[
  {
    "application_name": "rook-ceph-osd-1",
    "resources": {
      "requests": {
        "memory": "0.0M",
        "cpu": 0.0
      },
      "limits": {
        "memory": "0.0M",
        "cpu": 0.0
      }
    }
  }
]
```  

**After running some tests/adding load:**  


```text
[root@kruize-inf ~]# curl http://kruize-openshift-monitoring.apps.kruize.cp.fyre.ibm.com/recommendations?application_name=rook-ceph-osd-1

[
  {
    "application_name": "rook-ceph-osd-1",
    "resources": {
      "requests": {
        "memory": "3427.2M",
        "cpu": 0.5
      },
      "limits": {
        "memory": "6415.4M",
        "cpu": 1.0
      }
    }
  }
]
```




