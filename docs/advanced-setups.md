## Advanced Setups

To get started with CAPMOX please refer to the [Getting Started](Usage.md#quick-start) section.

## Multiple NICs

If you want to create VMs with multiple network devices,
You will need to create `InClusterPool` or `GlobalInClusterPool` to manage IPs.

here is a `GlobalInClusterPool` example:

```yaml
apiVersion: ipam.cluster.x-k8s.io/v1alpha2
kind: GlobalInClusterIPPool
metadata:
  name: shared-inclusterippool
spec:
  addresses: ${NODE_SECONDARY_IP_RANGES}
  prefix: ${SECONDARY_IP_PREFIX}
  gateway: ${SECONDARY_GATEWAY}
```

In the cluster template flavor=multiple-vlans you can define a secondary network device for the VMs.
To do that you will need to set extra environment variables along with the required ones:

```bash
# The secondary IP ranges for Cluster nodes
export NODE_SECONDARY_IP_RANGES="[10.10.10.100-10.10.10.150]"
# The Subnet Mask in CIDR notation for your node secondary IP ranges
export SECONDARY_IP_PREFIX=24
# The secondary gateway for the machines network-config
export SECONDARY_GATEWAY="10.10.10.254"
# The secondary dns nameservers for the machines network-config
export SECONDARY_DNS_SERVERS="[8.8.8.8, 8.8.4.4]"
# The Proxmox secondary network bridge for VMs
export SECONDARY_BRIDGE=vmbr2
```

### Multiple gateways
If you have multiple gateways (especially without VRF devices), you may
want to control gateway selection by inserting metrics.
For this purpose, you can add a metric annotation to your pools:

```yaml
apiVersion: ipam.cluster.x-k8s.io/v1alpha2
kind: GlobalInClusterIPPool
metadata:
  annotations:
    metric: "200"
  name: shared-inclusterippool
spec:
  addresses: ${NODE_SECONDARY_IP_RANGES}
  prefix: ${SECONDARY_IP_PREFIX}
  gateway: ${SECONDARY_GATEWAY}
```
This annotation will be used when creating a netplan definition for a VM.

The metric of the default gateway can be controlled with the proxmoxcluster definition:
```yaml
[...]
    ipv4Config:
      addresses:
      - 10.10.0.70-10.10.0.79
      gateway: 10.10.0.1
      metric: 100
      prefix: 24
```

Metrics are, like all network configuration, part of bootstrap, and will not reconcile.

#### Generate a Cluster

```bash
clusterctl generate cluster test-multiple-vlans  \
  --infrastructure proxmox \
  --kubernetes-version v1.31.6  \
  --control-plane-machine-count=1 \
  --worker-machine-count=2 \
  --flavor=multiple-vlans > cluster.yaml
```

## Dual Stack

Regarding dual-stack support, you can use the following environment variables to define the IPv6 ranges for the VMs:

```bash
# The IPv6 ranges for Cluster nodes
export NODE_IPV6_RANGES="[2001:db8:1::1-2001:db8:1::10]"
# The Subnet Mask in CIDR notation for your node IPv6 ranges
export IPV6_PREFIX=64
# The ipv6 gateway for the machines network-config.
export IPV6_GATEWAY="2001:db8:1::1"
```

If you're using cilium, be aware that cilium's helm chart requires `ipv6.enabled=true` to actually support IPv6 pod- and service networks.

## IPv6 only cluster

Clusters without IPv4 are possible, but require kube-vip to be newer than 0.7.1 (version 0.7.0 probably works, but we did not test it).

If you're using cilium, be aware that Cilium's helm chart requires `ipv6.enabled=true` to actually support IPv6 pod- and service networks.


#### Generate a Cluster

```bash
clusterctl generate cluster test-duacl-stack  \
  --infrastructure proxmox \
  --kubernetes-version v1.31.6  \
  --control-plane-machine-count=1 \
  --worker-machine-count=2 \
  --flavor=dual-stack > cluster.yaml
```


## Cluster with LoadBalancer nodes

The template for LoadBalancers is for [dual stack](##dual-stack) with [multiple nics](##multiple-nics). All
environment variables regarding those need to be set. You may want to reduce the template to your usecase.

The idea is that there are special nodes for load balancing. These have an extra network card which is supposed
to be connected to the BGP receiving switches. All services exposed with the type "LoadBalancer" will take an
IP from `METALLB_IPV4_RANGE` or `METALLB_IPV6_RANGE` which will be announced to the BGP peers.

The template presupposes two bgp peers per address family (ipv4,ipv6) because this is a high availability setup.

For the routing to work, we employ source ip based routing. This does not work (reliably) without source IPs.
For this reason, all nodes are created with `ipvs` in kube-proxy. This neccesitates also setting `strictARP`,
as otherwise packets may still take wrong paths and cause reverse path filter issues.

If you require changing `METALLB_IPV{4,6}_RANGE` after a cluster has been deployed, you need to redeploy load balancer
nodes, as these variables are also used in bootstrap to establish source ip based routing.

LoadBalancer nodes are tainted and only run pods required for load balancing.

```
## -- loadbalancer nodes -- #
LOAD_BALANCER_MACHINE_COUNT: 2                                # Number of load balancer nodes
EXT_SERVICE_BRIDGE: "vmbr2"                                   # The network bridge device used for load balancing and bgp.
LB_BGP_IPV4_RANGES: "[172.16.4.10-172.16.4.20]"               # The IP ranges used by the cluster for establishing the bgp session.
LB_BGP_IPV6_RANGES:
LB_BGP_IPV4_PREFIX: "24"                                      # Subnet Mask in CIDR notation for your bgp IP ranges.
LB_BGP_IPV6_PREFIX:
METALLB_IPV4_ASN: "65400"                                     # The nodes bgp asn.
METALLB_IPV6_ASN:
METALLB_IPV4_BGP_PEER: "172.16.4.1"                           # The nodes bgp peer IP address.
METALLB_IPV4_BGP_PEER2: "172.16.4.2"                          # Backup bgp peer for H/A
METALLB_IPV6_BGP_PEER:
METALLB_IPV6_BGP_PEER2:
METALLB_IPV4_BGP_SECRET: "REDACTED"                           # The secret required to establish a bgp session (if any).
METALLB_IPV6_BGP_SECRET:
METALLB_IPV4_BGP_PEER_ASN: "65500"                            # The bgp peer's asn.
METALLB_IPV4_BGP_PEER2_ASN:                                   # Backup bgp peer's asn
METALLB_IPV6_BGP_PEER_ASN:
METALLB_IPV6_BGP_PEER2_ASN:
METALLB_IPV4_RANGE: 7.6.5.0/24                                # The IP Range MetalLB uses to announce your services.
METALLB_IPV6_RANGE:
```

#### Generate a Cluster

```bash
clusterctl generate cluster test-bgp-lb  \
  --infrastructure proxmox \
  --kubernetes-version v1.31.6  \
  --control-plane-machine-count=1 \
  --worker-machine-count=2 \
  --flavor=cilium-load-balancer > cluster.yaml
```

#### Node over-/ underprovisioning

By default our scheduler only allows to allocate as much memory to guests as the host has. This might not be a desirable behaviour in all cases. For example, one might to explicitly want to overprovision their host's memory, or to reserve bit of the host's memory for itself.

This behaviour can be configured in the `ProxmoxCluster` CR through the field `.spec.schedulerHints.memoryAdjustment`.

For example, setting it to `0` (zero), entirely disables scheduling based on memory. Alternatively, if you set it to any value greater than `0`, the scheduler will treat your host as it would have `${value}%` of memory. In real numbers that would mean, if you have a host with 64GB of memory and set the number to `300`, the scheduler would allow you to provision guests with a total of 192GB memory and therefore overprovision the host. (Use with caution! It's strongly suggested to have memory ballooning configured everywhere.). Or, if you were to set it to `95` for example, it would treat your host as it would only have 60,8GB of memory, and leave the remaining 3,2GB for the host.

## Template lookup based on Proxmox tags

Our provider is able to look up templates based on their attached tags, for `ProxmoxMachine` resources, that make use of an tag selector.

For example, you can set the `TEMPLATE_TAGS="tag1,tag2"` environment variable. Your custom image will then be used when using the [auto-image](https://github.com/ionos-cloud/cluster-api-provider-ionoscloud/blob/main/templates/cluster-template-auto-image.yaml) template.

Please note: Passed tags must be an exact 1:1 match with the tags on the template you want to use.  The matched result must be unique. If multiple templates are found, provisioning will fail.

## Proxmox RBAC with least privileges

For the Proxmox API user/token you create for CAPMOX, these are the minimum required permissions.

### Prerequisites

* Create a role called `Sys.Audit` with the `Sys.Audit` permission and a role called `Datastore.AllocateSpace` with the `Datastore.AllocateSpace` permission only. Apart from these roles, we only need the built-in roles.

* Create a pool for the VMs created and managed by CAPMOX (called `capi` in the example).
* Create a pool for the templates used by CAPMOX (called `templates` in the example) and assign all templates that should be accessible by CAPMOX to this pool.

### Privileges

| Path                                 | Role                    | Propagate |
| ------------------------------------ | ----------------------- | --------- |
| `/`                                  | Sys.Audit               | false     |
| `/nodes`                             | Sys.Audit               | true      |
| `/pool/capi`                         | PVEVMAdmin              | false     |
| `/pool/templates`                    | PVETemplateUser         | false     |
| `/sdn/zones/localnetwork/vmbr0/1234` | PVESDNUser              | false     |
| `/storage/capi_files`                | PVEDataStoreAdmin       | false     |
| `/storage/shared_block`              | Datastore.AllocateSpace | false     |

* In the SDN example, `1234` is the optional VLAN ID if you want to restrict the user to a specific VLAN.
* CAPMOX needs `PVEDataStoreAdmin` on a storage suitable for ISO images for cloud-init. Create a dedicated storage for this (you can use subdirectories in an existing network share for example).
* CAPMOX needs `AllocateSpace` permissions on a storage suitable for disc images. This can be shared with other users as it is only accessed indirectly by cloning/deleting VMs.

## Proxmox TLS communication

The default behavior of the Proxmox API is to skip TLS verification when communicating with the Proxmox API,
The `PROXMOX_INSECURE` environment variable is set to `true` by default in the CAPMOX manager, to skip the verification of the TLS certificate with the Proxmox API.
```
containers:
  - name: manager
    args:
    - --leader-elect
    - --feature-gates=ClusterTopology=true
    - "--metrics-bind-address=localhost:8080"
    - "--v=0"
    env:
    - name: PROXMOX_INSECURE
      value: "true"
```

If you want to use a certificate for communication with the Proxmox API, you can set the `proxmox-root-cert-file` flag variable to the path of the certificate file, and
set the `PROXMOX_INSECURE` environment variable to `false`.

```yaml
containers:
  - name: manager
    args:
    - --leader-elect
    - --feature-gates=ClusterTopology=true
    - "--metrics-bind-address=localhost:8080"
    - "--v=0"
    - "--proxmox-root-cert-file=/var/lib/proxmox/certs/root-ca.pem"
    env:
    - name: PROXMOX_INSECURE
      value: "false"  
    volumeMounts:
    - name: proxmox-root-cert
      mountPath: /var/lib/proxmox/certs
      readOnly: true

volumes:
  - name: proxmox-root-cert
    secret:
      secretName: proxmox-root-cert

```


## Custom Allowed Nodes for ProxmoxMachine

Previously, the Proxmox nodes that will host the Machines are defined in `ProxmoxCluster.spec.allowedNodes`, that config restrict us from placing some set of machines into some specific nodes.
Now, you can also define and override the `allowedNodes` in the ProxmoxMachine, to do so:

```diff
kind: ProxmoxMachineTemplate
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
metadata:
  name: "test-control-plane"
spec:
  template:
    spec:
       sourceNode: "pve"
       templateID: 1000
       format: "qcow2"
       full: true
+      allowedNodes: ["pve-1", "pve-3", "pve-4"]
```

With the following config, you can override what has been set in the Proxmox Cluster, and also you have more flexibility for example you can have a custom allowed Nodes per Machine Deployments.

## Custom Default Network IP Pool for ProxmoxMachine

Like the Allowed Nodes, in the past we couldn't set a custom IP Pool for the default network device, everything was tied to the ProxmoxCluster IP Config.
Now, you can also customize the IP Pool for the default network device, just by setting the `ipv4PoolRef` and/or `ipv6PoolRef`.

```diff
kind: ProxmoxMachineTemplate
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
metadata:
  name: "test-control-plane"
spec:
  template:
    spec:
     sourceNode: "pve"
     templateID: 1000
     format: "qcow2"
     full: true
     network:
       default:
         bridge: ${BRIDGE}
         model: virtio
+        ipv4PoolRef:
+          apiGroup: ipam.cluster.x-k8s.io
+          kind: GlobalInClusterIPPool
+          name: shared-inclusterippool
```

You can set either `ipv4PoolRef` or `ipv6PoolRef` or you can also set them both for dual-stack.
It's up for you also to manage the IP Pool, you can choose a `GlobalInClusterIPPool` or an `InClusterIPPool`.

## Notes

* Clusters with IPV6 only is supported.
* Multiple NICs & Dual-stack setups can be mixed together.
* If you're looking for more customized setups, you can create your own cluster template and use it with the `clusterctl generate cluster` command, by passing it `--from yourtemplate.yaml`.

## API Reference

Please refer to the API reference:
* [CAPMOX API Reference](https://doc.crds.dev/github.com/ionos-cloud/cluster-api-provider-proxmox).
