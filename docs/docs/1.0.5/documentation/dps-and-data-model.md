# DPs and Data Model

When Kuma (`kuma-cp`) runs, it will be waiting for the data plane proxies to connect and register themselves. In order for a data plane proxy to successfully run, there must exist at least one [`Mesh`](../../policies/mesh) in Kuma. By default the system auto-generates a `default` Mesh when the control-plane is run for the first time.

<center>
<img src="/images/docs/0.5.0/diagram-10.jpg" alt="" style="width: 500px; padding-top: 20px; padding-bottom: 10px;"/>
</center>

::: tip
On Universal the [`Dataplane`](#dataplane-entity) entity must be **manually** created and passed to `kuma-dp`, on Kubernetes it is **automatically** created.
:::

## Dataplane Entity

A `Dataplane` entity must be passed to `kuma-dp` when instance attempts to connect to the control-plane. On Kubernetes, this operation is fully **automated**. On Universal, it must be executed **manually**.

To understand why the `Dataplane` entity is required, we must take a step back. As we have explained already, Kuma follow a sidecar proxy model for the data plane proxies, where we have an instance of a data plane proxy for every instance of our services. Each Service and DP will communicate with each other on the same machine, therefore on `127.0.0.1`.

For example, if we have 6 replicas of a "Redis" service, then we must have one instances of `kuma-dp` running alongside each replica of the service, therefore 6 replicas of `kuma-dp` and 6 `Dataplane` entities as well.

<center>
<img src="/images/docs/0.4.0/diagram-11.jpg" alt="" style="width: 500px; padding-top: 20px; padding-bottom: 10px;"/>
</center>

::: tip
**Many DPs!** The number of data plane proxies that we have running can quickly add up, since we have one replica of `kuma-dp` for every replica of every service. That's why it's important for the DP process to be lightweight and consume a few resources, otherwise we would quickly run out of memory, especially on platforms like Kubernetes where multiple services are running on the same underlying host machine. And that's one of the reasons why Kuma leverages Envoy for this task.
:::

When we start a new data plane proxy in Kuma, **two things** have to happen:

1. The data plane proxy needs to advertise what service it is responsible for. This is what the `Dataplane` entity does.
2. The data plane proxy process needs to start accepting incoming and outgoing requests.

To do this, we have to create a file with a `Dataplane` definition and pass it to `kuma-dp run`. This way, data-plane will be registered in the Control Plane and Envoy will start accepting requests.

::: tip
**Remember**: this is all automated if you are running Kuma on Kubernetes!
:::

The registration of the `Dataplane` includes three main sections that are described below in the [Dataplane Specification](#dataplane-specification):

* `address` IP at which this dataplane will be accessible to other dataplanes
* `inbound` networking configuration, to configure on what port the DP will listen to accept external requests, specify on what port the service is listening on the same machine (for internal DP <> Service communication), and the [Tags](#tags) that belong to the service. 
* `outbound` networking configuration, to enable the local service to consume other services.

For example, this is how we start a `Dataplane` for an hypotetical Redis service and then start the `kuma-dp` process:

```sh
$ cat dp.yaml
type: Dataplane
mesh: default
name: redis-1
networking:
  address: 192.168.0.1
  inbound:
  - port: 9000
    servicePort: 6379
    tags:
      kuma.io/service: redis

$ kuma-dp run \
  --cp-address=https://127.0.0.1:5678 \
  --dataplane-file=dp.yaml
  --dataplane-token-file=/tmp/kuma-dp-redis-1-token
```

In the example above, any external client who wants to consume Redis will have to make a request to the DP on address `192.168.0.1` and port `9000`, which internally will be redirected to the Redis service listening on address `127.0.0.1` and port `6379`.

Now let's assume that we have another service called "Backend" that internally listens on port `80`, and that makes outgoing requests to the `redis` service:

```sh
$ cat dp.yaml
type: Dataplane
mesh: default
name: {{ name }}
networking:
  address: {{ address }}
  inbound:
  - port: 8000
    servicePort: 80
    tags:
      kuma.io/service: backend
      kuma.io/protocol: http
  outbound:
  - port: 10000
    tags:
      kuma.io/service: redis

kuma-dp run \
  --cp-address=https://127.0.0.1:5678 \
  --dataplane-file=dp.yaml \
  --dataplane-var name=`hostname -s` \
  --dataplane-var address=192.168.0.2 \
  --dataplane-token-file=/tmp/kuma-dp-backend-1-token
```

In order for the `backend` service to successfully consume `redis`, we specify an `outbound` networking section in the `Dataplane` configuration instructing the DP to listen on a new port `10000` and to proxy any outgoing request on port `10000` to the `redis` service. For this to work, we must update our application to consume `redis` on `127.0.0.1:10000`.

::: tip
You can parametrize your `Dataplane` definition, so you can reuse the same file for many `kuma-dp` instances or even services.
:::

::: tip
As mentioned before, this is only required in Universal. In Kubernetes no change to our applications are required thanks to automated transparent proxying.
:::

## Envoy

`kuma-dp` is built on top of `Envoy`, which has a powerful [Admin API](https://www.envoyproxy.io/docs/envoy/latest/operations/admin) that enables monitoring and troubleshooting of a running dataplane.

By default, `kuma-dp` starts `Envoy Admin API` on the loopback interface (that is only accessible from the local host) and the first available port from the range `30001-65535`.

If you need to override that behaviour, you can use `--admin-port` command-line option or `KUMA_DATAPLANE_ADMIN_PORT` environment variable.

E.g.,

* you can change the default port range by using `--admin-port=10000-20000`
* you can narrow it down to a single port by using `--admin-port=9901`
* you can turn `Envoy Admin API` off by using `--admin-port=`

::: warning
If you choose to turn `Envoy Admin API` off, you will not be able to leverage some of `Kuma` features, such as enabling `Prometheus` metrics on that dataplane.
:::

## Tags

Each Kuma data plane proxy is associated with tags - or attributes - that can be used to both identify the service that the data plane proxy is representing, and they can also be used when configuring the service mesh [policies](/policies) to determine their behavior in a more flexible way.

A tag attributes a qualifier to the data plane proxy, and the tags that are reserved to Kuma are prefixed with `kuma.io` like:

* `kuma.io/service`: Identifies the service name. On Kubernetes this tag is automatically created, while on Universal it must be specified manually.
* `kuma.io/zone`: Identifies the zone name in a [multi-zone deployment](/docs/1.0.5/documentation/deployments/). This tag is automatically created and cannot be overwritten.
* `kuma.io/protocol`: Identifies the protocol that is being exposed by the service and its data plane proxies. Accepted values are `tcp`, `http`, `http2`, `grpc` and `kafka`.

::: tip
The `kuma.io/service` tag must always exist.
:::

## Dataplane Specification

The [`Dataplane`](#dataplane-entity) entity includes the networking and naming configuration that a data-plane proxy (`kuma-dp`) must have attempting to connect to the control-plane (`kuma-cp`).

In Universal mode we must manually create the [`Dataplane`](#dataplane-entity) entity before running `kuma-dp`. A [`Dataplane`](#dataplane-entity) entity can be created with [`kumactl`](#kumactl) or by using the [HTTP API](#http-api). When using [`kumactl`](#kumactl), the regular entity definition will look like:

```yaml
type: Dataplane
mesh: default
name: web-01
networking:
  address: 127.0.0.1
  inbound:
    - port: 11011
      servicePort: 11012
      tags:
        kuma.io/service: backend
        kuma.io/protocol: http
  outbound:
    - port: 33033
      tags:
        kuma.io/service: redis
```
And the [`Gateway mode`](#gateway)'s entity definition will look like:
```yaml
type: Dataplane
mesh: default
name: kong-01
networking:
  address: 10.0.0.1
  gateway:
    tags:
      kuma.io/service: kong
  outbound:
  - port: 33033
    tags:
      kuma.io/service: backend
```

The `Dataplane` entity includes a few sections:

* `type`: must be `Dataplane`.
* `mesh`: the `Mesh` name we want to associate the data-plane with.
* `name`: this is the name of the data-plane instance, and it must be **unique** for any given `Mesh`. We might have multiple instances of a Service, and therefore multiple instances of the sidecar data-plane proxy. Each one of those sidecar proxy instances must have a unique `name`.
* `networking`: this is the meaty part of the configuration. It determines the behavior of the data-plane on incoming (`inbound`) and outgoing (`outbound`) requests.
  * `address` IP address or domain name at which this dataplane will be accessible to other dataplanes. Domain name will be resolved to IP in the control plane.
  * `inbound`: an array of objects that determines what services are being exposed via the data-plane. Each object only supports one port at a time, and you can specify more than one objects in case the service opens up more than one port.
    * `port`: determines the port at which other dataplanes will consume the service
    * `serviceAddress`: IP at which the service is listening. Defaults to `127.0.0.1`. Typical usecase is Universal mode, where `kuma-dp` runs ina  separate netns, container or host than the service. 
    * `servicePort`: determines the port of the service deployed next to the dataplane. This can be omitted if service is exposed on the same port as the dataplane, but only listening on `serviceAddress` or `127.0.0.1` and differs from `networking.address`. 
    * `address`: IP at which inbound listener will be exposed. By default it is inherited from `networking.address`
    * `tags`: each data-plane can include any arbitrary number of tags, with the only requirement that `kuma.io/service` is **mandatory** and it identifies the name of service. You can include tags like `version`, `cloud`, `region`, and so on to give more attributes to the `Dataplane` (attributes that can later on be used to apply policies).
  * `gateway`: determines if the data-plane will operate in Gateway mode. It replaces the `inbound` object and enables Kuma to integrate with existing API gateways like [Kong](https://github.com/Kong/kong). 
    * `tags`: each data-plane can include any arbitrary number of tags, with the only requirement that `kuma.io/service` is **mandatory** and it identifies the name of service. You can include tags like `version`, `cloud`, `region`, and so on to give more attributes to the `Dataplane` (attributes that can later on be used to apply policies).
  * `outbound`: every outgoing request made by the service must also go thorugh the DP. This object specifies ports that the DP will have to listen to when accepting outgoing requests by the service: 
    * `port`: the port that the service needs to consume locally to make a request to the external service
    * `address`: the IP at which outbound listener is exposed. By default it is `127.0.0.1` since it should only be consumed by the app deployed next to the dataplane.
    * `tags`: traffic on `port:address` will be sent to each data-plane that matches those tags. You can put many tags here. However, it is recommended to keep the list short and then use [`TrafficRoute`](../policies/traffic-route.md) for dynamic management of the traffic.

::: tip
On Kubernetes this whole process is automated via transparent proxying and without changing your application's code. On Universal Kuma doesn't support transparent proxying yet, and the outbound service dependencies have to be manually specified in the [`Dataplane`](#dataplane-entity) entity. This also means that in Universal **you must update** your codebases to consume those external services on `127.0.0.1` on the port specified in the `outbound` section.
:::

## Kubernetes

On Kubernetes the data-planes are automatically injected by Kuma as long as the K8s Namespace or Pod are **annotated** with
`kuma.io/sidecar-injection = enabled`, e.g.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kuma-example
  annotations:
    # inject Kuma sidecar into every Pod in that Namespace,
    # unless a user explicitly opts out on per-Pod basis
    kuma.io/sidecar-injection: enabled
```

To opt out of data-plane injection into a particular `Pod`, you need to **annotate** it
with `kuma.io/sidecar-injection = disabled`, e.g.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
  namespace: kuma-example
spec:
  ...
  template:
    metadata:
      ...
      annotations:
        # indicate to Kuma that this Pod doesn't need a sidecar
        kuma.io/sidecar-injection: disabled
    spec:
      containers:
        ...
```

On Kubernetes the [`Dataplane`](#dataplane-entity) entity is also automatically created for you, and because transparent proxying is being used to communicate between the service and the sidecar proxy, no code changes are required in your applications.

## Kubernetes Probes

Kuma natively supports the `httpGet` Kubernetes probes. By default, Kuma overrides the specified probe with a virtual one. For example, if we specify the following probe:

```yaml
livenessProbe:
  httpGet:
    path: /metrics
    port: 3001
  initialDelaySeconds: 3
  periodSeconds: 3
```

Kuma will replace it with:

```yaml
livenessProbe:
  httpGet:
    path: /3001/metrics
    port: 9000
  initialDelaySeconds: 3
  periodSeconds: 3
```

Where `9000` is a default virtual probe port, which can be configured in `kuma-cp.config`:

```yaml
runtime:
  kubernetes:
    injector:
      virtualProbesPort: 19001
```
And can also be overwritten in the Pod's annotations:

```yaml
annotations:
  kuma.io/virtual-probes-port: 19001
```

To disable Kuma's probe virtualziation, we can either set it in Kuma's configuration gile `kuma-cp.config`:

```yaml
runtime:
  kubernetes:
    injector:
      virtualProbesEnabled: true
```

or in the Pod's annotations:

```yaml
annotations:
  kuma.io/virtual-probes: enabled
```

## Gateway

The `Dataplane` can operate in Gateway mode. This way you can integrate Kuma with existing API Gateways like [Kong](https://github.com/Kong/kong).

When you use a Dataplane with a service, both inbound traffic to a service and outbound traffic from the service flows through the Dataplane.
API Gateway should be deployed as any other service within the mesh. However, in this case we want inbound traffic to go directly to API Gateway,
otherwise clients would have to be provided with certificates that are generated dynamically for communication between services within the mesh.
Security for an entrance to the mesh should be handled by API Gateway itself.

Gateway mode lets you skip exposing inbound listeners so it won't be intercepting ingress traffic.

### Universal

On Universal, you can define such Dataplane like this:

```yaml
type: Dataplane
mesh: default
name: kong-01
networking:
  address: 10.0.0.1
  gateway:
    tags:
      kuma.io/service: kong
  outbound:
  - port: 33033
    tags:
      kuma.io/service: backend
```

When configuring your API Gateway to pass traffic to _backend_ set the url to `http://localhost:33033` 

### Kubernetes

On Kubernetes, `Dataplane` entities are automatically generated. To inject gateway Dataplane, mark your API Gateway's Pod with `kuma.io/gateway: enabled` annotation. Here is example with Kong for Kubernetes:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: ingress-kong
  name: ingress-kong
  namespace: kong
spec:
  template:
    metadata:
      annotations:
        kuma.io/gateway: enabled
    spec:
      containers:
        image: kong:1.3
      ...
```

The optimal gateway in Kubernetes mode would be Kong. You can use [Kong for Kubernetes](https://github.com/Kong/kubernetes-ingress-controller) to implement authentication, transformations, and other functionalities across Kubernetes clusters with zero downtime. When integrating [Kong for Kubernetes](https://github.com/Kong/kubernetes-ingress-controller) with Kuma you have to annotate every `Service` that you want to pass traffic to with [`ingress.kubernetes.io/service-upstream=true`](https://github.com/Kong/kubernetes-ingress-controller/blob/master/docs/references/annotations.md#ingresskubernetesioservice-upstream) annotation. Otherwise Kong will do the load balancing which unables Kuma to do the load balancing and apply policies. 

For an in-depth example on deploying Kuma with [Kong for Kubernetes](https://github.com/Kong/kubernetes-ingress-controller), please follow this [demo application guide](https://github.com/kumahq/kuma-demo/tree/master/kubernetes).

## Ingress

To implement cross-zone communication when Kuma is deployed in a [multi-zone](/docs/1.0.5/documentation/deployments/#multi-zone-mode) mode, the `Dataplane` model introduces the `Ingress` mode. Such data plane is not attached to any particular workload, but instead, it is bound to that particular zone.
All the requests that are sent from one zone to another will be directed to the proper instance by the Ingress.
The specifics of the `Ingress` data plane are described in the `networking.ingress` dictionary in the YAML resource.
Ingress has a regular address and one inbound just like a regular data plane, this address is routable within the local Ingress zone. It also has the following public coordinates:
* `networking.ingress.publicAddress` - an IP address or hostname which will be used by data plane proxies from other zones
* `networking.ingress.publicPort` - a port which will be used by data plane proxies from other zones

Ingress that don't have this information is not taken into account when generating Envoy configuration, because they cannot be accessed by data plane proxies from other zones.

:::: tabs :options="{ useUrlFragment: false }"
::: tab "Kubernetes"
The recommended way to deploy an `Ingress` dataplane in Kubernetes is to use `kumactl`, or the Helm charts as specified in [multi-zone](/docs/1.0.5/documentation/deployments/#remote-control-plane). It works as a separate deployment of a single-container pod.

Kuma will try to resolve `networking.ingress.publicAddress` and `networking.ingress.publicPort` automatically by checking the Service associated with this Ingress.

If the Service type is Load Balancer, Kuma will wait for public IP to be resolved. It may take a couple of minutes to receive public IP depending on the LB implementation of your Kubernetes provider. 

If the Service type is Node Port, Kuma will take an External IP of the first Node in the cluster and combine it with Node Port.

You can provide your own public address and port using the following annotations on the Ingress deployment
* `kuma.io/ingress-public-address`
* `kuma.io/ingress-public-port`
:::
::: tab "Universal"

In Universal mode the dataplane resource should be deployed as follows:

```yaml
type: Dataplane
mesh: default
name: dp-ingress
networking:
  address: 192.168.0.1
  ingress:
    publicAddress: 10.0.0.1
    publicPort: 10000 
  inbound:
  - port: 10001
    tags:
      kuma.io/service: ingress
```
::::

Ingress deployment can be scaled horizontally. Many instances can have the same public address and port because they can be put behind one load balancer.

## Direct access to services

By default on Kubernetes data plane proxies communicate with each other by leveraging the `ClusterIP` address of the `Service` resources. Also by default, any request made to another service is automatically load balanced client-side by the data plane proxy that originates the request (they are load balanced by the local Envoy proxy sidecar proxy).

There are situations where we may want to bypass the client-side load balancing and directly access services by using their IP address (ie: in the case of Prometheus wanting to scrape metrics from services by their individual IP address).

When an originating service wants to directly consume other services by their IP address, the originating service's `Deployment` resource must include the following annotation:

```yaml
kuma.io/direct-access-services: Service1, Service2, ServiceN
```

Where the value is a comma separated list of Kuma services that will be consumed directly. For example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
  namespace: kuma-example
spec:
  ...
  template:
    metadata:
      ...
      annotations:
        kuma.io/direct-access-services: "backend_example_svc_1234,backend_example_svc_1235"
    spec:
      containers:
        ...
```

We can also use `*` to indicate direct access to every service in the Mesh:

```yaml
kuma.io/direct-access-services: *
```

::: warning
Using `*` to directly access every service is a resource intensive operation, so we must use it carefully.
:::