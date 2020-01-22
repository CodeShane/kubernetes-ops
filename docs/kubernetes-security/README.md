# Kubernetes Security
This page is here to describe security challenages and possible solutions to various security concerns in a 
Kubernetes deployment.

## Traditional n-tier archtecture
This diagram represents a non-containerized n-tier architecture:

![n-tier-application-architecture](/docs/kubernetes-security/images/n-tier-application-architecture.png)

### [1] Internet
The Internet is the Internet in general.  Items placed here can be reach by any node on the Internet.

### [2] Internal Network
The internal network is a network that has IPs in the [RFC 1918](https://tools.ietf.org/html/rfc1918) ranges.  These IPs are not routable via the internet.
The hosts in this network will need something like a load balancer with a public IP and an private IP in this network to get traffic inbound from the internet.
If these nodes want to connect to other hosts on the Internet, it will need to go through something like a NAT gateway (which will have a public IP and a
private IP) to reach these external hosts.

The internal network is where instances for the application will reside.  Usually not every single machine and not the entire machine needs to be reachable by
the internet to serve out an application.  Usually only the HTTP/HTTPS (80/443) ports needs to be exposed for a web application to function properly.  This means
there is no need to expose the entire machine to the internet and the application can only expose a certain set of ports which makes it much safer because the
attack surface for this application will be only 2 ports.

### [3] Loadbalancer
The load balancer has a public IP address that is reachable anywhere on the internet.  This is the main point of entry for the application.  The load balancer
bridges the external network (Internet) and the internal private network where the application servers resides.

This is a vulnerable point since it is on the internet and anyone can reach it.  The job of the load balancer is to forward traffic from the internet to a set
of redundant servers we have internally that is handling request for the application.  This also means that the "web tier" servers are vulnerable to attacks
from the outside.

### [4] Bastion host
The bastion host has a public IP address that is reachable anywhere on the internet and a private IP address for the internal network.  The bastion host bridges
the external network (Internet) and the internal private network where the application servers resides.

This is a vulnerable point since it is on the internet and anyone can reach it.  While this machine does not blindly forward traffic from the internet into our backend servers, it is most likely sitting on a well known SSH port 22.  We can limit the IPs that can reach this port but it is still sitting on the internet and
it would be something to watch out for.

This host is very important since it is bridging the internet and the internal network and it is a full Linux system.  If someone is able to compromise this host,
they would have a whole suite of tools available to them to further attack the internal network.

### [5] VPN
This might be a duplicate node but wanted to highlight that it might be here.  The VPN will serve about the same purpose as the bastion host.  The VPN has a
public IP address and an internal IP address spanning the two networks.  It has similar vulnerabilities as the bastion host but maybe a little less.  These are
usually appliances which has limited functionality and are hardend for security.  However, this is again just software and over the years, there has been numerous
remotely exploitable vulnerabilities against top VPN vendors.

### [6] Availability zones
In this particular example there are two availability zones.  These are usually physical segments of the network where each availability zone is isolated in a sense where a zone can have it's own power, racks, routers, switches, and servers.  This means that if there is a physical problem or a configuration issue on these set of items, it will only affect this zone.  The other zone(s) should be still good.

In this setup, we are using 2 zones and the load balancer is pointed to both zones and we have the same servers one in each zone for high availability and
redundancy.

### [7] Web tier subnet
The web tier holds the externally facing web servers.  This is the only subnet that the load balancers are pointed to and to specific servers sitting on specific
set of ports.
* This subnet has access control rules (ACLs) to allow traffic from the load balancer to it's servers in this subnet
* Ther are no other inbound traffic that is allowed into this subnet (except for management traffic)

### [8] Business tier
The business tier is just one or more set(s) of subnets and server(s) that does backend work.  These items are not exposed externally that has an external load balancer pointed to it.  This set of workload provides support functionality to the application.  This could be supporting the web tier to retrieve some information or it could be
a set of batch jobs that runs on some interval to crunch data.  

The picture depicts one subnet in each zone but it could be a bunch of different subnets with different types of workload(s) living in it.  The main point
here is that these set of servers are isolated from the internet and has no direct connections.  Further more, it can have only limited connections inbound
to it.  If this set of servers only serves the web tier, then inbound traffic should only be able to come from the web tier.  If there are additional business
tier that does a batch job and it doesnt need any incoming connections and all it needs is to go and fetch data from the database, it might not allow any
incomming connections to it (except for management traffic).

### [9] Datastore
The datastore tier is one or more set of subnets and servers that provide data storage functionality.  These can be databases, object stores, NoSQL clusters, etc.
Since the datastores is where a lot of valuable information is kept, it is usually isolated off and monitored heavily.  It has tight controls on what can connect
to it.

## Traditional n-tier archtecture controle plane
![n-tier-application-architecture control plane](/docs/kubernetes-security/images/n-tier-application-architecture-control-plane.png)

The control plane can be seen as the "thing" that created this setup and controls how it is configured.  This environment can be in a cloud provider
such as AWS, GCP, Azure or on premise with physical machines or it could be on VMware on virtual machines.  We will not go over every permutation
on how this can be deployed.  We are going to go with this is on a cloud provider (AWS, GCP, Azure), they are similar enough to make these generalizations.

The tools depicted here are only one of many such tools that can help you create this setup in a cloud provider.  I wouldn't go as far as saying these are the best ones but they are very popular and used widely.  So, we'll go with these for the sake of just picking some tool that can create this for us.  Most tools will have similar functionality and purpose.  You can pretty much swap out one for another, you might loose something here and there and gain other items but they are pretty much the same.

Terraform, is a tool to help you create infrastructure as code.  You define what "resources" you want in Terraform's own language and you give it access keys to interact with your cloud and you can apply the Terraform template.  Terraform will then go out to your cloud and create what you have defined.  A tool like Terraform is useful because it handles a lot of the underlying development work for you.  You don't have to know how to authenticate with your cloud or even know how to work with your cloud's API to create resources.  Terraform abstracts those things from you to make it easier for your to build infrastructure in a cloud.

Chef, is a configuration management tool to configure servers.  There is a Chef agent running on each of the servers connected to a Chef master to get what configuration should go onto it.  Depending on the servers Chef configuration it can go and get the web server's setup, the backend tier server setup or any other server setup and configuration you might define.  This tool will help you setup the virtual machine and perform actions like setup the host file, install Apache, install and setup Java, or update the machine.  Basically anything you can do on a Linux command line, you can program Chef to do it in an automated way on one or more machines.  This make it easy to control hundreds if not thousands of machine and have them configured just the way you want them.  When you want to change something like the version of Apache or Java running on the machines, you update your Chef recipe(s) (the configuration code) with what you want to change and then publish it to the Chef Master.  The machines checks for new configurations periodically and once it sees a new configuration for itself, it will start applying it.  If the change was to update the version of Apache, it will go and update the version of Apache.

### [1] The network
The cloud providers usually gives you a construct mostly known as a virtual private cloud (VPC).  You can think of this as your datacenter or a logical contruct that holds everything else in side of it.  Everything in the grey box will be inside of tihs VPC.

Creating this logical contruct you usually define items such as:
* CIDR - the IP address range for this network and how everything inside this contruct will be IP'ed
* Gateways to the external
* General routing

In the diagram, the Terraform tool creates these items.

### [2] Subnets
The next thing to define are subnets.  You won't be able to create any resources such as VM instances, load balancer, databases without a subnet because you have to put these items in a subnet.

In this diagram, the Terraform tool creates these items.

### [3] Load Balancer
The load balancer needs to be created and the Terraform tool can create that for us.  

We define the parameters such as:
* subnet we want to place it into
* Port(s) that are exposed to the internet
* Backend server(s) or server groups to forward traffic and the port(s)
* Healthchecks for the backend servers
* TLS settings and certs
* Routing algorithms
* etc

### [4] Server groups
While this arrow is only pointing to one set of server groups.  Each server groups similar to this will be created in a similar way.  Server groups is one or more servers that are identical in configuration.  You create server groups for high availability or to handle more load.  

Configurations:
* Type of servers - how many CPU, memory, networking, etc
* IPs that it gets
* Which subnet is it in
* How many are there
* Scaling characteristics
* Startup configurations - what server type is it
* etc

### [5] Bastion host
This is similar to the "server groups" but instead this is usually just one server.  You would still use Terraform to create this server because you don't want to do this manually.  This also makes recreating it easy since all of the configuration on how it was created is in code.

### [6] Databases
Databases are also created with Terraform.  Most cloud providers has some kind of a managed database option and for the most part Terraform can help you create those also.  For example it can help you create AWS RDS which is a managed database service from AWS.  At the end of the day, these managed database services are still machine and configurating them ask you the same questions as if you were configuration a machine (like the configurations above).

### [7] Server configuration
After the machine (virtual machine instance) has been created by Terraform, the machine will boot up and start doing what you configured it to do.  In this scenario, the machine will connect to it's Chef master and get the configs for itself.  In Terraform when we created this server group or machine we also gave it information on what type of server it is and access to the Chef Master.  The machine will use this information when contacting the Chef Master for it's configuration and then proceed to provision itself and bring it up to a ready state.

### [8] Bastion configuration
Same as "server configuration", this is just denoting that the bastion host is under the same controls

### [9] Datastore configuration
Depending on what the datastore is, it might or might not be under the configuration management's control.  Services like AWS RDS, will not allow you to do this.  All configuration changes will be done through Terraform.  However, if these were regular machines, then it would be under the configuration managements control.

## Transition to Kubernetes
The transition to Kubernetes has a lot of similarities to the "n-tier architecture" setup.  At the end of the day, it is trying to provide the same functionality but with more ease of use in the workflow and cheaper compute cost by colocating more items to make your usage of the cloud more dense.

Easier workflow:
* With the "n-tier architecture" and the control plane above, the machanism to update code was not so great.  To update code, you were basically dealing with infrastructure to perform the upgrade.  It was very hard to test the setup locally because you just didn't have the machines locally and if you did, you need it under the control of the Chef Master.  This made it hard to create something and then reasonably test it locally to make sure it works like how it will be deployed in production.
* This usually meant that developers had a development environment locally where they created the application.  They tested the application to the best of their abilities locally.  Then they would send the application off to the CI/CD system.  If the application called for adding a library or updating something, they had to either go to the Chef code and make the changes there or ask the infrastructure people to make that change.  Then they would have to coordinate on when those changes will be pushed out.  Can it be pushed out before the new application code gets pushed out?  Can the current application version handle that?  Does the infra changes have to happen at the same time as the application code gets pushed out?  There were just a lot of dependencies here which make updating anything a tricky process.

Cost:
* Usually each of the server(s) is running only one application.  How utilized is this server?  Servers are usually only 20% utilized (point to some sources of this 20% number here) which means 80% of what you are paying the cloud provider is going to waste.
* It also cost a company more when there has to be a high degree of coordination between teams like the above example between dev and infrastructure teams to deploy something out.  This usually mean that you had one or more person from each team to attend the deployment.  Sometime you can only do deployments during off hours.  Now you have these people at some odd hours of the night performing these tasks.  Not only will they not like it, their productivity the next day will surely suffer.

Moving to the Kubernetes infrastructure and workflow tries to solve some of these problems.  I'm not saying that Kubernetes solves all the problems but it does solve some of the big ones.  Is it the panacea of all infrastructure design, most likely not.  In my opinion, it is just another step in infrastructure technology.  Sooner or later there will be something that is better than Kubernetes.  Also, in my opinion, Kubernetes currently represents the best way to deploy, run, and maintain infrastructure and applications.

This does not necessarily mean that Kubernetes is easier.  In fact, Kubernetes makes everything more complicated.  Think about when we went from bare metal servers to virtualization.  That added a layer of complexity.  Just like that, we are now not only using virtualization and containers, there is the Kubernetes layer that is placed on top of all of that.  With more layers and each layer interacting with the layer below and above it, there is just more to know about the entire stack.  However, these layers also helps us abstract things we do not want to deal with.  For example, performing a "rolling deployment" of an application.  Prior to Kubernetes, we mostly had to create a scheme or write something up to do this.  It would bring up a new node or application with the new version, make sure it is up and running, and then put it behind the load balancer.  That was usually custom code.  With Kubernetes, that concept and workflow is already built into Kubernetes.  If you follow their opinionated way of doing it, you don't have to create anything custom and you can just use it.  There are many more examples like that but the point here is that with the added complexity you get added functionality.  The downside (there is always a downside to something you get for free), is that it makes everything more complex and difficult to understand and troubleshoot.  I think the projects like these have to strike a balance between giving useful functionality to the users and keeping it to a level where people can reasonable learn how to make this work for them and be able to troubleshoot it.  I do think that Kubernetes is leaning a little on the more difficult side though.

## How the n-teir architecture maps to Kubernetes

### Control plane
There are still two levels here:

1) Infrastructure building
* This is mostly the same.  Terraform has less duties in the Kubernetes architecture
* Terraform still creates the VPC and some some subnets but it does not create the server nodes anymore
* Kubernetes will create the subnets, nodes, load balancers, etc that it needs.  

2) Orchestration and scheduling
* Chef will not be used anymore
* Kubernetes will handle scheduling workloads onto a node
* There are still various groups of nodes in various subnets but Kubernetes will be controlling all of this
* There can now also be shared workload nodes where a generic type of workload can run on to gain more node usage efficiency


|             | Chef / Virtualization                 |  Docker / Containers / Kubernetes                                                                                                                                              |
|-------------|---------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| An Instance | This usually means a virtual machine. |  The terminology here gets a little fuzzy depending on the context.  Someone can be referring   the virtual machine or a container instance.  It is best to be specific here.S |
|             |                                       |                                                                                                                                                                                |
|             |                                       |                                                                                                                                                                                |
(Table generated with [https://www.tablesgenerator.com/markdown_tables](https://www.tablesgenerator.com/markdown_tables))

## Control plane

![kubernetes control plane](/docs/kubernetes-security/images/kubernetes-controle-plane.png)

### [1] Kubernetes API
The "Kubernetes API" (RESTful API) is composed of a few services composed in a microservice fashion.  The main point we will go over here is that this is how you interact with Kubernetes and how the kubernetes ecosystem interacts with Kubernetes.  This is the single (redundant) interface where everything talks to Kubernetes through.  If you are using the `kubectl` tool or an Kubernetes SDK.  This handles authentication, authorization, and accounting. 

That makes these items one of the most important pieces in the environment.  If these items are compromised that can likely mean that the intruder can do whatever they feel like on your cluster.  This means that these items should be protected very very well.

For example you should:
* Not put the Kubernetes API on the internet.  You should not be able to reach this API endpoint directly.  This is basically your main control point for the Kubernetes cluster.  While this endpoint does have authentication, there has been numerous vulnerabilities with this endpoint where a user can bypass authentication or authorization and who's to say that there are no more of these bugs in this API.  It is best just to not put this directly on the internet without anyother layer of protection.  At a minimum there should be an IP whitelist to only allow certain IPs to be able to reach this endpoint.  The best thing to do is to set the Kubernetes API only with a private IP address and only accessible from your internal network.  Then the only way to reach it is to get into your internal network first either via a VPN or some other method and then the user can reach this endpoint.
* All of the machines in this group should only have private IPs and not on the internet
* Firewall rules to only allow source IPs to it that needs access even on your internal network
* Use TLS for every communication link


### [2] etcd
This is the datastore where all persistent information is stored that the Kubernetes API uses.  This is usually a highly available redundant system.

Security considerations:
* This datastore is generally note shared with any other applications.  This datastore is very important and stores all state information about the cluster.  This should be a one application datastore.
* Limited network access should be configured to this datastore.  This datastore can either run on their own hosts or as containers inside of the Kubernetes clusters. No matter where it is running, only the Kubernetes masters should have network access to this datastore.
* Encryption on the transport layer.  TLS should be used on the transport layer
* Authentication should be used
* A regular backup of this datastore should be performed

### [3] kubelet
This is the process that runs on worker nodes which reports back to the Kubernetes API.  This process gives information and takes orders from the Kubernetes API on what to schedule onto the nodes.

Security considerations:
* Encryption on the transport layer.  TLS should be used.
* Anonymous access is turned off

### [4] The cloud
The cloud is your cloud.  This could be AWS, GCP, Azure, Digital Ocean, etc.  This is the platform that you are renting compute and network from.  Securing this down is a whole other big topic by itself.  Each cloud has authentication, authorization, and accounting (AAA) and each cloud does it a little differently.  You should take these items for your cloud seriously because if this is compromised and depending on the access level of which credentials are compromised that can give full access to your cloud.  That basically means no matter what other security measures you have set or how many security layers you have, it can most likely be bypassed with an admin level type credential.

Security considerations:
* Don't allow the entire internet access your cloud account's API.  
* Use assume type roles and use single sign on (SSO)

### [5] Namespaces
Namespaces are logical boundries on the cluster.

![Kubernetes Namepsace](/docs/kubernetes-security/images/kubernetes-namespaces.png)

A namespace can cross the host boundry.  This means that a pod running on two different hosts can be in the same namespace and multiple namespaces can be on a single host depending what pods are running on it.  This might add complexity but it allows the pod owner not have to worry about lower level implementation on where the pod will run.  The pod owner just knows that it asked for a certain number of pods to run and Kubernetes will go and make that happen on the cluster (if it can).  

This does mean that the isolation mechanisms are a little bit more complex.  You can set network boundries between namespaces.  You can tell Kubernetes to only allow the pods in `namespace 1` to be able to open network connections to other pods on the same namespace and not to let any other namespace to be able to reach it.  You have fine grain controls over the network and you can even tell Kubernetes that another namespace can contact a certain pod's sevice on a certain port only.  With these semantics all on the Kubernetes level, these concepts and configurations of it is cloud agnostic and is portable between clouds with no change.

### Pod
A pod is one of the smallest units in Kubernetes.  This is a logical contruct that holds containers and the configurations around it.  You can think of this as a "server" and developers should think of it mostly this way.  They have mostly full control of this unit of work.  For example, each Pod will get an unique IP.  This means that the ports that this pod exposes will never collide with anything else, just like a server.  Internally to the server, there can not be overlapping ports because they will all try to bind to the same IP which will cause a collision.  

Some items that can be defined:
* Container image
* Run commands and args
* Docker security settings
* CPU/Memory requests and limits
* Config files, disks, secrets to mount into it
* Environment variables to add
* Labels for naming and other querying uses


## n-tier application in Kubernetes
This is a re-implementation of the n-tier application but in Kubernetes.

![the stack](/docs/kubernetes-security/images/n-tier-in-kubernetes.png)

### [1] Internet
The Internet is the Internet in general.  Items placed here can be reach by any node on the Internet.

### [2] Internal Network
The internal network is a network that has IPs in the [RFC 1918](https://tools.ietf.org/html/rfc1918) ranges.  These IPs are not routable via the internet.  This portion is the same as before.

While this diagram does not show the availability zones, it can be in one or more availability zones like before.  When using Kubernetes the mindset on deploying applications changes from looking at it from a network perspective and "physicallY" definining that a web or a db server should be in two different availablity zones.  It shifts over to looking at deploying an application to the application itself.  You are telling Kubernetes what you want the end state to be and Kubernetes tries to make that happen for you.  This means that yes...your Kubernetes cluster has to have workers (kubelets) in different availablity zones or else no workload can ever get there.  However, from an applications owner's point of view, they are expecting that already and they are telling the deployment of their application that it should be in different availability zones.

The various different types of subnets are also removed.  There might be various subnets for various Kubernetes workers but they might not map one to one on the application workload.  With the use of Kubernetes namespaces we can simulate these network boundries.  If we want to tightly control the network between two different types of applications we can do this on the Kubernetes level and not have to re-arrange the network level.

In our previous setup, we had dedicated hosts for each application.  This usually tends to lead to under use of the servers.  In Kubernetes we tell Kubernetes how much CPU/Memory/Disk that each pod wants and Kubernetes handles turning on worker nodes for us.

Security Considerations:
* You are now potentially dealing with diverse workloads all running on the same host.  Before you had a host that was dedicated to a certain application.  With Kubernetes, you generally don't tell it where to run your application (but you can).  If you need strict host isolation you will need to set the Kubernetes taints so that only certain workloads can run on those nodes. 
* Detecting network traffic per application is more difficult now due to the fact that multiple workload(s) can run on the same node and these nodes are not dedicated to one thing where you can monitor that for.  Your monitoring will have to be dynamic now and know what is running on the node.  This mainly means your security application(s) needs to be Kubernetes aware and go to the Kubernetes API for information on what is running on each node.

### [3] Load Balancers
Just like before, we still have a load balancer here that spans the external network (internet) and our internal network.  This is still the main way we get traffic inbound from the internet into our application.  With the Kubernetes setup, there is some flexibility here on how this is setup.  You can have you cloud load blancer off load the main TLS (just like before) or make it a layer 4 load balancer and it will simiply forward traffic inbound to a Kubernetes ingress (in this example, it is using the nginx-ingress but there are many other ingress controllers you can use).  This Kubernetes ingress controller can offload the TLS.  Kubernetes also makes it easy to get free certificates from Let's Encrypt or to use your own certificates.  This is all driven by Kubernetes configurations and most of these items are portable across clouds.

Security considerations:
* Generally the same considerations here as before

### [4] API Service
This is a representation of a web application being exposed outwards to the external internet.  Traffic from the internet will hit these pods.  It could be one or more pods (in this example there are 3).  

Security considerations:
* This is still the first point where external traffic hits the application.  Anything that is sent here should be considered untrusted just like in the previous architecture.  Strict checking of authentication and payload should be performed on all inputs and outputs.
* Kubernetes RBAC should be used for access controls on who can administer and make changes to this application

### [5] Datastores
The datastore is the same as before but now it is under Kubernetes control.  These can be on shared or dedicated hosts.  Usually databases would be on a dedicated host.

Security considerations:
* Kubernetes RBAC should be used for access controls on who can administer and make changes to this application
* You should treat this like any other application running on Kubernetes and apply the standard set of security controls for this application as well.



## Deployment workflow

![the stack](/docs/kubernetes-security/images/deployment-workflow.png)