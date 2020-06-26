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
Service Topology enables a service to route traffic based upon the Node topology of the cluster.

If your cluster has Service Topology enabled, you can control Service traffic routing by specifying the topologyKeys field on the Service spec.

Only local endpoints
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "kubernetes.io/hostname"
```

Prefer local endpoints
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "kubernetes.io/hostname"
    - "*"
```

Only zonal or regional endpoints
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "topology.kubernetes.io/zone"
    - "topology.kubernetes.io/region"
```
Prefer node local, zonal, then regional endpoints
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "kubernetes.io/hostname"
    - "topology.kubernetes.io/zone"
    - "topology.kubernetes.io/region"
    - "*"
```


## Endpoint Slices
In Kubernetes, an EndpointSlice contains references to a set of network endpoints. They are a more scalable alternatvie to classic Endpoints.

```yaml
apiVersion: discovery.k8s.io/v1beta1
kind: EndpointSlice
metadata:
  name: example-abc
  labels:
    kubernetes.io/service-name: example
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 80
endpoints:
  - addresses:
      - "10.1.2.3"
    conditions:
      ready: true
    hostname: pod-1
    topology:
      kubernetes.io/hostname: node-1
      topology.kubernetes.io/zone: us-west2-a
```
By default, EndpointSlices managed by the EndpointSlice controller will have no more than 100 endpoints each. Below this scale, EndpointSlices should map 1:1 with Endpoints and Services and have similar performance.

EndpointSlices can act as the source of truth for kube-proxy when it comes to how to route internal traffic. When enabled, they should provide a performance improvement for services with large numbers of endpoints.

The EndpointSlice controller watches Services and Pods to ensure corresponding EndpointSlices are up to date.



## DNS for Services and Pods
Kubernetes DNS schedules a DNS Pod and Service on the cluster, and configures the kubelets to tell individual containers to use the DNS Service's IP to resolve DNS names.

Every Service defined in the cluster (including the DNS server itself) is assigned a DNS name.

Every Service defined in the cluster (including the DNS server itself) is assigned a DNS name.

Currently when a pod is created, its hostname is the Pod's metadata.name value.

The Pod spec also has an optional subdomain field which can be used to specify its subdomain.

DNS policies can be set on a per-pod basis:
* Default. Pod inherits the name resolution configuration from the node that the pods run on.
* ClusterFirst. Any DNS query that does not match the configured cluster domain suffix, such as "www.kubernetes.io", is forwarded to the upstream nameserver.
* ClusterFirstWithHostNet. For Pods running with hostNetwork.
* None. allows a Pod to ignore DNS settings from the Kubernetes environment.

Pod DNS Config:
* nameservers
* searches
* options

service/networking/custom-dns.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
    - name: test
      image: nginx
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 1.2.3.4
    searches:
      - ns1.svc.cluster-domain.example
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
```

## Connecting Applications with Services
By default, Docker uses host-private networking, so containers can talk to other containers only if they are on the same machine.

Kubernetes assumes that pods can communicate with other pods, regardless of which host they land on.

A Kubernetes Service is an abstraction which defines a logical set of Pods running somewhere in your cluster, that all provide the same functionality. When created, each Service is assigned a unique IP address (also called clusterIP). 

Create a pod:

service/networking/run-my-nginx.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
```

```bash
$ kubectl apply -f ./run-my-nginx.yaml
...
$ kubectl get pods -l run=my-nginx -o wide
...
$ kubectl get pods -l run=my-nginx -o yaml | grep podIP
```

Create a Service:

```bash
$ kubectl expose deployment/my-nginx
service/my-nginx exposed
```
Equivalent to ```kubectl apply -f``` to a config like this:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
```
Check service:

```bash
$ kubectl get svc my-nginx
```

A Service is backed by a group of Pods. These Pods are exposed through endpoints. The Service's selector will be evaluated continuously and the results will be POSTed to an Endpoints object also named my-nginx.

```bash
$ kubectl describe svc my-nginx
```

```bash
$ kubectl get ep my-nginx
```
(where "ep" is "endpoint")

Only two primary modes of finding a Service: environment variables and DNS.

Environment variables set for each active service:

```bash
$ kubectl exec my-nginx-3800858182-jr4a2 -- printenv | grep SERVICE
KUBERNETES_SERVICE_HOST=10.0.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
```
Note no mention of service, because replicas were created before service. To
fix, kill Pods and wait for Deployment to recreate.

```bash
$ kubectl scale deployment my-nginx --replicas=0; \
kubectl scale deployment my-nginx --replicas=2;
...
$ kubectl get pods -l run=my-nginx -o wide
```
Pods may have different names, since their killing and recreation:

```bash
$ kubectl exec my-nginx-3800858182-e9ihh -- printenv | grep SERVICE
KUBERNETES_SERVICE_PORT=443
MY_NGINX_SERVICE_HOST=10.0.162.149
KUBERNETES_SERVICE_HOST=10.0.0.1
MY_NGINX_SERVICE_PORT=80
KUBERNETES_SERVICE_PORT_HTTPS=443
```

DNS cluster addon Service available. Check if running:

```bash
$ kubectl get services kube-dns --namespace=kube-system
```

Enable as set out in the [CoreDNS README](https://github.com/coredns/deployment/tree/master/kubernetes):

```bash
$ kubectl run curl --image=radial/busyboxplus:curl -i --tty
Waiting for pod default/curl-131556218-9fnch to be running, status is Pending, pod ready: false
Hit enter for command prompt
```

Then try to dig for my-nginx.

Security requirements:
* Self-signed certificates for https (unless cert already in use)
* Nginx server configured to use certs
* A secret making certs available to pods (PKCS12 cert)

See [https-nginx example](https://github.com/kubernetes/examples/tree/master/staging/https-nginx/).

```bash
$ make keys KEY=/tmp/nginx.key CERT=/tmp/nginx.crt
$ kubectl create secret tls nginxsecret --key /tmp/nginx.key --cert /tmp/nginx.crt
```

```
$ kubectl get secrets
```

Do configmap:

```bash
$ kubectl create configmap nginxconfigmap --from-file=default.conf
configmap/nginxconfigmap created
$ kubectl get configmaps
NAME             DATA   AGE
nginxconfigmap   1      114s
```

```bash
# Create a public private key pair
$ openssl req -x509 -nodes -days 365 \ 
-newkey rsa:2048 -keyout /d/tmp/nginx.key \
-out /d/tmp/nginx.crt -subj "/CN=my-nginx/O=my-nginx"
# Convert the keys to base64 encoding
$ cat /d/tmp/nginx.crt | base64
$ cat /d/tmp/nginx.key | base64
```

YAML file to implement:

```yaml
apiVersion: "v1"
kind: "Secret"
metadata:
  name: "nginxsecret"
  namespace: "default"
type: kubernetes.io/tls
data:
  tls.crt: "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURIekN..."
  tls.key: "LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1..."
```

```bash
$ kubectl apply -f nginxsecrets.yaml
```

## Ingress
An API object that manages external access to the services in a cluster, typically HTTP.

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          serviceName: test
          servicePort: 80
```

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: IngressClass
metadata:
  name: external-lb
spec:
  controller: example.com/ingress-controller
  parameters:
    apiGroup: k8s.example.com/v1alpha
    kind: IngressParameters
    name: external-lb
```

## Ingress Controllers
In order for the Ingress resource to work, the cluster must have an ingress controller running.

Unlike other types of controllers which run as part of the kube-controller-manager binary, Ingress controllers are not started automatically with a cluster. 

Kubernetes as a project currently supports and maintains GCE and nginx controllers.

## Network Policies
A network policy is a specification of how groups of pods are allowed to communicate with each other and other network endpoints.

NetworkPolicy resources use labels to select pods and define rules which specify what traffic is allowed to the selected pods.

Default policies:
* Deny all ingress traffic
* Allow all ingress
* Deny all egress
* Allow all egress
* Deny all ingress and egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```

## Adding entries to Pod /etc/hoss with HostAliases
Adding entries to a Pod's /etc/hosts file provides Pod-level override of hostname resolution when DNS and other options are not applicable. You can add these custom entries with the HostAliases field in PodSpec.

Modification not using HostAliases is not suggested because the file is managed by the kubelet and can be overwritten on during Pod creation/restart.

## IPv4/IPv6 dual-stack
IPv4/IPv6 dual-stack enables the allocation of both IPv4 and IPv6 addresses to Pods and Services.

If you enable IPv4/IPv6 dual-stack networking for your Kubernetes cluster, the cluster will support the simultaneous assignment of both IPv4 and IPv6 addresses.

