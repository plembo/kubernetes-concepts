# Services, Load-balancing, and Networking

## Service
An abstract way to expose an application running on a set of Pods as a network service.

### Motivation
"if some set of Pods provides functionality to other Pods inside your cluster, how do the frontends find out and keep track of which IP address to connect to?"

Services.

### Service Resources
The set of Pods targeted by a Service is usually determined by a selector.

#### Cloud-native service discovery
When using Kubernetes APIs for service discovery, query the API server for endpoints.

Kubernetes can place a network port or load balancer for non-native apps.

### Defining a Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

This specification creates a new Service object named “my-service”, which targets TCP port 9376 on any Pod with the app=MyApp label.

**Note: A Service can map any incoming port to a targetPort. By default and for convenience, the targetPort is set to the same value as the port field.**

Port definitions in Pods have names, and you can reference these names in the targetPort attribute of a Service.

The default protocol for Services is TCP.

#### Services without selectors
Where:

* You want to have an external database cluster in production, but in your test environment you use your own databases.
* You want to point your Service to a Service in a different Namespace or on another cluster.
* You are migrating a workload to Kubernetes. Whilst evaluating the approach, you run only a proportion of your backends in Kubernetes.

Definition:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```
Because this Service has no selector, the corresponding Endpoint object is not created automatically. Manually map the Service to the network address and port where it's running:

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 192.0.2.42
    ports:
      - port: 9376
```
The name of the Endpoints object must be a valid DNS subdomain name.

Kube-proxy does not support loopback, link-local or virtual IPs for endpoints.

### Virtual IPs and service proxies
Every node in a Kubernetes cluster runs a kube-proxy. kube-proxy is responsible for implementing a form of virtual IP for Services of type other than ExternalName.

#### Why not use round-robin DNS?
Because it sucks.

Some real reasons:
* There is a long history of DNS implementations not respecting record TTLs, and caching the results of name lookups after they should have expired.
* Some apps do DNS lookups only once and cache the results indefinitely.
* Even if apps and libraries did proper re-resolution, the low or zero TTLs on the DNS records could impose a high load on DNS that then becomes difficult to manage.

#### User space proxy mode
In this mode, kube-proxy watches the Kubernetes master for the addition and removal of Service and Endpoint objects.

By default, kube-proxy in userspace mode chooses a backend via a round-robin algorithm.

![User Space Overview](services-userspace-overview.png)

#### iptables proxy mode
In this mode, kube-proxy watches the Kubernetes control plane for the addition and removal of Service and Endpoint objects.

By default, kube-proxy in iptables mode chooses a backend at random.

![iptables overview](services-iptables-overview.png)

#### IPVS proxy mode
In ipvs mode, kube-proxy watches Kubernetes Services and Endpoints, calls netlink interface to create IPVS rules accordingly and synchronizes IPVS rules with Kubernetes Services and Endpoints periodically.

IPVS provides more options for balancing traffic to backend Pods; these are:

* rr: round-robin
* lc: least connection (smallest number of open connections)
* dh: destination hashing
* sh: source hashing
* sed: shortest expected delay
* nq: never queu

### Multi-Port Services
Kubernetes lets you configure multiple port definitions on a Service object.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
    - name: https
      protocol: TCP
      port: 443
      targetPort: 9377
```

### Choosing your own IP address
You can specify your own cluster IP address as part of a Service creation request.

### Discovering services
Kubernetes supports 2 primary modes of finding a Service - environment variables and DNS.

For example, the Service "redis-master" which exposes TCP port 6379 and has been allocated cluster IP address 10.0.0.11, produces the following environment variables:

```bash
REDIS_MASTER_SERVICE_HOST=10.0.0.11
REDIS_MASTER_SERVICE_PORT=6379
REDIS_MASTER_PORT=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP_PROTO=tcp
REDIS_MASTER_PORT_6379_TCP_PORT=6379
REDIS_MASTER_PORT_6379_TCP_ADDR=10.0.0.11
```

### DNS
You can (and almost always should) set up a DNS service for your Kubernetes cluster using an add-on.

The Kubernetes DNS server is the only way to access ExternalName Services. 

### Headless Services
Sometimes you don't need load-balancing and a single Service IP. In this case, you can create what are termed “headless” Services, by explicitly specifying "None" for the cluster IP (.spec.clusterIP).

#### With selectors

#### Without selectors


### Publishing Services (ServiceTypes)

Type values and their behaviors are:

* ClusterIP: Exposes the Service on a cluster-internal IP. Choosing this value makes the Service only reachable from within the cluster. This is the default ServiceType.

* NodePort: Exposes the Service on each Node's IP at a static port (the NodePort). A ClusterIP Service, to which the NodePort Service routes, is automatically created. You'll be able to contact the NodePort Service, from outside the cluster, by requesting <NodeIP>:<NodePort>.

* LoadBalancer: Exposes the Service externally using a cloud provider's load balancer. NodePort and ClusterIP Services, to which the external load balancer routes, are automatically created.

* ExternalName: Maps the Service to the contents of the externalName field (e.g. foo.bar.example.com), by returning a CNAME record with its value. No proxying of any kind is set up.

#### Type NodePort
Using a NodePort gives you the freedom to set up your own load balancing solution, to configure environments that are not fully supported by Kubernetes, or even to just expose one or more nodes' IPs directly.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: MyApp
  ports:
      # By default and for convenience, the `targetPort` is set to the same value as the `port` field.
    - port: 80
      targetPort: 80
      # Optional field
      # By default and for convenience, the Kubernetes control plane will allocate a port from a range (default: 30000-32767)
      nodePort: 30007
```

#### Type LoadBalancer
On cloud providers which support external load balancers, setting the type field to LoadBalancer provisions a load balancer for your Service.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  clusterIP: 10.0.171.239
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 192.0.2.127
```

**Internal load balancer**

In a mixed environment it is sometimes necessary to route traffic from Services inside the same (virtual) network address block.

You can achieve this by adding a cloud Service specific annotation to a Service:

GCP:

```yaml
[...]
metadata:
    name: my-service
    annotations:
        cloud.google.com/load-balancer-type: "Internal"
[...]
```


AWS:

```yaml
[...]
metadata:
    name: my-service
    annotations:
        service.beta.kubernetes.io/aws-load-balancer-internal: "true"
[...]
```

Azure:

```yaml
[...]
metadata:
    name: my-service
    annotations:
        service.beta.kubernetes.io/azure-load-balancer-internal: "true"
[...]
```

**TLS Support on AWS**

Add 3 annotations to the LoadBalancer service:

```yaml
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012
```

```yaml
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: (https|http|ssl|tcp)
```

```yaml
    metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
        service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443,8443"
```

From v1.9 on you can use [predefined AWS SSL policies](http://docs.aws.amazon.com/elasticloadbalancing/latest/classic/elb-security-policy-table.html) with HTTPS or SSL listeners for your Services.

Show available policies:

```bash
aws elb describe-load-balancer-policies --query 'PolicyDescriptions[].PolicyName'
```

Then specify chosen policy:

```yaml
    metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy: "ELBSecurityPolicy-TLS-1-2-2017-01"
```
## Service Topology

## Endpoint Slices

## DNS for Services and Pods

## Connecting Applications with Services

## Ingress

## Ingress Controllers

## Network Policies

## Adding entries to Pod /etc/hoss with HostAliases

## IPv4/IPv6 dual-stack

