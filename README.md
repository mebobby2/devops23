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
* Port 8443 in Apache Tomcat is used for running your service at HTTPS. The default port for HTTPS is 443, so to avoid conflicts it uses 8443 instead of 443 just like 8080 for HTTP instead of 80.
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
Almost everything Kubernetes needs is stored in etcd. That includes Secrets. The problem is that they are stored as plain text. Anyone with access to etcd has access to Kubernetes Secrets. We can limit the access to etcd, but that’s not the end of our troubles. etcd stores data to disk as plain text. Restricting the access to etcd still leaves the Secrets vulnerable to who has access to the file system.

That, in a way, diminishes the advantage of storing Secrets in containers in tmpfs. There’s not much benefit of having them in tmpfs used by containers, if those same Secrets are stored on disk by etcd.

Even after securing the access to etcd and making sure that unauthorized users do not have access to the file system partition used by etcd, we are still at risk. When multiple replicas of etcd are running, data is synchronized between them. By default, etcd communication between replicas is not secured. Anyone sniffing that communication could get a hold of our secrets.

### Authorization Models
#### ACL (access control list)
* 'Subject' can 'Action' to 'Object'
* Based on user and group
#### DAC (discretionary access control)
* 'Subject can' 'Action' to 'Object'
* 'Subject can' 'grant' other 'Subject'
* Based on user and group

The controls are discretionary in the sense that a subject with a certain access permission is capable of passing that permission (perhaps indirectly) on to any other subject.

#### MAC (mandatory access control)
Subjects and objects each have a set of security attributes. Whenever a subject attempts to access an object, an authorization rule enforced by the operating system kernel examines these security attributes and decides whether the access can take place. Any operation by any subject on any object is tested against the set of authorization rules (aka policy) to determine if the operation is allowed.

With mandatory access control, this security policy is centrally controlled by a security policy administrator; users do not have the ability to override the policy and, for example, grant access to files that would otherwise be restricted.

#### RBAC (role based access control)
RBAC differs from access control lists (ACLs), used in traditional discretionary access-control systems, in that it assigns permissions to specific operations with meaning in the organization, rather than to low level data objects. For example, an access control list could be used to grant or deny write access to a particular system file, but it would not dictate how that file could be changed. In an RBAC-based system, an operation might be to ‘create a credit account’ transaction in a financial application or to ‘populate a blood sugar level test’ record in a medical application.

* Subject is a Role which has Permission of Action to Object
* Can implement mandatory access control (MAC) or discretionary access control (DAC).
* (User or group)-role-permission-object

##### Group vs Role
* Group: a collection of users
  * Dino, James and Liam are members of Meifamly Organization
* Role: a collection of permissions
  * Writer is a role, which can create, update articles
  * Role can be applied to user and group.

Example:
```
Permission:
    - Name: write article
    - Operations:
        - Object: Article
          Action: Created
        - Object: Article
          Action: Updated
        - Object: Article
          Action: Read
    - Name: manage article
    - Operations:
        - Object: Article
          Action: Delete
        - Object: Article
          Action: Read
```

#### ABAC (attribute-based access control)
Unlike role-based access control (RBAC), which employs pre-defined roles that carry a specific set of privileges associated with them and to which subjects are assigned, the key difference with ABAC is the concept of policies that express a complex Boolean rule set that can evaluate many different attributes.
* Subject who is xxx can Action to Object which is xxx in Environment
* Concept
  * Policies: bring together attributes to express what can happen and is not allowed.
  * Attributes
    * Subject (age, clearance, department, role, job title)
    * Action (read, delete, view, approve)
    * Resource (the object type (medical record, bank account…), the department, the classification or sensitivity, the location)
    * Contextual (environment) - attributes that deal with time, location or dynamic aspects of the access control scenario
* Standard: XACML (eXtensible Access Control Markup Language)

Example: AWS Resource-Based Policies is a kind of ABAC
```
{
"Version": "2012-10-17",
"Statement": [
    {
        "Effect": "Allow",
        "Action": [
            "ec2:TerminateInstances"
        ],
        "Resource": [
            "*"
        ]
    },
    {
        "Effect": "Deny",
        "Action": [
            "ec2:TerminateInstances"
        ],
        "Condition": {"NotIpAddress": {"aws:SourceIp": [
            "192.0.2.0/24",
            "203.0.113.0/24"
            ]}},
        "Resource": [
            "arn:aws:ec2:<REGION>:<ACCOUNTNUMBER>:instance/*"
        ]
    }
]
}
```

Reference: https://dinolai.com/notes/others/authorization-models-acl-dac-mac-rbac-abac.html

### Stress Testing?
What you need is a way to predict how much resources an application will use in production, not in a simple test environment. You might be inclined to run stress tests that would simulate production setup. Do that. It’s significant, but it does not necessarily result in real production-like behavior.

Replicating production and behavior of real users is tough. Stress tests will get you half-way. For the other half, you’ll have to monitor your applications in production and, among other things, adjust resources accordingly.

Simple test environments do not reflect production usage of resources. Stress tests are a good start, but not a complete solution. Only production provides real metrics.

### Namespaces and Multi-tenancy
Dividing a cluster into Namespaces and employing RBAC is not enough. RBAC prevents unauthorized users from accessing the cluster and provides permissions to those we trust. However, RBAC does not prevent users from accidentally (or intentionally) putting the cluster in danger through too many deployments, too big applications, or inaccurate sizing. Only by combining RBAC with resource quotas and network policies can we hope for a fault tolerant and robust cluster capable of reliably hosting our applications.

### Master Node HA
Every piece of information that enters one of the master nodes is propagated to the others, and only after the majority agrees, that information is committed. If we lose majority (50%+1), masters cannot establish a quorum and cease to operate. If one out of two masters is down, we can get only half of the votes, and we would lose the ability to establish the quorum. Therefore, we need three masters or more. Odd numbers greater than one are “magic” numbers. Always set an odd number greater than one for master nodes.

### Networking
Which networking shall we use? We can choose between kubenet, CNI, classic, and external networking. The classic Kubernetes native networking is deprecated in favor of kubenet, so we can discard it right away. The external networking is used in some custom implementations and for particular use cases, so we’ll discard that one as well.

That leaves us with kubenet and CNI.

Container Network Interface (CNI) allows us to plug in a third-party networking driver. Kops supports Calico, flannel, Canal (Flannel + Calico)100, kopeio-vxlan101, kube-router102, romana103, weave104, and amazon-vpc-routed-eni105 networks. Each of those networks comes with pros and cons and differs in its implementation and primary objectives.

Kubenet is kops’ default networking solution. It is Kubernetes native networking, and it is considered battle tested and very reliable. However, it comes with a limitation. On AWS, routes for each node are configured in AWS VPC routing tables. Since those tables cannot have more than fifty entries, kubenet can be used in clusters with up to fifty nodes. If you’re planning to have a cluster bigger than that, you’ll have to switch to one of the previously mentioned CNIs.

Use kubenet networking if your cluster is smaller than fifty nodes.

### Mix Master & Worker Nodes
It might be worth pointing out that containers that form our applications are always running in worker nodes. Master servers, on the other hand, are entirely dedicated to running Kubernetes system. That does not mean that we couldn’t create a cluster in the way that masters and workers are combined into the same servers, just as we did with Minikube. However, that is risky, and we’re better off separating the two types of nodes. Masters are more reliable when they are running on dedicated servers. Kops knows that, and it does not even allow us to mix the two.

### AWS Volumes
Elastic File System (EFS), has a distinct advantage that it can be mounted to multiple EC2 instances spread across multiple availability zones. It is closest we can get to fault-tolerant storage. Even if a whole zone (datacenter) fails, we’ll still be able to use EFS in the rest of the zones used by our cluster. However, that comes at a cost. EFS introduces a performance penalty. It is, after all, a network file system (NFS), and that entails higher latency.

Elastic Block Store (EBS) is the fastest storage we can use in AWS. Its data access latency is very low thus making it the best choice when performance is the primary concern. The downside is availability. It doesn’t work in multiple availability zones. Failure of one will mean downtime, at least until the zone is restored to its operational state. Unlike EFS, Elastic Block Store cannot span multiple zones.

#### Failover
We have only two worker nodes, distributed in two (out of three) availability zones. If the node that hosted Jenkins failed, we’d be left with only one node. To be more precise, we’d have only one worker node running in the cluster until the auto-scaling group detects that an EC2 instance is missing and recreates it. During those few minutes, the single node we’re left with is not in the same zone. As we already mentioned, each EBS instance is tied to a zone, and the one we mounted to the Jenkins Pod would not be associated with the zone where the other EC2 instance is running. As a result, the PersistentVolume could not re-bound the EBS volume and, therefore, the failed container could not be recreated, until the failed EC2 instance is recreated.

The chances are that the new EC2 instance would not be in the same zone as the one where the failed server was running. Since we’re using three availability zones, and one of them already has an EC2 instance, AWS would recreate the failed server in one of the other two zones. We’d have fifty percent chances that the new EC2 would be in the same zone as the one where the failed server was running. Those are not good odds.

In the real-world scenario, we’d probably have more than two worker nodes. Even a slight increase to three nodes would give us a very good chance that the failed server would be recreated in the same zone. Auto-scaling groups are trying to distribute EC2 instances more or less equally across all the zones. However, that is not guaranteed to happen. A good minimum number of worker nodes would be six.

The more servers we have, the higher are the chances that the cluster is fault tolerant. That is especially true if we are hosting stateful applications. As it goes, we almost certainly have those. There’s hardly any system that does not have a state in some form or another.

If it’s better to have more servers than less, we might be in a complicated position if our system is small and needs, let’s say, less than six servers. In such cases, I’d recommend running smaller VMs. If, for example, you planned to use three t2.xlarge EC2 instances for worker nodes, you might reconsider that and switch to six t2.large servers. Sure, more nodes mean more resource overhead spent on operating systems, Kubernetes system Pods, and few other things. However, I believe that is compensated with bigger stability of your cluster.

#### How about an entire AZ fails?
There is still one more situation we might encounter. A whole availability zone (data center) might fail. Kubernetes will continue operating correctly. It’ll have two instead of three master nodes, and the failed worker nodes will be recreated in healthy zones. However, we’d run into trouble with our stateful services. Kubernetes would not be able to reschedule those that were mounted to EBS volumes from the failed zone. We’d need to wait for the availability zone to come back online, or we’d need to move the EBS volume to a healthy zone manually. The chances are that, in such a case, the EBS would not be available and, therefore, could not be moved.

We could create a process that would be replicating data in (near) real-time between EBS volumes spread across multiple availability zones, but that also comes with a downside. Such an operation would be expensive and would likely slow down state retrieval while everything is fully operational. Should we choose lower performance over high-availability? Is the increased operational overhead worth the trouble? The answer to those questions will differ from one use-case to another.
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
