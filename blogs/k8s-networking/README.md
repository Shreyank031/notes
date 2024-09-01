## Networkng in kubernetes

Full structured blog written by me [here](https://medium.com/@shreysms31/networking-in-kubernetes-130c6ac5b616)

1. Container to Container Communication
2. Pod to Pod communication
3. Pod to Service Communication
4. External Communication (Ingress & Egress)

kubernetes doesn't ship with default cni plugin. However managed k8s service like EKS, AKS come with preconfigured `CNI`

1. Container to Container Communication: 
  - Soved by network namespaces: Feature in linux which allows us to have a seperate networking interface and routing table from rest of the system [Host has it's own network namespace and pod can have it's own network namespace]. Which means every pod gets it's own ip address and shared pord space.
  - The containers within that pod within that network namespace they can communicate with eachother through localhost and they share the same ip address and ports.

2. Pod to Pod Communication: 
- The Cluster and Nodes have the ip address and a CIDR range where they can assign ip address from where they can assign ip adresses to the pods. What it does is it gurantees a unique ip address for every pod in the cluster. Now the pod to pod communication happens happens through this addresses.
- A virtual ethernet devices are used to connect two virtual ethernet accross the across the different namespaces.
- The traffic in the below example flows from pod 10.244.1.1 through eth0 and show up on veth interface on the node side and it's gonna go through virtual bridge through routing rules through another virtual interface veth that is connected to the destination pod.
- One of requirements in k8s in networking is the pods must be reachable by there ip addresses not just within the nodes, but also across nodes.
- Now the problem that we need to solve is how we are going to get traffic from eth0 of node 1 to eth0 of another node..

- This is where `container networking interface` [CNI] comes into play. The CNI knows how to setup cross node communication for pods. 
- The CNI runs as daemonsets meaning that one pod is running on each node and that pod on node is responsible for setting up that bridge between the nodes and it allows one pod to `call one pod to another pod using an ip address`

3. Pod to Service Communication:
- The Service here is refering here to kubernetes service. We know that pod's ip addresses are unique. We also know that the nature of pods are ephemeral. We can't relay on pods ip addresses. The way we can solve this is by using k8s `service`.
- When you create a k8s service, it will give you a virtual ip address. And that ip address is backed by one or more pods. What that means is that whenever you send the traffic to that virtual ip, the traffic will get routed to one of the backing pods ips.
- Now you might stuck with one question, How do we assign ip addresses for services. We are going to solve this by following the same approach that we used in pod to pod communication i.e We are going to assign an ip addresses from CIDR range that is typically reserved for services and is different from CIDR range for pods.
- Second question: How do we route the traffic that is sent to the service ip and then we route it to the backing pods ip and third Question is how do we keep track of pods that are backing the Service because we know that pods are ephemeral in nature. That is solved by a kubernetes component called `kube-proxy` with iptables or IPVS(ip virtual server)

- So what is Kube-proxy: High level overview -> It is a controller that connects to kubernetes api server. What it does is it watches for any changes that are made to services and pods. So whenever a kubernetes services gets updated, kube-proxy will go and update the ip table rules so that traffic that sent to the ip to service ip will get correctly routed to one of the backing pods. Pods get created and destroyed, kube-proxy will update the iptables rules to reflect that changes.

- When we send packets/requests from pods to service ip, imagine the destination of service ip is 10.96.1.1, those packets will flow through the iptables rules as part of those rules the destination ip of that service will get changed one of the ips of the backing pods. The iptables rules will make sure that to switch that service ip to the ip of one of the backing pods.

And then on the way back, so when destination pod is returning/sending the response back that destination pod ip will not remain the pod ip anymore, it will be changed back to the service IP so that original pod that made that call or sent the packets thinks that it's recieving the response from the service from service IP instead of one of the backing pods.

4. External Communication (Ingress and Egress):
- The last group of problem we are solving here is "How do we deal with external communication". External meaning any communication/packets that are exiting/incomming the cluster. For eg pod is making a call to an service that leaves out in internet or any packets that are entering the cluster i.e internet to service calls.

One the way out the iptables will make sure that the source IP of the pod gets changed to the internal IP address of the node. Typically when we are running this at cloud hosted cluster like AWS, the nodes have a private IPs and they run inside a VPC network. 
In Order to allow that nodes to access to the internet, we typically need to attach an gateway to that vpc network. And what that internet gateway does is Network Address Translation and it changes the internal node IP to the Public Node IP. So what it does is it allows the responce comming from the internet when it's comming back to be routed back to the node using public IP. And eventually it gets routed back to the original caller and on the way back when the packets are flowing back, similar translations happen but in the reverse order.

Ingress side i.e bringing the traffic inside the cluster in order to do that what we need is a public ip address. The way we do that is by using kubernets load balancer Service. The loadbalancer service allowes us to do is to obtain external ip address.
Behind the scene there is a specific controller that actually creates an actual Load Balancer instance in the cloud and that Load Balancer has an external ip address and we can use that to  send traffic back to the cluste. So when the traffic arrives at the Load Balancer it's gonna get routed to one of the node in the cluster and the ip tables rules on the chosen node will kick in  and they will do the necessary Network Address Translation and they are going to direct the packets to one of the pods that's part of the service.

> The more number of pods our cluster has, the less efficient the communication, because will be more IPtables rules to handle.
> Another downside of using ip talbe is everything has to be processed sequentially, the more you add the slower it is. IP tables alone can be bottleneck especially they don't scale well.

- If you have done any k8s, you know that recommended to have LoadBalancer k8s service for every application you want to expose because you would end up with lot of loadBalancers  with different external IP addresses and you it's going to cost you a lot of money.
So what you idially do is you have a single external IP address and then have an ability to service multiple k8s services or application behind that. So the way you do that is by using the Ingress Resource in a backing Ingress Controller.

### What is an Ingress Controller?
It's just a pod that runs inside a cluster and it watches the changes to the Ingress Resource, so when a new Ingress Resource gets created the Ingress Controller will create the necessary rules to route the traffic to the correct service. You can think of it as a proxy server.
Then when the traffic goes from the Load Balancer, it's not gonna go to your services directly, it will go to ingress controller then that Ingress Controller will route the traffic to the correct k8s Service.

## Kube-proxy: 
- kube-proxy/Network Proxy runs on every node in the cluster which is connected to kube api-server.
- Responsible for keeping the services and backing the endpoints. It also configures and manages the routing rules.
- You can run kube-proxy in 4 modes.
1. userspace: Running a kube-proxy as a web server and then routing all services ips to the web server using ip tables and web server is responsible for proxying to the pods.It's currently depricated.
2. Kernelspace: It's a windows only mode. Yes it is an alternative to userspace mode for k8s in windows.
3. iptables: It is the default mode and solely relies on iptables.
> There was a test that they did: 5000 node cluster with 2000 services and 10 pods each caused atleast 20000 iptables rules on each worker. Theat's a lot of rules and takes a lot of time to process all this. Hence ip tables don't scale well. A more performant alternative is `ipvs` mode.
ipvs supports multiple loadbalancing modes.
- ip tables use random algorithm based on probability to route to one of the backing pods.
- ipvs  supports  `roundrobbin` `least connections` `destination source`, `hasing` and so on.

ipvs still uses ip tables however, when ip tables mode all the changes are made in the ip tables. But when using ipvs mode, the iptables just consult ipvs for any moving parts. All the moving parts are basically stored in ipvs side.









