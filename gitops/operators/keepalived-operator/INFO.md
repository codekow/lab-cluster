# keepalived-operator

The objective of the keepalived operator is to allow for a way to create self-hosted load balancers in an automated way. From a user experience point of view the behavior is the same as of when creating [`LoadBalancer`](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) services with a cloud provider able to manage them.

The keepalived operator can be used in all environments that allows nodes to advertise additional IPs on their NICs (and at least for now, in networks that allow multicast), however it's mainly aimed at supporting LoadBalancer services and ExternalIPs on bare metal installations (or other installation environments where a cloud provider is not available).

One possible use of the keepalived operator is also to support [OpenShift Ingresses](https://docs.openshift.com/container-platform/4.5/networking/configuring_ingress_cluster_traffic/overview-traffic.html) in environments where an external load balancer cannot be provisioned.

## How it works

The keepalived operator will create one or more [VIPs](https://en.wikipedia.org/wiki/Virtual_IP_address) (an HA IP that floats between multiple nodes), based on the [`LoadBalancer`](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) services and/or services requesting [`ExternalIPs`](https://kubernetes.io/docs/concepts/services-networking/service/#external-ips).

For `LoadBalancer` services the IPs found at `.Status.LoadBalancer.Ingress[].IP` will become VIPs.

For services requesting a `ExternalIPs`, the IPs found at `.Spec.ExternalIPs[]` will become VIPs.

Note that a service can be of `LoadBalancer` type and also request `ExternalIPs`, it this case both sets of IPs will become VIPs.

Due to a [keepalived](https://www.keepalived.org/manpage.html) limitation a single keepalived cluster can manage up to 256 VIP configurations. Multiple keepalived clusters can coexists in the same network as long as they use different multicast ports [TODO verify this statement].

To address this limitation the `KeepalivedGroup` [CRD](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) has been introduced. This CRD is supposed to be configured by an administrator and allows you to specify a node selector to pick on which nodes the keepalived pods should be deployed. Here is an example:

```yaml
apiVersion: redhatcop.redhat.io/v1alpha1
kind: KeepalivedGroup
metadata:
  name: keepalivedgroup-router
spec:
  image: registry.redhat.io/openshift4/ose-keepalived-ipfailover
  interface: ens3
  nodeSelector:
    node-role.kubernetes.io/loadbalancer: ""
  blacklistRouterIDs:
  - 1
  - 2  
```

This KeepalivedGroup will be deployed on all the nodes with role `loadbalancer`. One must also specify the network device on which the VIPs will be exposed, it is assumed that all the nodes have the same network device configuration.

Services must be annotated to opt-in to being observed by the keepalived operator and to specify which KeepalivedGroup they refer to. The annotation looks like this:

`keepalived-operator.redhat-cop.io/keepalivedgroup: <keepalivedgroup namespace>/<keepalivedgroup-name>`

The image used for the keepalived containers can be specified with `.Spec.Image` it will default to `registry.redhat.io/openshift4/ose-keepalived-ipfailover` if undefined. 

## Requirements

### Security Context Constraints

Each KeepalivedGroup deploys a [daemonset](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) that requires the [privileged scc](https://docs.openshift.com/container-platform/4.5/authentication/managing-security-context-constraints.html), this permission must be given to the `default` service account in the namespace where the keepalived group is created by and administrator.

```shell
oc adm policy add-scc-to-user privileged -z default -n keepalived-operator
```

### Cluster Network Operator

In Openshift, use of an external IP address is governed by the following fields in the `Network.config.openshift.io` CR named `cluster`

* `spec.externalIP.autoAssignCIDRs` defines an IP address block used by the load balancer when choosing an external IP address for the service. OpenShift supports only a single IP address block for automatic assignment.

* `spec.externalIP.policy` defines the permissible IP address blocks when manually specifying an IP address. OpenShift does not apply policy rules to IP address blocks defined by `spec.externalIP.autoAssignCIDRs`

The following patch can be used to configure the Cluster Network Operator:

```yaml
spec:
  externalIP:
    policy:
      allowedCIDRs:
      - ${ALLOWED_CIDR}
    autoAssignCIDRs:
      - "${AUTOASSIGNED_CIDR}"
```

Here is an example of how to apply the patch:

```shell
export ALLOWED_CIDR="192.168.131.128/26"
export AUTOASSIGNED_CIDR="192.168.131.192/26"
oc patch network cluster -p "$(envsubst < ./network-patch.yaml | yq r -j -)" --type=merge
```
Additionally, the fields can be edited manually via `oc edit Network.config.openshift.io cluster`

## Blacklisting router IDs

If the Keepalived pods are deployed on nodes which are in the same network (same broadcast domain to be precise) with other keepalived the process, it's necessary to ensure that there is no collision between the used routers it.
For this purpose it is possible to provide a `blacklistRouterIDs` field with a list of black-listed IDs that will not be used.

## OpenShift RHV, vSphere, OSP and bare metal IPI instructions

When IPI is used for RHV, vSphere, OSP or bare metal platforms, three keepalived VIPs are deployed. To make sure that keepalived-operator can work in these environment we need to discover and blacklist the corresponding VRRP router IDs.

To discover the VRRP router IDs being used, run the following command, you can run this command from you laptop:

```shell
podman run quay.io/openshift/origin-baremetal-runtimecfg:4.5 vr-ids <cluster_name>
```

If you don't know your cluster name, run this command:

```shell
podman run quay.io/openshift/origin-baremetal-runtimecfg:4.5 vr-ids $(oc get cm cluster-config-v1 -n kube-system -o jsonpath='{.data.install-config}'| yq -r .metadata.name)
```

Then use these [instructions](#Blacklisting-router-IDs) to blacklist those VRRP router IDs.

## Verbatim Configurations

Keepalived has dozens of [configurations](https://www.keepalived.org/manpage.html). At the early stage of this project it's difficult to tell which one should be modeled in the API. Yet, users of this project may still need to use them. To account for that there is a way to pass verbatim options both at the keepalived group level (which maps to the keepalived config `global_defs` section) and at the service level (which maps to the keepalived config `vrrp_instance` section).

KeepalivedGroup-level verbatim configurations can be passed as in the following example:

```yaml
apiVersion: redhatcop.redhat.io/v1alpha1
kind: KeepalivedGroup
metadata:
  name: keepalivedgroup-router
spec:
  interface: ens3
  nodeSelector:
    node-role.kubernetes.io/loadbalancer: ""
  verbatimConfig:  
    vrrp_iptables: my-keepalived
```

this will map to the following `global_defs`:

```
    global_defs {
        router_id keepalivedgroup-router
        vrrp_iptables my-keepalived
    }
```

Service-level verbatim configurations can be passed as in the following example:

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    keepalived-operator.redhat-cop.io/keepalivedgroup: keepalived-operator/keepalivedgroup-router
    keepalived-operator.redhat-cop.io/verbatimconfig: '{ "track_src_ip": "" }'
```

this will map to the following `vrrp_instance` section

```
    vrrp_instance openshift-ingress/router-default {
        interface ens3
        virtual_router_id 1  
        virtual_ipaddress {
          192.168.131.129
        }
        track_src_ip
    }
```

## Metrics collection

Each keepalived pod exposes a [Prometheus](https://prometheus.io/) metrics port at `9650`. Metrics are collected with [keepalived_exporter](github.com/gen2brain/keepalived_exporter), the available metrics are described in the project documentation.

When a keepalived group is created a [`PodMonitor`](https://github.com/coreos/prometheus-operator/blob/master/Documentation/api.md#podmonitor) rule to collect those metrics. All PodMonitor resources created that way have the label: `metrics: keepalived`. It is up to you to make sure your Prometheus instance watches for those `PodMonitor` rules. Here is an example of a fragment of a `Prometheus` CR configured to collect the keepalived pod metrics:

```yaml
  podMonitorSelector:
    matchLabels:
      metrics: keepalived
```
