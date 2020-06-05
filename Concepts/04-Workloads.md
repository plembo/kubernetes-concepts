# Workloads

## Pods

### Pod Overview

#### Understanding Pods

A pod is the smallest deployable object in the Kubernetes object model.

A pod encapsulates an application's container (or multiple containers), storage resources, a unique network identity and options governing how the container should run.

Docker is the most common container runtime used in a Kubernetes pod.

Pods can be used in two ways:

* Pods that run a single container
* Pods that run multipole containers

Each pod is meant to run a single instance of a given application.

![Pod diagram](images/pod.png)


**Networking**

Each pod is assigned a unique IP address. Every container in a pod shares that network namespace, including IP address and network ports. Containers inside a pod see each other all as localhost.

**Storage**

A pod can specify a set of shared storage volumes, that can also persist data.

#### Working with Pods

**Pods and controllers**

Workload resources that manage one or more pods:
* Deployment
* StatefulSet
* DaemonSet

#### Pod Templates
Controllers for workload resources create pods from a pod template and then manage them on your behalf.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello
spec:
  template:
    # This is the pod template
    spec:
      containers:
      - name: hello
        image: busybox
        command: ['sh', '-c', 'echo "Hello, Kubernetes!" && sleep 3600']
      restartPolicy: OnFailure
    # The pod template ends here
```

Modifying a pod template or switching to a new one has no effect on existing pods. Instead, a new pod is created to match the revised (or new) template.

### Pods

#### Motivation for pods

**Management**

**Resource sharing and communication**

#### Uses of pods
Pods can host vertically integrated application stacks (like LAMP), but they primarily support co-located, co-managed helper programs such as:

* content management systems, file and data loaders, local cache managers
* log and checkpoint backup, compression, rotation, snapshotting
* data change watchers, log tailers, logging and monitoring adapters, event publishers
* proxies, bridges and adapters
* controllers, managers, configurators, and updaters

#### Alternatives considered
_Why not just run multiple programs in a single (Docker) container?_

* Transparency
* Decoupling dependencies
* Ease of use
* Efficiency

#### Durability (or not) of pods
Pods not intended to be treated as durable. The do not survive: scheduling, node failures or other evictions.

Users should not create pods directly. They should always use controllers.

Contollers provide self-healing within a cluster scope, as well as replication and rollout management.

A pod is exposed to facilitate:

* scheduler and controller plugability
* support for pod-level ops
* decoupling of pod lifetime from controller lifetime
* decoupling of controllers and services
* clean composition of kublet-level functionality within cluster
* high availability apps

#### Termination of Pods
Pod processes should be allowed to gracefully terminate rather than violently killed.

Note flow is different if one of pod's containers has a [preStop hook](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/#hook-details).

By default all deletes (```kubectl delete```) are graceful within 30 seconds. ```kubectl delete``` supports ```--grace-period=<seconds>``` option. To kill immediately, use ```--force``` with ```grace-period=0```.

Force deletions can potentially be dangerous for some pods and should be performed with caution. Refer to doc for [deleting Pods from a StatefulSet](https://kubernetes.io/docs/tasks/run-application/force-delete-stateful-set-pod/).

#### Privileged mode for pod containers
Any container in a pod can enable privileged mode by using the ```privileged``` flag in the [security context](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/#hook-details) of the container spec. This is used where container wants to invoke Linux features like network manipulation or accessing devices.

#### API Object
Pod is a top-level resource in the Kubernetes API

### Pod Lifecycle

#### Pod phase

* Pending
* Running
* Succeeded
* Failed
* Unknown

#### Pod conditions
* ```lastProbeTime```
* ```lastTransitionTime```
* ```message```
* ```reason```
* ```status```
* ```type``` is a string with one of these possible values:
    * ```podScheduled```
    * ```Ready```
    * ```Initialized```
    * ```ContainersReady```

#### Container Probes
To perform a probe (diagnostic), kubelet calls a Handler:

* ExecAction
* TCPSocketAction
* HTTPGetAction

Probes have one of 3 results:
* Success
* Failure
* Unknown

Three additional kinds of probe:
* livenessProbe
* readinessProbe
* startupProbe

**When to use a liveness probe**

If process in container is about to crash on its own, pod's restartPolicy should handle. But if you want to kill and restart if a probe fails, specify liveness _and_ a restartPolicy of Always or OnFailure.

**When to use a readiness probe**

If you'd like to start sending to a pod only when probe succeeds, use a readiness probe.

Also if you want container to take itself down for maintenance.

Requests will be automatically drained when pod is deleted _without_ a readiness probe, as  on deletion pod puts itself in an unready state.

**When to use a startup probe**

If container usually starts in more than ```initialDeplaySeconds``` + ```failureThreshold``` x ```periodSeconds```, use a startup probe.

#### Pod and Container status

#### Container States
* Waiting
* Running
* Terminated
*
#### Pod readiness

```yaml
kind: Pod
...
spec:
  readinessGates:
    - conditionType: "www.example.com/feature-1"
status:
  conditions:
    - type: Ready                              # a built in PodCondition
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
    - type: "www.example.com/feature-1"        # an extra PodCondition
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
  containerStatuses:
    - containerID: docker://abcd...
      ready: true
...
```
**Status for Pod readiness**

Ready only when both:
* All containers ready
* All conditions in ```ReadinessGates``` are ```True```

When a pod's containers are Ready but one custom condition is missing/False, the kublet will set the pod to ```ContainersReady```.

#### Restart policy
* Always
* On Failure
* Never

Default is Always.

#### Pod lifetime
Pods remain until a human or controller explicitly removes them. Control plane cleans up terminated pods when number exceeds the configured threshold (```terminated-pod-gc-threshold``` in kube-controller-manager).

Pod creation resources:
* Deployment, ReplicaSet or StatefulSet
* Job
* DaemonSet

All workload resources have a PodSpec.

If a node dies or is disconnected, Kubernetes sets the phase of all pods on the lost node to ```Failed```.

#### Examples
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - args:
    - /server
    image: k8s.gcr.io/liveness
    livenessProbe:
      httpGet:
        # when "host" is not defined, "PodIP" will be used
        # host: my-host
        # when "scheme" is not defined, "HTTP" scheme will be used. Only "HTTP" and "HTTPS" are allowed
        # scheme: HTTPS
        path: /healthz
        port: 8080
        httpHeaders:
        - name: X-Custom-Header
          value: Awesome
      initialDelaySeconds: 15
      timeoutSeconds: 1
    name: liveness
```
### Init Containers
A pod can have multiple containers running apps, but can have one or more init containers that run before the app containers are started.

Init containers are exactly like regular containers, except:
* Init containers always run to completion
* Each init container must complete successfully before the next one starts

#### Using init containers
* Init containers can contain utilities or custom code for setup not present in the app image
* Run with a different view, for example access to Secrets other app containers cannot
* Block or delay app container startup until preconditions met
* Securely run utilities or custom code that would otherwise make a container less secure

#### Examples
* Wait for a service to be created, using a shell one-liner
* Register pod with a remote server with an API call
* Wait for some time before starting the app container
* Clone a git repo to a volume
* Place values into a config file and run a template to generate a config file

Here a simple pod is defined that has two init containers: (1) waits for myservice, (2) waits for mydb. Once both init containers are up, the pod runs the app container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
```

To run

```
kubectl apply -f myapp.yaml
```

Check status

```
kubectl get -f myapp.yaml
```

For details

```
kubectl describe -f myapp.yaml
```

Logs

```
kubectl logs myapp-pod -c init-myservice
kubectl logs myapp-pod -c init-mydb
```

Config to make services the init containers are waiting to discover:

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: myservice
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
---
apiVersion: v1
kind: Service
metadata:
  name: mydb
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9377
```

To create services

```
kubectl apply -f services.yaml
```

Then confirm init containers complete

```
kubectl get -f myapp.yaml
```

#### Rules for resource usage
* highest resource request or limit on all init containers is effective init request/limit
* Pod’s effective request/limit for a resource is the higher of:
    * the sum of all app containers request/limit for a resource
    * the effective init request/limit for a resource
* Scheduling done based on effective requests/limits
* Both init and app containers share the Pod's effective QoS tier

#### Pod restarts
* User updates pod spec
* Pod infrastructure container is restarted
* All containers in a pod terminated while restartPolicy set to Always

### Pod Preset
In v1.6 [alpha] Pod presets are objects for injecting information into pods at creation time. This info can include secrets, volumes, volume mounts and environment variables.

See [Injecting data into a Pod using PodPreset](https://kubernetes.io/docs/tasks/inject-data-application/podpreset/).

### Pod Topology Spread Constraints
In v1.18 [beta] you can use topology spread constraints to control how Pods are spread across your cluster among failure-domains such as regions, zones, nodes, and other user-defined topology domains. This can help to achieve high availability as well as efficient resource utilization.

For an incoming pod to be evenly spread within existing pods across zones:

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: mypod
  labels:
    foo: bar
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        foo: bar
  containers:
  - name: pause
    image: k8s.gcr.io/pause:3.1
```

To spread on both zone and node:

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: mypod
  labels:
    foo: bar
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        foo: bar
  - maxSkew: 1
    topologyKey: node
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        foo: bar
  containers:
  - name: pause
    image: k8s.gcr.io/pause:3.1
```

#### Cluster-level default constraints
In v1.18 [alpha] it is possible to set default topology spread constraints for a cluster.

```yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha2
kind: KubeSchedulerConfiguration

profiles:
  pluginConfig:
    - name: PodTopologySpread
      args:
        defaultConstraints:
          - maxSkew: 1
            topologyKey: failure-domain.beta.kubernetes.io/zone
            whenUnsatisfiable: ScheduleAnyway
```

#### Comparison with PodAffinity/PodAntiAffinity
In Kubernetes, directives related to “Affinity” control how Pods are scheduled - more packed or more scattered.

### Disruptions
See complete [guide](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/).

#### Voluntary and involuntary Disruptions
Pods do not disappear until someone (a person or a controller) destroys them, or there is an unavoidable hardware or system software error.

We call these unavoidable cases _involuntary disruptions_ to an application.

We call other cases _voluntary disruptions_. These include both actions initiated by the application owner and those initiated by a Cluster Administrator. 

#### Dealing with Disruptions
* Ensure your pod requests the resources it needs
* Replicate your application if you need high availability
* For even higher availability, spread apps across racks (using anti-affinity) or zones (if using a multi-zone cluster)

#### How Disruption Budgets work
In v1.5 [beta], an application owner can create a PodDisruptionBudget object (PDB) for each application. A PDB limits the number of pods of a replicated application that are down simultaneously from voluntary disruptions. 

PDBs cannot prevent involuntary disruptions from occurring, but they do count against the budget.

#### Separating Cluster Owner and Application Owner Roles
Often, it is useful to think of the Cluster Manager and Application Owner as separate roles with limited knowledge of each other. This separation of responsibilities may make sense in these scenarios:
* when there are many application teams sharing a Kubernetes cluster, and there is natural specialization of roles
* when third-party tools or services are used to automate cluster management

#### How to perform Disruptive Actions on your Cluster
If you are a Cluster Administrator, and you need to perform a disruptive action on all the nodes in your cluster:
* Accept downtime during upgrade
* Failover to another complete replica cluster
* Write disruption tolerant apps and use PDBs

### Ephemeral Containers
In v1.16 [alpha]. Ephemeral containers are a special type of container that runs temporarily in an existing pod to accomplish user-initiated actions like troubleshooting.

Need to describe the container in an EphemeralContainers list:

```yaml
{
    "apiVersion": "v1",
    "kind": "EphemeralContainers",
    "metadata": {
            "name": "example-pod"
    },
    "ephemeralContainers": [{
        "command": [
            "sh"
        ],
        "image": "busybox",
        "imagePullPolicy": "IfNotPresent",
        "name": "debugger",
        "stdin": true,
        "tty": true,
        "terminationMessagePolicy": "File"
    }]
}
```
To update containers in the already running pod:

```
kubectl replace --raw /api/v1/namespaces/default/pods/example-pod/ephemeralcontainers -f ec.json
```
Response:

```yaml
{
   "kind":"EphemeralContainers",
   "apiVersion":"v1",
   "metadata":{
      "name":"example-pod",
      "namespace":"default",
      "selfLink":"/api/v1/namespaces/default/pods/example-pod/ephemeralcontainers",
      "uid":"a14a6d9b-62f2-4119-9d8e-e2ed6bc3a47c",
      "resourceVersion":"15886",
      "creationTimestamp":"2019-08-29T06:41:42Z"
   },
   "ephemeralContainers":[
      {
         "name":"debugger",
         "image":"busybox",
         "command":[
            "sh"
         ],
         "resources":{

         },
         "terminationMessagePolicy":"File",
         "imagePullPolicy":"IfNotPresent",
         "stdin":true,
         "tty":true
      }
   ]
}
```
View new container with

```
kubectl describe pod example-pod
```

```
...
Ephemeral Containers:
  debugger:
    Container ID:  docker://cf81908f149e7e9213d3c3644eda55c72efaff67652a2685c1146f0ce151e80f
    Image:         busybox
    Image ID:      docker-pullable://busybox@sha256:9f1003c480699be56815db0f8146ad2e22efea85129b5b5983d0e0fb52d9ab70
    Port:          <none>
    Host Port:     <none>
    Command:
      sh
    State:          Running
      Started:      Thu, 29 Aug 2019 06:42:21 +0000
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:         <none>
...
```

## Controllers

### ReplicaSet

A ReplicaSet’s purpose is to maintain a stable set of replica Pods running at any given time.

#### When to use
Deployments (which can create and manage ReplicaSets) are preferred over directly created ReplicaSets because deployments provide declarative updates to pods.

#### Example

controllers/frontend.yaml

```json
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  # modify replicas according to your case
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
```

```
kubectl apply -f https://kubernetes.io/examples/controllers/frontend.yaml
```
Get deployed ReplicaSets:

```
$ kubectl get rs
NAME       DESIRED   CURRENT   READY   AGE
frontend   3         3         3       6s
```

Check state of ReplicaSet:

```
$ kubectl describe rs/frontend
```

Check for pods up:

```
$ kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
frontend-b2zdv   1/1     Running   0          6m36s
frontend-vcmts   1/1     Running   0          6m36s
frontend-wtsmm   1/1     Running   0          6m36s
```
Verify owner of pods is set to the frontend ReplicaSet:

```
$ kubectl get pods frontend-b2zdv -o yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2020-02-12T07:06:16Z"
  generateName: frontend-
  labels:
    tier: frontend
  name: frontend-b2zdv
  namespace: default
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: frontend
    uid: f391f6db-bb9b-4c09-ae74-6a1f77f3d5cf
...
```
#### Non-Template Pod acquisitions
A ReplicaSet is not limited to owning Pods specified by its template– it can acquire other Pods in the manner specified in the previous sections, so make sure bare Pods do not have labels that match one of your ReplicaSets.

#### Writing a ReplicaSet manifest
As with all other Kubernetes API objects, a ReplicaSet needs the apiVersion, kind, and metadata fields.

#### Working with ReplicaSets
To delete, use ```kubectl delete```. The [Garbage Collector](https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/) automatically deletes all dependent pods.

Over the REST API or client-go, you must set ```propagationPolicy``` to Background or Foreground in the -d option.

```bash
$ kubectl proxy --port=8080 curl -X DELETE  \
'localhost:8080/apis/apps/v1/namespaces/default/replicasets/frontend' \
-d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Foreground"}' \
-H "Content-Type: application/json"
```

You can delete a ReplicaSet without affecting any of its pods by using the ```--cascade=false``` option.

```bash
$ kubectl proxy --port=8080 curl -X DELETE \
'localhost:8080/apis/apps/v1/namespaces/default/replicasets/frontend' \
-d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Orphan"}' \
-H "Content-Type: application/json"
```

Pods can be removed by changing their labels.

ReplicaSets can easily scaled by updating the .spec.replicas field.

ReplicaSets can be targets for [Horizontal Pod Autoscalers](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) (HPA).

controllers/hpa-rs.yaml

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-scaler
spec:
  scaleTargetRef:
    kind: ReplicaSet
    name: frontend
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

```
$ kubectl apply -f https://k8s.io/examples/controllers/hpa-rs.yaml
```
Use kubectl autoscale instead:

```
$ kubectl autoscale rs frontend --max=10 --min=3 --cpu-percent=50
```

#### Alternatives
* Deployment (Recommended)
* Bare Pods
* Job
* DaemonSet
* ReplicationController



### ReplicationController
A ReplicationController ensures that a specified number of pod replicas are running at any one time. 

#### How it works
If there are too many pods, the ReplicationController terminates the extra pods.

#### Example
controllers/replication.yaml

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

Use ```kubectl apply``` to run it.

To list all pods beloging to a ReplicationController:

```bash
$ pods=$(kubectl get pods \
--selector=app=nginx
--output=jsonpath={.items..metadata.name})

$ echo $pods
```

#### Writing a ReplicationController Spec
As with all other Kubernetes config, a ReplicationController needs apiVersion, kind, and metadata fields. It also needs a .spec section.

* Pod Template
* Labels on ReplicationController
* Pod Selector

You can specify how many pods should run concurrently by setting .spec.replicas to the number of pods you would like to have running concurrently

#### Working with ReplicationControllers
Deleting controllers and pods is accomplished with ```kubectl delete```.

For the REST API or go client, the steps need to be done explicitly (scale replicas to 0, wait for pod deletions, then delete the controller).

To delete just the controller, use ```--cascade=false```.

When using the REST APO or go client, just delete the controller object.

#### Common Usage Patterns

* Rescheduling
* Scaling
* Rolling updates
* Multiple release tracks

Multiple ReplicationControllers can sit behind a single service, so that, for example, some traffic goes to the old version, and some goes to the new version.

ReplicationControllers are an obvious fit for replicated stateless servers, but ReplicationControllers can also be used to maintain availability of master-elected, sharded, and worker-pool applications.

#### Alternatives
* Deployment (Recommended)
* ReplicaSet
* Bare Pods
* Job
* DaemonSet

### Deployments
A Deployment provides _declarative_ updates for Pods and ReplicaSets.

You describe a desired state in a Deployment, and the Deployment Controller changes the actual state to the desired state at a controlled rate.

#### Use Case
* Create a deployment to rollout a ReplicaSet
* Declare the new state of the Pods
* Rollback to an earlier deployment revision
* Scale up the deployment to facilitate more load
* Pause the deployment to apply multiple fixes
* Use the status of the deployment as an indicator that a rollout is stuck
* Clean up older ReplicaSets

#### Creating
controllers/nginx-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

Create:

```bash
$ kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml
```

Check:

```bash
$ kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   0/3     0            0           1s
```

Status:

```bash
$ kubectl rollout status deployment.v1.apps/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment.apps/nginx-deployment successfully rolled out
```

For ReplicaSet that's part of deploy:

```bash
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-75675f5897   3         3         3       18s
```

Show the labels:

```bash
$ kubectl get pods --show-labels
NAME                                READY     STATUS    RESTARTS   AGE       LABELS
nginx-deployment-75675f5897-7ci7o   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
nginx-deployment-75675f5897-kzszj   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
nginx-deployment-75675f5897-qqcnn   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
```

The pod-template-hash label is added by the Deployment controller to every ReplicaSet that a Deployment creates or adopts. DO NOT CHANGE IT!

#### Updating
A rollout is triggered when a deployment pod template (.spec.template) is changed.

**Example: Steps update pods from nginx 1.14.2 to 1.16.1 image**

```bash
$ kubectl --record deployment.apps/nginx-deployment \
set image deployment.v1.apps/nginx-deployment \
nginx=nginx:1.16.1
```

or

```bash
$ kubectl set image deployment/nginx-deployment \
nginx=nginx:1.16.1 --record

deployment.apps/nginx-deployment image updated
```

You could also edit the deployment and change .spec.template.spec.containers[0] from nginx:1.14.2 to nginx:1.16.1.

```bash
$ kubectl edit deployment.v1.apps/nginx-deployment

deployment.apps/nginx-deployment edited
```

For status:

```bash
$ kubectl rollout status deployment.v1.apps/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...

...
deployment.apps/nginx-deployment successfully rolled out
```

After success, use ```kubectl get deployments```:

```bash
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           36s
```

**Rollover (multiple updates in-flight)**

If you update a Deployment while an existing rollout is in progress, the Deployment creates a new ReplicaSet as per the update and start scaling that up, and rolls over the ReplicaSet that it was scaling up previously – it will add it to its list of old ReplicaSets and start scaling it down.

**Label selector updates**

It is generally discouraged to make label selector updates and it is suggested to plan your selectors up front.

#### Rolling Back
By default, all of the Deployment’s rollout history is kept in the system so that you can rollback anytime you want.

**Checking Rollout History**

```bash
$ kubectl rollout history deployment.v1.apps/nginx-deployment
deployments "nginx-deployment"
REVISION    CHANGE-CAUSE
1           kubectl apply --filename=https://k8s.io/examples/controllers/nginx-deployment.yaml --record=true
2           kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1 --record=true
3           kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.161 --record=true
```

Details of revision:

```bash
$ kubectl rollout history deployment.v1.apps/nginx-deployment --revision=2
deployments "nginx-deployment" revision 2
  Labels:       app=nginx
          pod-template-hash=1159050644
  Annotations:  kubernetes.io/change-cause=kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1 --record=true
  Containers:
   nginx:
    Image:      nginx:1.16.1
    Port:       80/TCP
     QoS Tier:
        cpu:      BestEffort
        memory:   BestEffort
    Environment Variables:      <none>
  No volumes.
```

**Rolling Back***

```bash
$ kubectl rollout undo deployment.v1.apps/nginx-deployment
deployment.apps/nginx-deployment rolled back
```
Could also do this with ```--to-revision``` option:

```bash
$ kubectl rollout undo deployment.v1.apps/nginx-deployment --to-revision=2
```

To check on success, run ```get deployment```, then get details (```kubectl describe deployment nginx-deployment```).

#### Scaling

Scale using the following:

```bash
$ kubectl scale deployment.v1.apps/nginx-deployment --replicas=10
deployment.apps/nginx-deployment scaled
```

**Proportional scaling**

When you or an autoscaler scales a RollingUpdate Deployment that is in the middle of a rollout, the Deployment controller balances the additional replicas in the existing active ReplicaSets in order to mitigate risk. This is called proportional scaling.

#### Pausing or Resuming
You can pause a Deployment before triggering one or more updates and then resume it.

```bash
$ kubectl rollout pause deployment.v1.apps/nginx-deployment
```

```bash
$ kubectl rollout resume deployment.v1.apps/nginx-deployment
```

#### Deployment Status
A Deployment enters various states during its lifecycle. It can be progressing while rolling out a new ReplicaSet, it can be complete, or it can fail to progress.

Progressing when a deployment:
* creates new ReplicaSet
* scaling up to its newest RS
* scaling down to its older RS
* new pods become ready

Monitor with ```kubectl rollout status```.

**Complete Deployment**

Kubernetes marks a Deployment as complete when:
* All replicas have been updated to latest verion specified
* All replicas are available
* No old replicas running

Check using ```rollout status```. If completely successful, zero exit code is returned:

```bash
Waiting for rollout to finish: 2 of 3 updated replicas are available...
deployment.apps/nginx-deployment successfully rolled out
$ echo $?
0
```

**Failed Deployment**

Your Deployment may get stuck trying to deploy its newest ReplicaSet without ever completing. Cases:

* insufficient quota
* Readiness probe fails
* Image pull errors
* Insufficient perms
* Limit ranges
* App runtime misconfig

One way you can detect this condition is to specify a deadline parameter in your Deployment spec:

```bash
$ kubectl patch deployment.v1.apps/nginx-deployment \
-p '{"spec":{"progressDeadlineSeconds":600}}'

deployment.apps/nginx-deployment patched
```
Once the deadline has been exceeded, the controller ads a DeploymentCondition to .status.conditions:

* Type=Progressing
* Status=False
* Reason=ProgressDeadlineExceeded

You may experience transient errors, due to a low timeout or some other error.

```bash
<...>
Conditions:
  Type            Status  Reason
  ----            ------  ------
  Available       True    MinimumReplicasAvailable
  Progressing     True    ReplicaSetUpdated
  ReplicaFailure  True    FailedCreate
<...>
```
Once the deadline is exceeded:

```bash
Conditions:
  Type            Status  Reason
  ----            ------  ------
  Available       True    MinimumReplicasAvailable
  Progressing     False   ProgressDeadlineExceeded
  ReplicaFailure  True    FailedCreate
```

If the problem is an insufficient quota, you can scale down other running controllers or increasing the quota. If satisfied:

```bash
Conditions:
  Type          Status  Reason
  ----          ------  ------
  Available     True    MinimumReplicasAvailable
  Progressing   True    NewReplicaSetAvailable
```

Check for progress failure with ```kubectl rollout status```:

```bash
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
error: deployment "nginx" exceeded its progress deadline
$ echo $?
1
```
All actions that apply to a complete Deployment also apply to a failed Deployment.

#### Clean up Policy
Set .spec.revisionHistoryLimit to specify how many old RS to retain. The remainder will be garbage-collected. By default limit is 10.

#### Canary Deployment
To roll out releases to a subset of users or servers using the Deployment.

#### Deployment Spec

**Pod Template**

**Replicas**

**Selector**

**Strategy**
* Recreate Deployment
* Rolling Update Deployment

**Progress Deadline Seconds**

**Min Read Seconds**

**Rollback To**

**Revision History Limit**

**Paused**


### StatefulSets
StatefulSet is the workload API object used to manage stateful applications.

Manages the deployment and scaling of a set of Pods, and provides guarantees about the ordering and uniqueness of these Pods.

#### Using
* Stable, unique network identifiers.
* Stable, persistent storage.
* Ordered, graceful deployment and scaling.
* Ordered, automated rolling updates.

#### Limitations

#### Components

#### Pod Selector
You must set the .spec.selector field of a StatefulSet to match the labels of its .spec.template.metadata.labels

#### Pod Identity
StatefulSet Pods have a unique identity that is comprised of an ordinal, a stable network identity, and stable storage. The identity sticks to the Pod, regardless of which node it’s (re)scheduled on.

#### Deployment and Scaling Guarantees

**Pod Management Policies**

**OrderedReady Pod Management**

**Parallel Pod Management**

#### Update Strategies
In Kubernetes 1.7 and later, StatefulSet’s .spec.updateStrategy field allows you to configure and disable automated rolling updates for containers, labels, resource request/limits, and annotations for the Pods in a StatefulSet.

**Rolling Updates**

**Partitions**

**Forced Rollback**

### DaemonSet
A DaemonSet ensures that all (or some) Nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to them. As nodes are removed from the cluster, those Pods are garbage collected. Deleting a DaemonSet will clean up the Pods it created.

Typical uses:

* running a cluster storage daemon
* running a logs collection daemon on every node
* running a node monitoring daemon on every node,

### Garbage Collection
The role of the Kubernetes garbage collector is to delete certain objects that once had an owner, but no longer have an owner.

#### Owners and dependents
Some Kubernetes objects are owners of other objects. The owned objects are called dependents of the owner object. 

#### Controlling deletion of dependents

**Foreground cascading deletion**

**Background cascading deletion**

**Setting cascading deletion policy**

Deletes dependents in background:

```bash
$ kubectl proxy --port=8080
curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/replicasets/my-repset \
  -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Background"}' \
  -H "Content-Type: application/json"
```

Deletes in foreground:

```bash
$ kubectl proxy --port=8080
curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/replicasets/my-repset \
  -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Foreground"}' \
  -H "Content-Type: application/json"
```

Prior to 1.7, When using cascading deletes with Deployments you must use propagationPolicy: Foreground to delete not only the ReplicaSets created, but also their Pods. If this type of propagationPolicy is not used, only the ReplicaSets will be deleted, and the Pods will be orphaned. 


### TTL Controller for Finished Resources
FEATURE STATE: Kubernetes v1.12 [alpha]
The TTL controller provides a TTL (time to live) mechanism to limit the lifetime of resource objects that have finished execution. TTL controller only handles Jobs for now, and may be expanded to handle other resources that will finish execution, such as Pods and custom resources.


### Jobs Run to Completion
A Job creates one or more Pods and ensures that a specified number of them successfully terminate. As pods successfully complete, the Job tracks the successful completions. When a specified number of successful completions is reached, the task (ie, Job) is complete. Deleting a Job will clean up the Pods it created.

```bash
$ kubectl describe jobs/pi
```

**Controlling Parallelism**

#### Handling Pod and Container Failures

### Job Termination and Cleanup

#### Clean Up Finished Jobs Automatically
Finished Jobs are usually no longer needed in the system. Keeping them around in the system will put pressure on the API server. If the Jobs are managed directly by a higher level controller, such as CronJobs, the Jobs can be cleaned up by CronJobs based on the specified capacity-based cleanup policy.

#### Job Patterns

#### Alternatives
* Bare Pods
* Replication Controller
* Single Job starts Controller

#### Cron Jobs
You can use a CronJob to create a Job that will run at specified times/dates.

### CronJob
FEATURE STATE: Kubernetes v1.8 [beta]
A CronJob creates Jobs on a repeating schedule.

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

#### Limitations
A cron job creates a job object about once per execution time of its schedule.


