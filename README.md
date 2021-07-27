# devops 2.3

## Setup
### Minikube
Created using 'minikube start'. By default, uses Docker as the driver. Docker is installed using 'Docker Desktop for Mac'.

Minikube creates n container called k8s-minikube/kicbase. If you look inside the running container, it has docker installed inside it (trippy!), and this docker instance runs all the containers needed to form the Kubernetes cluster.

### Hyperkit
There are a bunch of issues with the Docker vm-driver. One of those being the ingress addon is not supported. Hence we will use hyperkit as the driver instead: 'minikube start --vm-driver=hyperkit'. HyperKit is an open-source hypervisor for macOS hypervisor, optimized for lightweight virtual machines and container deployment. If Docker for Desktop is installed, you already have HyperKit. Docker for Mac uses HyperKit instead of Virtual Box. Hyperkit is a lightweight macOS virtualization solution built on top of Hypervisor.framework in macOS 10.10 Yosemite and higher. HyperKit is basically a toolkit for embedding hypervisor capabilities in your application.

## Quick Notes & Tips
* High availability is accomplished through fault tolerance and scalability. If either is missing, any failure might have disastrous effects.
* Zero-downtime deployment is a prerequisite for higher frequency releases.
* Never deploy third-party images based on latest tags. By being explicit with the release, we have more control over what is running in production, as well as what should be the next upgrade.
* The number of replicas should not be part of the design. Instead, they are a fluctuating number that changes continuously (or at least often), depending on the traffic, memory and CPU utilization, and so on.
* Docker and many other images use alpine as the base. If you’re not familiar with alpine, it is a very slim and efficient base image, and I strongly recommend that you use it when building your own. Images like debian, centos, ubuntu, redhat, and similar base images are often a terrible choice made because of a misunderstanding of how containers work.
* Uses Docker’s multi-stage builds. E.g. The first stage downloads the dependencies, it runs unit tests, and it builds the binary. The second stage starts over. It builds a fresh image with the go-demo binary copied from the previous stage. For some advantages of milti-stage builds, read this: https://blog.alexellis.io/mutli-stage-docker-builds/
* hostPath is a great solution for accessing host resources like /var/run/docker.sock, /dev/cgroups, and others. That is, as long as the resource we’re trying to reach is on the same node as the Pod.
  * A hostPath Volume maps a directory from a host to where the Pod is running. Using it to 'inject' configuration files into containers would mean that we’d have to make sure that the file is present on every node of the cluster.
* The create command has dash (-) instead of the path to the file. That’s an indication that stdin should be used instead.
* An emptyDir volume is first created when a Pod is assigned to a node, and exists as long as that Pod is running on that node. As the name says, the emptyDir volume is initially empty. All containers in the Pod can read and write the same files in the emptyDir volume, though that volume can be mounted at the same or different paths in each container. When a Pod is removed from a node for any reason, the data in the emptyDir is deleted permanently.
* Secrets are almost the same as ConfigMaps. The main difference is that the secret files are created in tmpfs. Kubernetes secrets do not make your system secure. They are only a step towards such a system. We might want to turn our attention towards third-party Secret managers like HashiCorp Vault.
* The ability to remove a Namespace and all the objects and the resources it hosts is especially useful when we want to create temporary objects. A good example would be continuous deployment (CDP) processes. We can create a Namespace to build, package, test, and do all the other tasks our pipeline requires. Once we’re finished, we can simply remove the Namespace. Otherwise, we would need to keep track of all the objects we created and make sure that they are removed before we terminate the CDP pipeline.
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

### Kube Services
Provides access to Pods from inside the cluster (through Kube Proxy) or from outside the cluster (through Kube DNS).

Each node has a kube-proxy container process. kube-proxy manages forwarding of traffic addressed to the virtual IP addresses (VIPs) of the cluster’s Kubernetes Service objects to the appropriate backend pods.

As of Kubernetes v1.12, CoreDNS is the recommended DNS Server, replacing kube-dns.
### Rolling Back Or Rolling Forward?
Something unexpected will happen. A bug will sneak in and put our production cluster at risk. What should we do in such a case? The answer to that question largely depends on the size of the changes and the frequency of deployments.

If we are using continuous deployment process, we are deploying new releases to production fairly often. Instead of waiting until features accumulate, we are deploying small chunks. In such cases, fixing a problem might be just as fast as rolling back. After all, how much time would it take you to fix a problem caused by only a few hours of work (maybe a day) and that was discovered minutes after you committed? Probably not much. The problem was introduced by a very recent change that is still in engineer’s head. Fixing it should not take long, and we should be able to deploy a new release soon.

Rolling back a release that introduced database changes is often not possible. Even when it is, rolling forward is usually a better option when practicing continuous deployment with high-frequency releases limited to a small scope of changes.

### Ingress Controllers
Ingress objects manage external access to the applications running inside a Kubernetes cluster. While, at first glance, it might seem that we already accomplished that through Kubernetes Services, they do not make the applications truly accessible. We still need forwarding rules based on paths and domains, SSL termination and a number of other features. In a more traditional setup, we’d probably use an external proxy and a load balancer. Ingress provides an API that allows us to accomplish these things, in addition to a few other features we expect from a dynamic cluster.

Unlike other types of Controllers that are typically part of the kube-controller-manager binary, Ingress Controller needs to be installed separately. Instead of a Controller, kube-controller-manager offers Ingress resource that other third-party solutions can utilize to provide requests forwarding and SSL features. In other words, Kubernetes only provides an API, and we need to set up a Controller that will use it.

Fortunately, the community already built a myriad of Ingress Controllers.

### What is the difference between character and block device drivers in UNIX?
They are two main types of devices under all Unix systems:

A Character Device is a device whose driver communicates by sending and receiving single characters (bytes, octets). Example - serial ports, parallel ports, sound cards, keyboard.

A Block Device is a device whose driver communicates by sending entire blocks of data. Example - hard disks, USB cameras, Disk-On-Key.

(Note: Filesystems can only be mounted if they are on block devices.)

### Use ConfigMaps Judiciously
If you have a configuration that is the same across multiple clusters, or if you have only one cluster, all you should do is include it in your Dockerfile and forget it ever existed. When there are no variations of a config, there’s no need to have a configuration file.

Design your applications to use a combination of configuration files and environment variables. Make sure that the default values in a configuration file are sensible and applicable in most use-cases. Bake it into the image. When running a container, declare only the environment variables that represent the differences of a specific cluster. That way, your configuration will be portable and simple at the same time.

### Not So Secretive Secrets
Almost everything Kubernetes needs is stored in etcd62. That includes Secrets. The problem is that they are stored as plain text. Anyone with access to etcd has access to Kubernetes Secrets. We can limit the access to etcd, but that’s not the end of our troubles. etcd stores data to disk as plain text. Restricting the access to etcd still leaves the Secrets vulnerable to who has access to the file system.

That, in a way, diminishes the advantage of storing Secrets in containers in tmpfs. There’s not much benefit of having them in tmpfs used by containers, if those same Secrets are stored on disk by etcd.

Even after securing the access to etcd and making sure that unauthorized users do not have access to the file system partition used by etcd, we are still at risk. When multiple replicas of etcd are running, data is synchronized between them. By default, etcd communication between replicas is not secured. Anyone sniffing that communication could get a hold of our secrets.
## Issues
### Docker Networking
https://docs.docker.com/docker-for-mac/networking/#known-limitations-use-cases-and-workarounds

### There is no docker0 bridge on macOS
Because of the way networking is implemented in Docker Desktop for Mac, you cannot see a docker0 interface on the host. This interface is actually within the virtual machine.

That is, the docker0 bridge is not created on Docker Desktop of Mac, but created inside the docker engine running in the Minikube container running on Docker Desktop for Mac.

### I cannot ping my containers
Docker Desktop for Mac can’t route traffic to containers.

### Per-container IP addressing is not possible
The docker (Linux) bridge network is not reachable from the macOS host.

Because we are using Minikube with the Docker driver, Minikube will not get a host-only IP address. This means we cannot access NodePort services via the Minikube IP e.g. using http://$MINIKUBE_IP:NODE_PORT. If we used the VM driver, then we can, because the VM is exposed to the host system via a host-only IP address.

The workaround is to use a SSH tunnel. Use this command to create a tunnel: minikube service <SERVICE_NAME>

Example output:
```
 minikube service go-demo-2
|-----------|-----------|-------------|---------------------------|
| NAMESPACE |   NAME    | TARGET PORT |            URL            |
|-----------|-----------|-------------|---------------------------|
| default   | go-demo-2 |       28017 | http://192.168.49.2:30001 |
|-----------|-----------|-------------|---------------------------|
🏃  Starting tunnel for service go-demo-2.
|-----------|-----------|-------------|------------------------|
| NAMESPACE |   NAME    | TARGET PORT |          URL           |
|-----------|-----------|-------------|------------------------|
| default   | go-demo-2 |             | http://127.0.0.1:61768 |
|-----------|-----------|-------------|------------------------|
🎉  Opening service default/go-demo-2 in default browser...
❗  Because you are using a Docker driver on darwin, the terminal needs to be open to run it.
```

### Ingress Add On
Due to networking limitations of driver docker on darwin, ingress addon is not supported.
Alternatively to use this addon you can use a vm-based driver: 'minikube start --vm=true'.

## Official Repo
https://github.com/vfarcic/k8s-specs
## Upto
Page 217

Securing Kubernetes Clusters
