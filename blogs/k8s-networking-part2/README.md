## Networking in Kubernetes : Part-2 (with Cilium)

I have published structured blog in **medium**. Click [here][https://medium.com/@shreysms31/networking-in-kubernetes-part-2-with-cilium-bc85be2cb3bc] to read.


Before we talk about Cilium, I have to mention eBPF briefly. Cilium uses eBPF to provide secure and observe network connectivity between workloads

- So basically what eBPF allows us to do is, it allows us to run sandbox programs in the Linux Kernel without going back and forth between kernel space and user space, which is what ip tables do Below image shows you the difference between userspace and kernel space.

- So kernel is basically a layer between hardware and your applications. What it does is it can handle networking, memory and process management files and so on. 

- Any of your application mostly run in user-space with unprivilaged mode meaning that you can't directly access or interact with hardware. However what you can do is you can access the kernel functionalites through system calls (sys calls).

- One of the features of this kernel is that it allows us to write custom programs that can get dynamically loaded into the kernel and execute it. This feature which allows us to do this is called BPF or eBPF.

- The kernel also supports BPF hooks in the networking stacks(we'll talk specifically about networking). You can think of a different event points or hooks where you can attach your programs and then those programs will be executed by the kernel when that event happens. So if we talk about network packets instead of like processing the packets through the chains and rules and IP tables in the user space what we can do is we can write a program and attach that program to a specific hook in a kernel so those programs are automatically executed. 
For example a hook when traffic enters or when the traffic is exiting. Instead of writing the rules in the user space and processing them there, you can do it directly in the kernel.

If you look at the diagram below, you can see the crude packet flow, so when the process makes the requests the process will flow through the IP tables to the virtual interface and  the ip tables  on the nodes to reach the nodes interface.

What eBPF allows us to do is it allows us to bypass those iptables that are defined in the userspace and do everything in kernel with attached programs to the kernel hooks.


### Finally Cilium. 

Cilium Features:

-  Connecting Workload setups. Sets up load balancing between services and pods.
- Ability to enforce Network Policies. K8s ships with the Resource called Network Policy, however that resource doesn't have any implementation. So cilium is one of the CNI's that provides the implementation of that netwrok policy resource and allows you to apply policy and cilium is going to enforce them.
- Cilium also provides Visibility into the netwrok packets flow. It has UI component called Hubble. Hubble can be used view these flows.
- Ability to do encryption using ipsec or wireguard.

### Building Blocks of Cilium.

- Cilium agent: They run as a daemonsets. There is one agent per node. Think of it like kube-proxy one per node. That cilium agent talks to kube-api server, it interacts with the cilium kernel.  It loads the eBPF programs, updates the eBPF maps, so maps are the way to store exchange information. It also talks to cilium cni plugin and gets notified whenever a new workload gets created or deleted. Cilium agent is the one who launches the envoy proxy whenever it is needed. There is a L7 Policy that we create, cilium agent is going to make sure to launch an envoy proxy to process the L7 requests.

- CNI pluging: Basically installs the CNI onto the host, configures the nodes cni to use the plugin and talks to the cilium agent using socket.

- Hubble relay: Cluster wide observabiliy, it connects to the gRPC service that each cilium agent provides. And operator that just does like cluster wide one-time operations of installing things.

### Basic concepts of Cilium.

- Cilium Endpoints: And endpoint is collection of containers or applications that share the same IP address, basically a pod. By default when you install Cilium, when you create a workload Cilium is going to assign an ip4 and ip6 address to all the endpoints then you can use either of those addressess to reach that endpoint. Each endpoint will also get an internal ID that's unique within the cluster node and set of labels derives from the pod or the container. So the endpoint gets created whenever the pod is started and that's also kicks off the creation of cilium identity.
So Cilium identity is assigned to all endpoints and this identity is used for enforcing the basic connectivity between endpoints. The identity of the endpoint is derived based on the labels that are assigned to the pod, so if the labels change on the pod, the identity of the endpoints will get updated as well.

#### Good to know:

- IP Address Management (IPAM): IPAM defines how IP addressess get handed out to pods. While we install cilium, we have an option to which ip address management mode we want to use, default one cilium uses is called `cluster scope` which uses cilium node resource where you provide the cider range for specific nodes.
There is also the kubernetes host, CRD-backed, AWS ENI and so on.

- kube-proxy replacement: While installing Cilium, you can specify `kube-proxy replacement mode`. Cilium can either completely replace kube-proxy or it can co-exist with the kube-proxy in case if hte linux kernel doesn't support the full replacement. The default mode in here is `disabled` mode, here everything is deligated to kube-proxy.

### Network Policy:
- Network policy crd ships with k8s, however it doesn't have any impementation. Cilium is here to provide them.
- So Network policy resource allows us to control traffic at L3 and L4, so we can use the selector labels and selector pods and then use ports in our Network Policy.
- In a high level, we can break the resource into 3 parts, `pod selector`, `ingress` and `egress` policies. Using the pod selector we specify the pods we want to apply the policies to.
- Network Policies are `namespace` specify, not cluster wide.

### Cilium Network Policy
What’s Different with Cilium Network Policies?
Cilium supports the standard Kubernetes Network Policy out of the box but extends it with additional features and flexibility using Cilium Network Policies. These policies enable fine-grained control at Layers 3, 4, and 7 of the OSI model, providing more advanced use cases.

Cilium introduces two custom resource definitions (CRDs):
CiliumNetworkPolicy (namespaced)
CiliumClusterwideNetworkPolicy (cluster-wide)
These policies can define connectivity rules based on pod labels, IP/CIDR ranges, DNS names, specific ports, and Layer 7 protocols (e.g., HTTP, Kafka, gRPC). This gives you the ability to apply policies to single pods, groups of pods, namespaces, or the entire cluster.

Cilium also supports host-level policies, extending network security to the nodes themselves, such as blocking SSH access or ICMP pings. Combining Cilium’s network and cluster-wide policies allows for consistent and simplified security management across both pods and nodes.

Additional Enhancements
Encryption with WireGuard or IPSec: Cilium can encrypt traffic between nodes using WireGuard or IPSec, ensuring that traffic remains private and secure.
Advanced Load Balancing: Using eBPF, Cilium can implement highly efficient load balancing mechanisms that avoid the overhead associated with iptables. This enhances the performance and scalability of the cluster, especially in large deployments.
Conclusion
Cilium is an advanced CNI that leverages eBPF to provide high-performance, secure, and observable networking for Kubernetes. It offers several improvements over traditional methods like iptables, including:














