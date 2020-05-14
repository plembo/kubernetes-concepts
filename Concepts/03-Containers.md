# Containers
Containers are for packaging compiled code along with the dependencies needed at run time.

* Container images
* Container runtimes

Container image is a ready-to-run software package. Contains everything needed to run the application (code, runtime, app and system libs, and default values for essential settings).

Images are immutable by design. If changes needed: build new image.

A container runtime is responsible for running containers.

Kubernetes supports several: Docker, containerd, CRI-O, and any implementation of the [Kubernetes CRI](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md).

## Images
Docker image is created and pushed to a registry before being referred to in a K pod.

image property of containers support docker syntax.

### Updating Images
Default policy is ```IfNotPresent```, causing K to skip pulling if an image already exists. To force a pull:

* Set ```imagePullPolicy``` to ```Always```.
* Omit ```imagePullPolicy``` and use ```:latest``` as the image tag.
* Omit ```imagePullPolicy``` and the image tag.
* Enable [AlwaysPullImages](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#alwayspullimages ) admission controller.

Using the ```:latest``` tag is _not_ recommended as a [best practice](https://kubernetes.io/docs/concepts/configuration/overview/#container-images).

### Building Multi-architecture Images with Manifests
Docker CLI supports a [docker manifest](https://docs.docker.com/edge/engine/reference/commandline/manifest/) command with subcommands to build and push manifests. Use ```docker manifest inspect``` to view.

Commands are implemented purely in the Docker CLI. Requires editing $HOME/.docker/config.json and set "experimental" to "enabled" or setting env variable ```DOCKER_CLI_EXPERIMENTAL=enabled```.

Clean up $HOME/.docker/manifests to get rid of stale manifests.

Kubernetes has used the ```-$(ARCH)``` suffix with images. Recommended to continue using suffix for backward compatibility.

### Using a Private Registry
* Google Container Registry (GCR)
* Amazon Elastic Container Registry (ECR)
* Oracle Cloud Infrastructure Registry (OCIR)
* Azure Container Registry (ACR)
* IBM Cloud Container Registry

Each have their own special requirements. [Check documentation](https://kubernetes.io/docs/concepts/containers/images/#using-a-private-registry).

## Container Environment
Container environment provides these resources:
* filesystem
* container info
* cluster object info

_Hostname_ of container is the name of the pod in which it is running (available through ```hostname``` command or gethostname function call).

Pod name and namespace are available as [env variables](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/) (MY_POD_NAME, MY_POD_NAMESPACE).

Services running when container created are avialable as env variables as well. Syntax same as Docker.

Example:

```
FOO_SERVICE_HOST=<host service running on>
FOO_SERVICE_PORT=<port service running on>
```

Services has dedicated IPs and are available via DNS if [DNS addon](http://releases.k8s.io/master/cluster/addons/dns/) enabled.

## Runtime Class
Feature available with Kubernetes v1.14 beta

For selecting the runtime config.

### Motivation
Set different RuntimeClass for different pods to balance performance and security. For example: if workload requires high level of security assurance, you can schedule those pods to run in a container runtime that uses hardware virtualization to benefit from the extra isolation (at the expense of adding to overhead).

### Setup
Enable RuntimeClass feature on apiservers and kubelets, configure and create corresponding resources.

```yaml
apiVersion: node.k8s.io/v1beta1  # RuntimeClass is defined in the node.k8s.io API group
kind: RuntimeClass
metadata:
  name: myclass  # The name the RuntimeClass will be referenced by
  # RuntimeClass is a non-namespaced resource
handler: myconfiguration  # The name of the corresponding CRI configuration
```

### Usage
Specify in the pod spec:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  runtimeClassName: myclass
  # ...
```

### Scheduling
In Kubernetes v1.16 beta:

As of Kubernetes v1.16, RuntimeClass includes support for heterogenous clusters through its ```scheduling``` fields. 

RuntimeClass admission controller must be enabled.

Ensure nodes have a commond label so that it can be selected in the ```runtimeclass.scheduling.nodeSelector``` field.

Kubernetes v1.18 beta includes ability to specify overhead resources in ```overhead``` fields.

Pod Overhead feature gate must be enabled (it is on by default).

## Container Lifecyle Hooks
Lifecyle hooks make containers ware of events in their management lifecycle and run code in a handler when the hook is executed.

Two hooks exposed:

```PostStart``` (executed immediately after container created)

```PreStop``` (executed immediately before container terminated)

Exec performs a specific command, like ```pre-stop.sh``` inside the cgroups and namespaces of the container.

HTTP executes an HTTP request against a specific endpoint on the cluster.

Hook delivery is intended to be _at least once_ (it may be called multiple times for an event).

Logs for hook handlers not exposed in pod events. If the handler fails it will broadcast an event (```FailedPostStartHook``` or ```FailedPreStopHook```).

See these with

```bash
$ kubectl descript pod <podname>
```


