# topology aware routing of services

Author: @m1093782566

## Objective

Figure out a generic way to implement the "local service" route, say "topology aware routing of service". 

Locality is defined by user, it can be any topology-related thing. "Local" means the "same topology level", e.g. same node, same rack, same failure zone, same failure region, same cloud provider etc.

## GOAL

A generic way to support various topology levels, regarless whatever the topology key is.

## Non-goal

Scheduler spreading to implement this sort of topology guarantee.

## Use cases

* Logging agents such as fluentd. Deploy fluentd as DaemonSet and applications only need to communicate with the fluentd in the same node.
* For a sharded service that keeps per-node local information in each shard.
* Authenticating proxies such as [aws-es-proxy](https://github.com/kopeio/aws-es-proxy).
* In container identity wg, being able to give daemonset pods a unique identity per host is on the 2018 plan, and ensuring local pods can communicate to local node services securely is a key goal there. -- from @smarterclayton
* Regional data costs in multi-AZ setup - for instance, in AWS, with a multi-AZ setup, half of the traffic will switch AZ, incurring regional data Transfer costs, whereas if something was local, it wouldn't hit the network.
* Performance benefit (node local/rack local) is lower latency/higher bandwidth.

## Background

It has been a very often requested feature because people need to create a service which connects to local backend Pods, the so called local service. In this way, they can reduce network latency, improve security, save money and so on. However, because topology is arbitrary, zone, region, rack, generator, whatever, who knows? We should allow arbitrary locality.

`ExternalTrafficPolicy` was added in v1.4, but only for NodePort and external LB traffic. NodeName was added to `EndpointAddress` to allow kube-proxy to filter local endpoints for various future purposes.

Based on our experience of advanced routing setup and recent demo of enabling this in Kubernetes service, this document would like to introduce a more generic way to enable local service.

## Proposal

This proposal builds off of earlier requests to [use local pods only for kube-proxy loadbalancing](https://github.com/kubernetes/kubernetes/issues/7433) and [node-local service proposal](https://github.com/kubernetes/kubernetes/pull/28637). But, this document proposes that not only the particular "node-local" user case should be taken care, but also a more generic way should be figured out.

Locality is an "user-defined" thing. When we set topology key "hostname" for service, we expect node carries different node labels on the key "hostname".

Users can control the level of topology. For example, if someone run logging agent as a daemonset, he can set the "hard" topology requirement for same-host. If "hard" is not met, then just return "service not available". 

And if someone set a "soft" topology requirement for same-host, say he "preferred" same-host endpoints and can accept other hosts when for some reasons local service's backend is not available on some host.

If multiple endpoints satisfy the "hard" or "soft" topology requirement, we will randomly pick one by default. Routing decision is expected to be implemented in L3/4 VIP level such as kube proxy.

## Implementation details

### API changes

#### New type ServicePolicy

The user need a way to decalre which service is "local service" and what is the "topology key" of "local service".

This will be accomplished through a new type object `ServicePolicy`.
Endpoint(s) with specify label will be selected by label selector in
`ServicePolicy`, and `ServicePolicy` will declare the topology policy for those endpoints.

`ServicePolicy` will inside a unique namespace, and a single namespace
can own multiple `ServicePoligy`s.

```go
type ServicePolicy struct {
  TypeMeta
  ObjectMeta

  // specification of the topology policy of this ServicePolicy
  Spec TopologyPolicySpec
}

type TopologyPolicySpec struct {
  // ServiceSelector select the service to which this TopologyPolicy object applies.
  // One service only can be selected by single ServicePolicy, in this case, the topology rules are combined additively.
  // This field is NOT optional an empty ServiceSelector will result in err.
  ServiceSelector metav1.LabelSelector `json:"endPointSelector" protobuf:"bytes,1,opt,name=podSelector"`

  // topology is used to achieve "local" service in a given topology level.
  // User can control what ever topology level they want.
  // +optional
  Topology ServiceTopology `json:"topology" protobuf:"bytes,1,opt,name=topology"`
}

// Defines a service topolgoy information.
type ServiceTopology struct {
  // Valid values for mode are "ignored", "required", "preferred".
  // "ignored" is the default value and the associated topology key will have no effect.
  // "required" is the "hard" requirement for topology key and an example would be  “only visit service backends in the same zone”.
  // If the topology requirements specified by this field are not met, the LB, such as kube-proxy will not pick endpoints for the service.
  // "preferred" is the "soft" requirement for topology key and an example would be
  // "prefer to visit service backends in the same rack, but OK to other racks if none match"
  // +optional
  Mode ServicetopologyMode `json:"mode" protobuf:"bytes,1,opt,name=mode"`

  // key is the key for the node label that the system uses to denote
  // such a topology domain. There are some built-in topology keys, e.g.
  // kubernetes.io/hostname, failure-domain.beta.kubernetes.io/zone and failure-domain.beta.kubernetes.io/region etc.
  // The built-in topology keys can be good examples and we recommend users switch to a similar mode for portability, but it's NOT enforced.
  // Users can define whatever topolgoy key they like since topology is arbitrary.
  // +optional
  Key string `json:"key" protobuf:"bytes,2,opt,name=key"`
}
```

An example of `ServicePolicy`:

```yaml
kind: ServicePolicy
metadata:
  name: service-policy-example
  namespace: test
spec:
  serviceSelector:
    matchLabels:
      app: test
  topology:
    key: kubernetes.io/hostname
    mode: required
```

In our example,services in namespace "test" with label "app=test" will be selected, user's request to those services will be routed only to backends in the same host.

#### Endpoints API changes

Although `NodeName` was already added to `EndpointAddress`, we want `Endpoints` to carry more node's topology informations so that allowing more topology levels other than hostname. 

So, create a new `Topology` field in `Endpoints.Subsets.Addresses` for identifying what topology domain the endpoints pod exists, e.g. what host, rack, zone, region etc. In other words, copy the topology-related labels of node hosting the endpoint to `EndpointAddress.Topology`.

```go
type EndpointAddress struct {
  // labels of node hosting the endpoint
  Topology map[string]string
}
```

## Endpoints Controller changes

Endpoint Controller will populate the `Topology` for each `EndpointAddress`. We want `EndpointAddress.Topology` to tell the LB, such as kube-proxy what topology domain(e.g. host, rack, zone, region etc.) the endpoints is in.

Endpoints controller will need to watch ALL nodes and maintain a key-value store for node cache - the key is the nodename and value is the node object. Also, endpoints controller will need to watch ALL ServicePolicy, and maintain a store for service info, which will contains all services' namespacename-labels-topology information.

So, the new logic of endpoint controller might like:

```go
// serviceInfoMap is the map of service with it's information
// serviceNaName is service "namespace/name"
type serviceInfoMap map[serviceNsName]*serviceInfo

// serviceInfo is the service information
// which is used for maintain ServicePolicy information and update endpoint
type serviceInfo struct {

  labels map[string]string

  // the topology of unique service, a service can have multiple topology in case
  // this service is selected by multiple ServicePolicy
  []topology  ServiceTopology
}
go watch service, pod, node, servicePolicy

// update Service topology information
for servicePolicy := range ServicePolicys; do
  //find the service match servicePolicy selectors
  //write service topology info to serviceInfoMap
done

// sync endpoints
for i, pod := range service backends; do
  node := nodeCache[pod.Spec.NodeName]
  // Copy all topology-related labels of node hosting endpoint to endpoint
  // We can only include node labels referenced in the service spec's topology constraints
  endpointAddress := &v1.EndpointAddress {}
  for topologyKey in serviceInfo.topology; do
    endpointAddress.Topology[topologyKey] = node.Labels[topologyKey]
  done
  endpoints.Subsets[i].Addresses = endpointAddress
done
```

## Kube-proxy changes

Kube-proxy will respect topology keys for each service, so kube-proxy on different nodes may create different proxy rules.

Kube-proxy will watch its own node and will find the endpoints that are in the same topology domain as the node if `service.Topology.Mode != ignored`.

The new logic of kube-proxy might like:

```go
go watch node with its nodename
switch service.Topology.Mode {
  case "ignored":
    route request to an endpoint randomly
  case "required":
    endpointsMeetRequirement := make([]endpointInfo, 0)
    topologyKey := service.Topology.Key
    // filter out endpoints that does not meet the "hard" topology requirements
    for i := range service's endpoints.Subsets; do
      ss := endpoints.Subsets[i]
      for j := range ss.Addresses; do
        // check if endpoint are in the same topology domain as the node running kube-proxy
        if ss.Addresses[j].Topology[topologyKey] == node.Labels[topologyKey]; then
          endpointsMeetHardRequirement = append(endpointsMeetHardRequirement, endpoint)
        fi
      done
    done
    // If multiple endpoints match, randomly select one
    if len(endpointsMeetHardRequirement) != 0; then 
      route request to an endpoint in the endpointsMeetHardRequirement randomly
    fi
  case "preferred":
    // Try to find endpoints that meet the "soft" topology requirements firstly,
    // If no one match, kube-proxy tell the kernel all available endpoints and ask it to to route each request randomly to one of them.
}
```
