# How to Troubleshoot DNS

Clearly this is a very big topic, and there's no way this page can capture every trouble with DNS.
Kubernetes has a generic debug guide for checking the basic setup of DNS on a cluster using a ready image `dnsutils`.
See <https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/>.

This living page is just our place to capture and document what we faced, debugged, understood, the issues and "gotchas," on Power, though many may be architecture agnostic.

## Use of .nip.io Domain for Your Cluster

As many of you know, [.nip.io](https://nip.io) is a pretty cool service that would give you a publicly resolvable domain at will.
Using the service, regardless of how you deploy your OCP 4.x cluster, you may choose to make your cluster be accessed like <https://api.mycluster.1.2.3.4.nip.io> when your public IP address is `1.2.3.4`, and your Web Console at <https://console-openshift-console.apps.mycluster.1.2.3.4.nip.io>, etc..
This is wonderful for external clients because they don't have to add some entries in their `/etc/hosts` file to reach my test cluster that no public DNS servers know about.

In the meantime, the underlying K8s (ie. 1.19, 1.20) has `ndots:5` as its default in `/etc/resolv.conf` of every pod/container, unless overridden with [`dnsConfig` in the YAML](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-dns-config).
This `ndots` setting and how it works in (and messes with) K8s/OpenShift is well explained online. See, for example;  

* <https://mrkaran.dev/posts/ndots-kubernetes/>
* <https://pracucci.com/kubernetes-dns-resolution-ndots-options-and-why-it-may-affect-application-performances.html>

What led us into this was some test failures in [OpenShift Serverless](https://docs.openshift.com/container-platform/4.7/serverless/serverless-getting-started.html) on a test cluster with a [.nip.io](https://nip.io) domain, like `api.testocp47.1.2.3.4.nip.io`, where that `1.2.3.4` is public accessible and can spray incoming requests to the cluster.

Here's the nitty-gritty of the problem as we understood.
Let's say a pod wants to call another on the same cluster, and the name the calling pod knows and calls the other is `myendpoint.default.svc.cluster.local`.
It has 4 dots, while the pod's `/etc/resolv.conf` has `option ndots:5` by default, which means, if the query name doesn't have 5 or more dots, it would assume it's not a FQDN, and therefore, it'd try all the domains listed on `search` line first;
```text
search default.svc.cluster.local svc.cluster.local cluster.local testocp47.1.2.3.4.nip.io 1.2.3.4.nip.io
nameserver 172.30.0.10
options ndots:5
```
If the base domain were like [.ibm.com](https://ibm.com), there is no chance that big enterprise's public DNS server knows about my mere test cluster, and thus, even if it tries all the search domains, nothing resolves, like;

1. myendpoint.default.svc.cluster.local.default.svc.cluster.local
1. myendpoint.default.svc.cluster.local.svc.cluster.local
1. myendpoint.default.svc.cluster.local.cluster.local
1. myendpoint.default.svc.cluster.local.testocp47.ibm.com
1. myendpoint.default.svc.cluster.local.ibm.com

and at last the original query is tried as FQDN, and it would resolve to an internal IP. All good!

However, take a look at the case with [.nip.io](https://nip.io);

1. myendpoint.default.svc.cluster.local.default.svc.cluster.local
1. myendpoint.default.svc.cluster.local.svc.cluster.local
1. myendpoint.default.svc.cluster.local.cluster.local
1. myendpoint.default.svc.cluster.local.testocp47.1.2.3.4.nip.io **<<< this would resolve to 1.2.3.4! (public) **
1. myendpoint.default.svc.cluster.local.1.2.3.4.nip.io

This is precisely where [.nip.io](https://nip.io) is a double-edged sword.
The little pod-to-pod communication is not intended to be called from outside, via public IP address, and thus would fail because you wouldn't have a Route exposed (for obvious security reasons).

What can you do?
If you can change the name the calling pod uses to look up the callee, change it to a shorter version like `myendpoint.default`, then it would resolve at the 2nd attempt to an internal IP (see below).

1. myendpoint.default.default.svc.cluster.local
1. myendpoint.default.svc.cluster.local **<<< this would resolve to 172.30.0.x! (internal)**
1. myendpoint.default.cluster.local
1. myendpoint.default.testocp47.1.2.3.4.nip.io <<< this would resolve to 1.2.3.4, but moot
1. myendpoint.default.1.2.3.4.nip.io

If the name is out of reach, inside the `image:` that you can't tweak, then try adding the `dnsConfig:` to the pod YAML with `ndots:3` or something, so the name the caller uses would be considered as FQDN (with 4 dots), and thus no trying with the `search` list.
This would affect only the pod, not everything on the cluster, and thus perhaps _safer_, but there is still a risk of breaking something else unexpectedly.
So, beware!