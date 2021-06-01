# devops 2.3

## Setup
### Minikube
Created using 'minikube start'. By default, uses Docker as the driver. Docker is installed using 'Docker Desktop for Mac'.

Minikube creates n container called k8s-minikube/kicbase. If you look inside the running container, it has docker installed inside it (trippy!), and this docker instance runs all the containers needed to form the Kubernetes cluster.

## Notes
### Immutable vs Mutable Infrastructure
Chef, Puppet, Ansible are designed for mutable infrastructure, that is, they were designed with the idea that servers are brought into the desired state at runtime. Immutable processes, on the other hand, assume that (almost) nothing is changeable at runtime. Artifacts were supposed to be created as immutable images. In case of infrastructure, that meant that VMs are created from images, and not changed at runtime. If an upgrade is needed, new image should be created followed with a replacement of old VMs with new ones based on the new image.

Hence, we got tools capable of building VM images. Today, they are ruled by Packer. Configuration management tools quickly jumped on board, and their vendors told us that they work equally well for configuring images as servers at runtime. However, that was not the case due to the logic behind those tools. They are designed to put a server that is in an unknown state into the desired state. They assume that we are not sure what the current state is. VM images, on the other hand, are always based on an image with a known state. If for example, we choose Ubuntu as a base image, we know what’s inside it. Adding additional packages and configurations is easy. There is no need for things like 'if this then that, otherwise something else.'

Tools like Packer, Terraform, CloudFormation, and the like are the answer to today’s problems.

### Declarative vs Imperative
Instead of using imperative methods to achieve our goals, with schedulers we can be declarative. We can tell a scheduler what the desired state is, and it will do its best to ensure that our desire is (almost) always fulfilled. For example, instead of executing a deployment process five times hoping that we’ll have five replicas of a service, we can tell a scheduler that our desired state is to have the service running with five replicas.

The difference between imperative and declarative methods might seem subtle but, in fact, is enormous. With a declarative expression of the desired state, a scheduler can monitor a cluster and perform actions whenever the actual state does not match the desired. Compare that to an execution of a deployment script. Both will deploy a service and produce the same initial result. However, the script will not make sure that the result is maintained over time. If an hour later, one of the replicas fail, our system will be compromised. Traditionally, we were solving that problem with a combination of alerts and manual interventions. An operator would receive a notification that a replica failed, he’d login to the server, and restart the process. If the whole server is down, the operator might choose to create a new one, or he might deploy the failed replica to one of the other servers. But, before doing that, he’d need to check which server has enough available memory and CPU. All that, and much more, is done by schedulers without human intervention.

### Main Kubernetes Components
Three major components were involved in the process.
The API server is the central component of a Kubernetes cluster and it runs on the master node. Since we are using Minikube, both master and worker nodes are baked into the same virtual machine. However, a more serious Kubernetes cluster should have the two separated on different hosts.

All other components interact with API server and keep watch for changes. Most of the coordination in Kubernetes consists of a component writing to the API Server resource that another component is watching. The second component will then react to changes almost immediately.

The scheduler is also running on the master node. Its job is to watch for unassigned pods and assign them to a node which has available resources (CPU and memory) matching Pod requirements. Since we are running a single-node cluster, specifying resources would not provide much insight into their usage so we’ll leave them for later.

Kubelet runs on each node. Its primary function is to make sure that assigned pods are running on the node. It watches for any new Pod assignments for the node. If a Pod is assigned to the node Kubelet is running on, it will pull the Pod definition and use it to create containers through Docker or any other supported container engine.

The sequence of events that transpired with the kubectl create -f pod/db.yml command is as follows.

1. Kubernetesclient(kubectl)sent a request to the API server requesting creation of a Pod defined in the pod/db.yml file.
2. Since the scheduler is watching the API server for new events, it detected that there is an unassigned Pod.
3. The scheduler decided which node to assign the Pod to and sent that information to the API server.
4. Kubelet is also watching the API server. It detected that the Pod was assigned to the node it is running on.
5. Kubelet sent a request to Docker requesting the creation of the containers that form the Pod. In our case, the Pod defines a single container based on the mongo image.
6. Finally, Kubelet sent a request to the API server notifying it that the Pod was created successfully.

## Upto
Page 61

Since we saved the configuration, we can apply an updated definition
