## POD

Create an NGINX Pod

```bash
kubectl run nginx --image=nginx
```

Generate POD Manifest YAML file (-o yaml). Don't create it(--dry-run)

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml
```

## Deployment

Create a deployment

```bash
kubectl create deployment --image=nginx nginx
```

Generate Deployment YAML file (-o yaml). Don't create it(--dry-run)

```bash
kubectl create deployment --image=nginx nginx --dry-run=client -o yaml
```

Generate Deployment with 4 Replicas

```bash
kubectl create deployment nginx --image=nginx --replicas=4
```

You can also scale a deployment using the kubectl scale command.

```bash
kubectl scale deployment nginx --replicas=4
```

Another way to do this is to save the YAML definition to a file and modify

```bash
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > nginx-deployment.yaml
```

You can then update the YAML file with the replicas or any other field before creating the deployment.



## Service

Create a Service named redis-service of type ClusterIP to expose pod redis on port 6379

```bash
kubectl expose pod redis --port=6379 --name redis-service --dry-run=client -o yaml
```
(This will automatically use the pod's labels as selectors)

Or

```bash
kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml 
```
(This will not use the pods labels as selectors, instead it will assume selectors as app=redis. You cannot pass in selectors as an option. So it does not work very well if your pod has a different label set. So generate the file and modify the selectors before creating the service)


### Create a Service named nginx of type NodePort to expose pod nginx's port 80 on port 30080 on the nodes:

```bash
kubectl expose pod nginx --type=NodePort --port=80 --name=nginx-service --dry-run=client -o yaml
```

(This will automatically use the pod's labels as selectors, but you cannot specify the node port. You have to generate a definition file and then add the node port in manually before creating the service with the pod.)

Or

```bash
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml
```
(This will not use the pods labels as selectors)

Both the above commands have their own challenges. While one of it cannot accept a selector the other cannot accept a node port. I would recommend going with the kubectl expose command. If you need to specify a node port, generate a definition file using the same command and manually input the nodeport before creating the service.

---

```bash

kubectl	expose <pod_name> --port=6379 --name redis-service

kubectl run <pod_name> --image=nginx --port=8080

kubectl run <pod_name> --image=<image_name> --port=8080 --expose=true

```


note: when you use `apply` command in `kubectl apply -f name.yaml` -> The yaml version of cofiguration command is converted into json formate
it compares -> local yaml file , live configuration file and json file for deciding what configuration has to be made.

--------

scheduling pods
nodeName: node01 
nodeName: controlplane

------

### Node Selectors

```bash

kubectl label nodes <node_name> <label_key>=<label_value>

kubectl label nodes node1 size=Large

```

- Now we have labeled a node, lets create a pod.

```yaml

apiVersion:
kind:
metadata:
spec:
    nodeSelector:
        size: Large
#when the pod is created `kubectl create -f <yaml_manifest> it is placed on specified node with the help of labels..i.e `size: Large`
```

#### Limitations of Node Selector.

1. What if "place the pod in small or medium node. Not possible
2. Place the pod in any node, but not small. Cannot be achieved this using Node Selector.

Or and And not possible

For this **Node Affinity** and **Anti Affinity** were introduced

--------

### Node Affinity

Primary purpose of this feature is to ensure that pods in k8s are hosted in particular nodes.

- A bit more complex.

```yaml

spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
            - key: size
              operator: In  # NotIn  and Exists operator
              values: # You can give multiple value, and `in` operator will ensure that the pod will be placed on a node whose label size has any value               with the list of values specified here
              - Large   
              - Medium
```

- What if the nodeAffinity cannot match the labels and values. What if someone change the label on the node at a future point in time

This can be answered with Node Affinity types.

`requiredDuringSchedulingIgnoredDuringExecution:`

-  in cases where a matching node is not found, the scheduler will simply ignore node affinity rules and place the pod on any available node. This can  achieved by 
`prefferedDuringSchedulingIgnoredDuringExecution`

-  value set to ignored, which means pods will continue to run and any changes in node affinity will not impact them once they are scheduled.

- The other one is `prefferedDuringSchedulingRequiredDuringExecution`.
A new option called required during execution is introduced which will evict any pods that are running on nodes that do not meet affinity rules. In the earlier example, a pod running on the large node will be evicted or terminated if the label large is removed from the node.

---------

Combination of `Taints and Tolerations` and `Node Affinity`.



put <user_name> <passwrd> "notebook/path/to/file/"


#### Recource Requirements and limits:

- Kubernetes scheduler that decides which node a pod goes to. The scheduler takes into consideration the amount of resources required by a pod and thos available on the nodes and identifies the best node to place a pod on. In this case, the scheduler schedules a new pod on node two because there are sufficient resources available on that node. If nodes have no sufficient resources available, the scheduler avoids placing the pod on those nodes and instead places the pod on one where sufficient resources are available.

- And if there is no sufficient resources available on any of the nodes, then the scheduler holds back scheduling the pod. And you will see the pod in a pending state. -> `kubectl describe` -> insufficient cpu.

- The minimum amount of CPU or memory requested by the container. So when the scheduler tries to place the pod on a node, it uses these numbers to identify a node which has sufficient amount of resources available.

```yaml

resources:
  requests:
      memory: "2Gi"
      cpu: 2
  limits:
    memory: "4Gi"
    cpu: 4
#So when the scheduler gets a request to place this pod, it looks for a node that has this amount of resources available. And when the pod gets placed on a node, the pod gets a guaranteed amount of resources
```

1 count of CPU = 1vCPU in AWS
1vCPU = 1000millicore

##### Resource Limit

- you can set a limit for the resource usage on these pods. For example, if you set a limit of one vCPU to the containers, a container will be limited to consume only one vCPU from that node.

**A container cannot use more CPU resources than its limit.**

- what happens when a pod tries to exceed resources beyond its specified limit? In case of the CPU, the system throttles the CPU so that it does not go beyond the specified limit.

- A container cannot use more CPU resources than its limit. However, this is not the case with memory. A container can use more memory resources than its limit.
- So if a pod tries to consume more memory than its limit constantly, the pod will be terminated and you'll see that the pod terminated with an `OOM error` in the logs or in the output of the described command when you run it.

##### Default Behaviour

- By default, Kubernetes does not have a CPU or memory request or limit set. So this means that any pod can consume as much resources as required on any node and suffocate other pods or processes that are running on the node of resources.

<img width="928" alt="req_limit" src="https://github.com/Shreyank031/notes/assets/115367978/151f28b0-9641-4b13-8204-96c0b31114a7">

###### By default, Kubernetes does not have resource requests or limits configured for pods. But then how do we ensure that every pod created has some default set?

- This can be achieved using `limit ranges`

- Now this is possible with `limit ranges`. So limit ranges can help you define default values to be set for containers in pods that are created without a request or limit specified in the pod-definition files. This is applicable at the `namespace` level.


```yaml
apiVersion: v1
kind: LimitRange
metadata: 
  name: cpu-resource-constraints
spec:
  limits:
  - default:
      cpu: 500m
    defaultRequest:
      cpu: 500m
    max:
      cpu: 4000m
    min:
      cpu: 100m
    type: Container
```

- If you attempt to create or update an object (Pod or PersistentVolumeClaim) that violates a LimitRange constraint, your request to the API server will fail with an HTTP status code 403 Forbidden and a message explaining the constraint that has been violated.

- LimitRange validations occur only at Pod admission stage, not on running Pods. If you add or modify a LimitRange, the Pods that already exist in that namespace continue unchanged.

- k8s doc [here](https://kubernetes.io/docs/concepts/policy/limit-range/)

##### Resource Quotas:

- Is there any way to restrict the total amount of resources that can be consumed by applications deployed in a Kubernetes cluster? 
- For example, if we had to say that all the pods together shouldn't consume more than this much of CPU or memory what we could do is create quotas at a namespace level. 
- So a resource quota is a namespace level object that can be created to set hard limits for requests and limits.

```yaml

apiVersion: v1
kind: ResourceQuota
metadata:
  name: my_resource_quota
spec:
  hard:
    requests.cpu: 4
    requests.memory: "4Gi"
    limits.cpu: 6
    limits.memory: "8Gi"

```

#### A quick note on editing Pods and Deployments

**Edit a POD**

Remember, you CANNOT edit specifications of an existing POD other than the below.

- spec.containers[*].image

- spec.initContainers[*].image

- spec.activeDeadlineSeconds

- spec.tolerations

For example you cannot edit the environment variables, service accounts, resource limits  of a running pod. But if you really want to, you have 2 options:

1. Run the kubectl edit pod <pod name> command.  This will open the pod specification in an editor (vi editor). Then edit the required properties. When you try to save it, you will be denied. This is because you are attempting to edit a field on the pod that is not editable.



A copy of the file with your changes is saved in a temporary location as shown above.

You can then delete the existing pod by running the command:

kubectl delete pod webapp



Then create a new pod with your changes using the temporary file

kubectl create -f /tmp/kubectl-edit-ccvrq.yaml


2. The second option is to extract the pod definition in YAML format to a file using the command

kubectl get pod webapp -o yaml > my-new-pod.yaml

Then make the changes to the exported file using an editor (vi editor). Save the changes

- vi my-new-pod.yaml

- Then delete the existing pod

- kubectl delete pod webapp

- Then create a new pod with the edited file

- kubectl create -f my-new-pod.yaml


Edit Deployments

With Deployments you can easily edit any field/property of the POD template. Since the pod template is a child of the deployment specification,  with every change the deployment will automatically delete and create a new pod with the new changes. So if you are asked to edit a property of a POD part of a deployment you may do that simply by running the command

`kubectl edit deployment my-deployment`

----------------------------------------------------------------------------------------------------------

### Daemon Sets

Detailed explaination in k8s doc
k8s doc [here](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)

- DaemonSets are like ReplicaSets, as in it helps you deploy multiple instances of pods. But it runs one copy of your pod on each node in your cluster.
- Whenever a new node is added to the cluster, a replica of the pod is automatically added to that node. And when a node is removed the pod is automatically removed.
- The DaemonSet ensures that one copy of the pod is always present in all nodes in the cluster.

- So what are some use cases of DaemonSets? Say you would like to deploy a monitoring agent or log collector on each of your nodes in the cluster, so you can monitor your cluster better. 
- A DaemonSet is perfect for that as it can deploy your monitoring agent in the form of a pod in all the nodes in your cluster. Then you don't have to worry about adding or removing monitoring agents from these nodes when there are changes in your cluster as the DaemonSet will take care of that for you.
 
Usecases of Daemon sets:

- kube-proxy
- Networking soln like -> Vivenet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-daemon
  namespace: kube-systemspec:
  selector:
    matchLabels:
      name: monitoring-agent
  template:
    metadata:
      labels:
        name: monitoring-agent
    spec:
      containers:
        - name: monitoring-agent
          image: monitoring-agent
```

```bash
kubectl create -f <yaml_file.yaml>
kubectl get ds
kubectl describe daemonsets <name>
```

----------------------------------------------------------------------------------------------------------

### STatic PODS:

k8s doc [here](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/)

- Static Pods are managed directly by the kubelet daemon on a specific node, without the API server observing them. Unlike Pods that are managed by the control plane (for example, a Deployment); instead, the kubelet watches each static Pod (and restarts it if it fails).

- The kubelet automatically tries to create a mirror Pod on the Kubernetes API server for each static Pod. This means that the Pods running on a node are visible on the API server, but cannot be controlled from there.

- Pods that are created by the kubelet on its own without the intervention from the API server or rest of the Kubernetes cluster components are known as static Pods.

>You can only create pods this way, but not deployments or rs...by placing defination file in a designated directory.

- The kubelet works at a Pod level and can only understand Pods, which is why it is able to create static Pods this way.

- Now one way to figure out, or you know differentiate a static pod from the other pods is to look at its name and at the end you'll have the node name appended to it. eg,. `kube-controller-manager-controlplane` `kube-scheduler-controlplane`
- or `kubectl get pods/<pod_name> -n kube-system -o yaml`. And look at the `ownerReferences` section. You will find ->
```yaml

ownerReferences:
- apiVersion: v1
  kind: Node #this one
  name: controlplane
```


- path: `/etc/kubernetes/manifest`

```bash
# Run this command on the node where kubelet is running
mkdir -p /etc/kubernetes/manifests/
cat <<EOF >/etc/kubernetes/manifests/static-web.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-web
  labels:
    role: myrole
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - name: web
          containerPort: 80
          protocol: TCP
EOF
```

> `cat /var/lib/kubelet/config.yaml`
> `kubectl get events`

k8s doc [here](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/#configuration-files)

------------------------------------------------------------------------

#### Configure Multiple Schedulers

Kubernetes ships with a default scheduler. If the default scheduler does not suit your needs you can implement your own scheduler. Moreover, you can even run multiple schedulers simultaneously alongside the default scheduler and instruct Kubernetes what scheduler to use for each of your pods.

link for k8s doc [here](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/)

[https://arc.net/l/quote/ymxirdfj](here)

> `kubectl create configmap <config_map_name> --from-file=/path/to/yaml/manifest -n <namespace>`

#### Pod priority

when the pods are created, the pods end up in a scheduling queue. So this is where the pods wait to be scheduled. At this stage, pods are sorted based on the priority defined on the pods.

- Pods can have priority. Priority indicates the importance of a Pod relative to other Pods. If a Pod cannot be scheduled, the scheduler tries to preempt (evict) lower priority Pods to make scheduling of the pending Pod possible.


k8s doc [here](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption)


<img width="1259" alt="scheduling" src="https://github.com/Shreyank031/notes/assets/115367978/21bcb87f-08e4-4a66-82da-7aa51cc56ff2">

- So, to set a priority, you must first create a priority class that looks like this and you should set it a name and set it a priority value.
- In this case, it's set to 1 million so that's really high priority. So, this is how pods with higher priority gets to the beginning of the queue to be scheduled first. 
- And so that sorting happens in this scheduling phase. Then, our pod enters the filter phase. This is where nodes that cannot run the pod are filtered out.

- So, in our case, the first two nodes do not have sufficient resources so do not have 10 CPU remaining so they are filtered out. 
- The next phase is the scoring phase. So this is where nodes are scored with different weights. From the two remaining nodes, the scheduler associates a score to each node based on the free space that it will have after reserving the CPU required for that pod. 
- So, in this case, the first one has two left and the second node will have six left. So, the second node gets a higher score. And so, that's the node that gets picked up. 
- And finally, in the binding phase, this is where the pod is finally bound to a node with the highest score.

```yaml
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  priorityClassName: high-priority
  containers:
  - name: nginx
    image: nginx
```

#### Scheduling plugins and Extension Points

<img width="1339" alt="scheduling_plugin_extension_point" src="https://github.com/Shreyank031/notes/assets/115367978/87a57499-9db8-4736-b919-42b02aca30d2">

-----

#Fever-break and got busy in work


-----

### Logging and Monitoring

 - Node-level metrics, such as the number of nodes in the cluster, how many of them are healthy, as well as performance metrics such as CPU, memory, network, and disc utilization, as well as pod-level metrics, such as the number of pods, and the performance metrics of each pod, such as the CPU and memory consumption on them. So we need a solution that will monitor these metrics, store them, and provide analytics around this data.

- Kubernetes does not come with a full-featured built-in monitoring solution.

- However, there are a number of open-source solutions available today such as Metrics Server, Prometheus, the Elastic Stack,

- The **Metrics Server** retrieves metrics from each of the Kubernetes nodes and pods, aggregates them, and stores them in memory.

- Metrics Server is only an in-memory monitoring solution and does not store the metrics on the disk.

- Kubernetes runs an agent on each node known as the kubelet, which is responsible for receiving instructions from the Kubernetes API master server and running pods on the nodes. 

- The kubelet also contains a sub component known as the cAdvisor or Container Advisor. cAdvisor is responsible for retrieving performance metrics from pods and exposing them through the kubelet API to make the metrics available for the Metrics Server.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/high-availability-1.21+.yaml
```

Troubleshoot the error(I encountered this error): `Warning  Unhealthy  8s (x11 over 98s)  kubelet Readiness probe failed: HTTP probe failed with statuscode: 500`
[here](https://github.com/kubernetes-sigs/metrics-server/issues/812#issuecomment-907586608)


### Rolling updates and Rollbackc

```bash
kubectl rollout status deployment <deployment_name> -n <namespace>
kubectl rollout restart deployment <deployment_name> -n <namespace>
kubectl rollout history deployment <deployment_name> -n <namespace>
kubectl rollout undo deployment/<deployment_name> -n <namespace> # Rolback to the previous state of deployment

```

### ConfigMap in k8s

- A ConfigMap is an API object used to store non-confidential data in key-value pairs. Pods can consume ConfigMaps as environment variables, command-line arguments, or as configuration files in a volume.
- Use a ConfigMap for setting configuration data separately from application code.
- Unlike most Kubernetes objects that have a spec, a ConfigMap has data and binaryData fields
- ConfigMaps and Pods : The Pod and the ConfigMap must be in the same namespace.
- Mounted ConfigMaps are updated automatically

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # property-like keys; each key maps to a simple value
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # file-like keys
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true    
```

>Note: `kubectl replace --force -f /tmp/kubectl-edit-189876778868769.yaml`

k8s ConfigMap doc [here](https://kubernetes.io/docs/concepts/configuration/configmap/)

---

### Secrets in k8s

- A Secret is an object that contains a small amount of sensitive data such as a password, a token, or a key. Such information might otherwise be put in a Pod specification or in a container image. Using a Secret means that you don't need to include confidential data in your application code.
- Because Secrets can be created independently of the Pods that use them, there is less risk of the Secret (and its data) being exposed during the workflow of creating, viewing, and editing Pods. Kubernetes, and applications that run in your cluster, can also take additional precautions with Secrets, such as avoiding writing sensitive data to nonvolatile storage.
- Secrets are similar to ConfigMaps but are specifically intended to hold confidential data.

>Caution: Kubernetes Secrets are, by default, stored unencrypted in the API server's underlying data store (etcd). Anyone with API access can retrieve or modify a Secret, and so can anyone with access to etcd. 
>In order to safely use Secrets, take at least the following steps:
>- Enable Encryption at Rest for Secrets.
>- Enable or configure RBAC rules with least-privilege access to Secrets.
>- Restrict Secret access to specific containers.
>- Consider using external Secret store providers.

You can create multiple secrets:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dotfile-secret
data:
  secret_1: <base64_encoded_secret>
  secret_2: <base64_encoded_secret>
---
#reference it in your pod defination yaml
spec:
  - envFrom:
    - secretRef: 
        name: dotfile-secret
```

#### Use case: dotfiles in a secret volume

You can make your data "hidden" by defining a key that begins with a dot. This key represents a dotfile or "hidden" file. For example, when the following Secret is mounted into a volume, secret-volume, the volume will contain a single file, called .secret-file, and the dotfile-test-container will have this file present at the path /etc/secret-volume/.secret-file.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dotfile-secret
data:
  .secret-file: dmFsdWUtMg0KDQo=
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-dotfiles-pod
spec:
  volumes:
    - name: secret-volume
      secret:
        secretName: dotfile-secret
  containers:
    - name: dotfile-test-container
      image: registry.k8s.io/busybox
      command:
        - ls
        - "-l"
        - "/etc/secret-volume"
      volumeMounts:
        - name: secret-volume
          readOnly: true
          mountPath: "/etc/secret-volume"
```
k8s doc [here](https://kubernetes.io/docs/concepts/configuration/secret/)





