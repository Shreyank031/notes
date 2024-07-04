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

[req_limit.img] attached

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

kubectl edit deployment my-deployment






