# Kubernetes Fundamentals LFS258

### Lab 1.1. Configuring the System for sudo

It is very dangerous to run a **root shell** unless absolutely necessary: a single typo or other mistake can cause serious (evenfatal) damage.

Thus, the sensible procedure is to configure things such that single commands may be run with superuser privilege, by using the **sudo** mechanism. With **sudo** the user only needs to know their own password and never needs to know the root password.

If you are using a distribution such as **Ubuntu**, you may not need to do this lab to get **sudo** configured properly for the course. However, you should still make sure you understand the procedure.

To check if your system is already configured to let the user account you are using run **sudo**, just do a simple command like:

``` sh
sudo ls
```

You should be prompted for your user password and then the command should execute. If instead, you get an error messageyou need to execute the following procedure.

Launch a root shell by typing **su** and then giving the **root** password, not your user password.

On all recent **Linux** distributions you should navigate to the */etc/sudoers.d* subdirectory and create a file, usually with the name of the user to whom root wishes to grant **sudo** access. However, this convention is not actually necessary as **sudo** will scan all files in this directory as needed. The file can simply contain:

``` sh
student ALL=(ALL)  ALL
```

if the user is *student*.

An older practice (which certainly still works) is to add such a line at the end of the file */etc/sudoers*. It is best to do so using the **visudo** program, which is careful about making sure you use the right syntax in your edit.

You probably also need to set proper permissions on the file by typing:

``` sh
sudo chmod 440 /etc/sudoers.d/student
```

(Note some **Linux** distributions may require *400* instead of *440* for the permissions.)

After you have done these steps, exit the root shell by typing *exit* and then try to do *sudo ls* again.

There are many other ways an administrator can configure **sudo**, including specifying only certain permissions for certain users, limiting searched paths etc. The */etc/sudoers* file is very well self-documented.

However, there is one more setting we highly recommend you do, even if your system already has **sudo** configured. Most distributions establish a different path for finding executables for normal users as compared to root users. In particular the directories */sbin* and */usr/sbin* are not searched, since **sudo** inherits the *PATH* of the user, not the full root user.

Thus, in this course we would have to be constantly reminding you of the full path to many system administration utilities; any enhancement to security is probably not worth the extra typing and figuring out which directories these programs are in. Consequently, we suggest you add the following line to the *.bashrc* file in your home directory:

``` sh
PATH=$PATH:/usr/sbin:/sbin
```

If you log out and then log in again (you don’t have to reboot) this will be fully effective.

## Basics of Kubernetes

According to the kubernetes.io website, Kubernetes is: "an open-source system for automating deployment, scaling, and management of containerized applications".

In Greek, κυβερνητης means the Helmsman, or pilot of the ship. Keeping with the maritime theme of Docker containers, Kubernetes is the pilot of a ship of containers. Due to the difficulty in pronouncing the name, many will use a nickname, K8s, as Kubernetes has eight letters between K and S. The nickname is pronounced like Kate's.

### Kubernetes Architecture

![K8s architecture](../../images/k8s-arch.png "K8s architecture")

In its simplest form, Kubernetes is made of a central manager (aka master) and some worker nodes, once called minions (we will see in a follow-on chapter how you can actually run everything on a single node for testing purposes). The manager runs an API server, a scheduler, various controllers and a storage system to keep the state of the cluster, container settings, and the networking configuration.

Kubernetes exposes an API via the API server. You can communicate with the API using a local client called kubectl or you can write your own client and use curl commands. The kube-scheduler is forwarded the requests for running containers coming to the API and finds a suitable node to run that container. Each node in the cluster runs two processes: a kubelet and kube-proxy. The kubelet receives requests to run the containers, manages any necessary resources and watches over them on the local node. The kubelet interacts with the local container engine, which is Docker by default, but could be rkt or cri-o, which is growing in popularity. 

The kube-proxy creates and manages networking rules to expose the container on the network.

Using an API-based communication scheme allows for non-Linux worker nodes and containers. Support for Windows Server 2019 was graduated to Stable with the 1.14 release. Only Linux nodes can be master on a cluster.

### Terminology

Kubernetes is an orchestration system to deploy and manage containers. Containers are not managed individually; instead, they are part of a larger object called a Pod. A Pod consists of one or more containers which share an IP address, access to storage and namespace. Typically, one container in a Pod runs an application, while other containers support the primary application.

Kubernetes uses namespaces to keep objects distinct from each other, for resource control and multi-tenant considerations. Some objects are cluster-scoped, others are scoped to one namespace at a time. As the namespace is a segregation of resources, pods would need to leverage services to communicate.

Orchestration is managed through a series of watch-loops, also called controllers or operators. Each controller interrogates the kube-apiserver for a particular object state, then modifying the object until the declared state matches the current state. These controllers are compiled into the kube-controller-manager, but others can be added using custom resource definitions. The default and feature-filled operator for containers is a Deployment. A Deployment does not directly work with pods. Instead it manages ReplicaSets. The ReplicaSet is an operator which will create or terminate pods by sending out a podSpec. The podSpec is sent to the kubelet, which then interacts with the container engine, Docker by default, to spawn or terminate a container until the requested number is running.

The service operator requests existing IP addresses and endpoints and will manage the network connectivity based on labels. These are used to communicate between pods, namespaces, and outside the cluster. There are also Jobs and CronJobs to handle single or recurring tasks, among others.

To easily manage thousands of Pods across hundreds of nodes can be a difficult task. To make management easier, we can use labels, arbitrary strings which become part of the object metadata. These can then be used when checking or changing the state of objects without having to know individual names or UIDs. Nodes can have taints to discourage Pod assignments, unless the Pod has a toleration in its metadata.

There is also space in metadata for annotations which remain with the object but cannot be used by Kubernetes commands. This information could be used by third-party agents or other tools.

## Installation and Configuration

### Installation tools

**kubectl** runs locally on your machine and targets the API server endpoint. It allows you to create, manage, and delete all Kubernetes resources (e.g. Pods, Deployments, Services). This command line will use *$HOME/.kube/config* as a configuration file. This contains all the Kubernetes endpoints that you might use. If you examine it, you will see cluster definitions (i.e. IP endpoints), credentials, and contexts. A context is a combination of a cluster and user credentials. You can pass these parameters on the command line, or switch the shell between contexts with a command, as in: 

``` sh
kubectl config use-context foobar
```

**kubeadm**, the community-suggested tool from the Kubernetes project, that makes installing Kubernetes easy and avoids vendor-specific installers. Getting a cluster running involves two commands: kubeadm init, that you run on one Master node, and then, kubeadm join, that you run on your worker or redundant master nodes, and your cluster bootstraps itself. The flexibility of these tools allows Kubernetes to be deployed in a number of places.

### Installation methods

#### GKE

``` sh
gcloud container clusters create linuxfoundation
gcloud container clusters list
kubectl get nodes
gcloud container clusters delete linuxfoundation
```

#### Minikube

``` sh
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
chmod +x minikube
sudo mv minikube /usr/local/bin
minikube start
kubectl get nodes
minikube pause
minikube stop
minikube delete --all
```

#### kubeadm

* Run kubeadm init on the head node.
* Create a network for IP-per-Pod criteria.
* Run kubeadm join --token token head-node-IP on worker nodes.

You can also create the network with kubectl by using a resource manifest of the network.

For example, to use the Weave network, you would do the following: 

``` sh
kubectl create -f https://git.io/weave-kube 
```

Once all the steps are completed, workers and other master nodes joined, you will have a functional multi-node Kubernetes cluster, and you will be able to use kubectl to interact with it.

If you build your cluster with kubeadm, you also have the option to upgrade the cluster using the kubeadm upgrade command. While most choose to remain with a version for as long as possible, and will often skip several releases, this does offer a useful path to regular upgrades for security reasons.

* *plan*: This will check the installed version against the newest found in the repository, and verify if the cluster can be upgraded.
* *apply*: Upgrades the first control plane node of the cluster to the specified version.
* *diff*: Similar to an apply --dry-run, this command will show the differences applied during an upgrade.
* *node*: This allows for updating the local kubelet configuration on worker nodes, or the control planes of other master nodes if there is more than one. Also, it will access a phase command to step through the upgrade process.

General upgrade process:

* Update the software
* Check the software version
* Drain the control plane
* View the planned upgrade
* Apply the upgrade
* Uncordon the control plane to allow pods to be scheduled.

#### Installing a Pod Network

Prior to initializing the Kubernetes cluster, the network must be considered and IP conflicts avoided. There are several Pod networking choices, in varying levels of development and feature set. Many of the projects will mention the Container Network Interface (CNI)

* **Calico**: A flat Layer 3 network which communicates without IP encapsulation, used in production with software such as Kubernetes, OpenShift, Docker, Mesos and OpenStack. Viewed as a simple and flexible networking model, it scales well for large environments.
* **Flannel**: A Layer 3 IPv4 network between the nodes of a cluster. Developed by CoreOS, it has a long history with Kubernetes. Focused on traffic between hosts, not how containers configure local networking, it can use one of several backend mechanisms, such as VXLAN. A flanneld agent on each node allocates subnet leases for the host. While it can be configured after deployment, it is much easier prior to any Pods being added.
* **Kube-Router**: Feature-filled single binary which claims to "do it all". The project is in the alpha stage, but promises to offer a distributed load balancer, firewall, and router purposely built for Kubernetes.
* Romana: This is another project aimed at network and security automation for cloud native applications. Aimed at large clusters, IPAM-aware topology and integration with kops clusters.
* **Weave Net**: It is typically used as an add-on for a CNI-enabled Kubernetes cluster.

#### More Installation Tools

Since Kubernetes is, after all, like any other applications that you install on a server (whether physical or virtual), all of the configuration management systems (e.g., Chef, Puppet, Ansible, Terraform) can be used. Various recipes are available on the Internet.

* **kubespray**: kubespray is now in the Kubernetes incubator. It is an advanced Ansible playbook which allows you to set up a Kubernetes cluster on various operating systems and use different network providers. ​It was once known as kargo.
* **kops**: kops (Kubernetes Operations) lets you create a Kubernetes cluster on AWS via a single command line. Also in beta for GKE and alpha for VMware.
* **kube-aws**: kube-aws is a command line tool that makes use of the AWS Cloud Formation to provision a Kubernetes cluster on AWS.
* **kubicorn**: kubicorn is a tool which leverages the use of kubeadm to build a cluster. It claims to have no dependency on DNS, runs on several operating systems, and uses snapshots to capture a cluster and move it.

#### Main Deployment Configurations

* **Single-node**: With a single-node deployment, all the components run on the same server. This is great for testing, learning, and developing around Kubernetes.
* **Single head node, multiple workers**: Adding more workers, a single head node and multiple workers typically will consist of a single node etcd instance running on the head node with the API, the scheduler, and the controller-manager.
* **Multiple head nodes with HA, multiple workers**: Multiple head nodes in an HA configuration and multiple workers add more durability to the cluster. The API server will be fronted by a load balancer, the scheduler and the controller-manager will elect a leader (which is configured via flags). The etcd setup can still be single node.
* **HA etcd, HA head nodes, multiple workers**: The most advanced and resilient setup would be an HA etcd cluster, with HA head nodes and multiple workers. Also, etcd would run as a true cluster, which would provide HA and would run on nodes separate from the Kubernetes head nodes.

#### systemd Unit File for Kubernetes

In any of these configurations, you will run some of the components as a standard system daemon. As an example, you can see here a sample systemd unit file to run the controller-manager. Using kubeadm will create a system daemon for kubelet, while the rest will deploy as containers:

``` yaml

* name: kube-controller-manager.service

    command: start 
    content: |
      [Unit]
      Description=Kubernetes Controller Manager Documentation=https://github.com/kubernetes/kubernetes
      Requires=kube-apiserver.service
      After=kube-apiserver.service
      [Service]
      ExecStartPre=/usr/bin/curl -L -o /opt/bin/kube-controller-manager -z /opt/bin/kube-controller-manager https://storage.googleapis.com/kubernetes-release/release/v1.7.6/bin/linux/amd64/kube-controller-manager
      ExecStartPre=/usr/bin/chmod +x /opt/bin/kube-controller-manager
      ExecStart=/opt/bin/kube-controller-manager \
        --service-account-private-key-file=/opt/bin/kube-serviceaccount.key \
        --root-ca-file=/var/run/kubernetes/apiserver.crt \
        --master=127.0.0.1:8080 \
...
```

This is by no means a perfect unit file. It downloads the controller binary from the published release of Kubernetes and sets a few flags to run.

#### Using Hyperkube

While you can run all the components as regular system daemons in unit files, you can also run the API server, the scheduler, and the controller-manager as containers. This is what kubeadm does.

Similar to minikube, there is a handy all-in-one binary named hyperkube, which is available as a container image (e.g. gcr.io/google_containers/hyperkube:v1.16.7). This is hosted by Google, so you may need to add a new repository so Docker would know where to pull the file.

This method of installation consists in running a kubelet as a system daemon and configuring it to read in manifests that specify how to run the other components (i.e. the API server, the scheduler, etcd, the controller). In these manifests, the hyperkube image is used. The kubelet will watch over them and make sure they get restarted if they die.

``` sh
docker run --rm gcr.io/google_containers/hyperkube:v1.16.7 /hyperkube kube-apiserver --help
docker run --rm gcr.io/google_containers/hyperkube:v1.16.7 /hyperkube kube-scheduler --help
docker run --rm gcr.io/google_containers/hyperkube:v1.16.7 /hyperkube kube-controller-manager --help
```

#### Compiling from Source

Kubernetes can also be compiled from source relatively quickly. You can clone the repository from GitHub, and then use the Makefile to build the binaries. You can build them natively on your platform if you have a Golang environment properly set up, or via Docker containers if you are on a Docker host.

To build natively with Golang, first install Golang. Download files and directions can be found online.

Once Golang is working, you can clone the kubernetes repository, around 500MB in size. Change into the directory and use make:

``` sh
cd $GOPATH
git clone https://github.com/kubernetes/kubernetes
cd kubernetes
make
```

On a Docker host, clone the repository anywhere you want and use the make quick-release command. The build will be done in Docker containers. 

The *_output/bin* directory will contain the newly built binaries.

### Lab 3.1. Install Kubernetes

To download yaml well indented files:

``` sh
wget https://training.linuxfoundation.org/cm/LFS258/LFS258V2021-02-05SOLUTIONS.tar.xz --user=LFtraining --password=Penguin2014
tar -xvf LFS258V2021-02-05SOLUTIONS.tar.xz
```

From **master** node:

``` sh
sudo -i
apt-get update && apt-get upgrade -y
apt-get install -y vim
apt-get install -y docker.io
vim /etc/apt/sources.list.d/kubernetes.list #deb  http://apt.kubernetes.io/  kubernetes-xenial  main
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
apt-get update
apt-get install -y kubeadm=1.19.1-00 kubelet=1.19.1-00 kubectl=1.19.1-00
apt-mark hold kubelet kubeadm kubectl
wget https://docs.projectcalico.org/manifests/calico.yaml
ip addr show # 10.2.0.2
vim /etc/hosts # Add line -> 10.2.0.2 k8smaster
vim kubeadm-config.yaml
```

``` yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: 1.19.1              #<-- Use the word stable for newest version
controlPlaneEndpoint: "k8smaster:6443" #<-- Use the node alias not the IP
networking:
  podSubnet: 192.168.0.0/16            #<-- Match the IP range from the Calico config file
```

``` sh
kubeadm init --config=kubeadm-config.yaml --upload-certs | tee kubeadm-init.out
exit
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
less .kube/config
sudo cp /root/calico.yaml .
kubectl apply -f calico.yaml
sudo apt-get install bash-completion -y # Install bash-completion, if not installed -> exit and log back in
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> $HOME/.bashrc
kubectl describe nodes master
kubectl -n kube-system get pod
sudo kubeadm config print init-defaults # View other values to include in kubeadm-config.yaml
```

### Lab 3.2. Grow the Cluster

From **worker** node:

``` sh
sudo -i
apt-get update && apt-get upgrade -y
apt-get install -y docker.io
apt-get install -y vim
vim /etc/apt/sources.list.d/kubernetes.list #deb  http://apt.kubernetes.io/  kubernetes-xenial  main
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
apt-get update
apt-get install -y kubeadm=1.19.1-00 kubelet=1.19.1-00 kubectl=1.19.1-00
apt-mark hold kubeadm kubelet kubectl
```

From **master** node:

``` sh
ip addr show ens4 | grep inet
sudo kubeadm token list
sudo kubeadm token create # Create a new one if its expired (duration of 2h)
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/ˆ.* //'
```

From **worker** node:

``` sh
vim /etc/hosts # Add line -> 10.2.0.2 k8smaster
kubeadm join --token 7e2wt4.uwp2jwbukw2ign6p k8smaster:6443 --discovery-token-ca-cert-hash sha256:b34138b00fafa8ebe124c3bde3c464642962b9d5de6e707995009b4fc269f8ba
kubectl get nodes # It fails due .kube/config is not configured
exit
kubectl get nodes # It fails due .kube/config is not configured
ls -l .kube # ls: cannot access'.kube': No such file or directory
```

### Lab 3.3. Finish Cluster Setup

From **master** node:

``` sh
kubectl get node
kubectl describe node master
kubectl describe node | grep -i taint
kubectl taint nodes --all node-role.kubernetes.io/master- # The master node begins tainted for security and performance reasons. We will allow usage of the node in the training environment, but this step may be skipped in a production environment
kubectl describe node | grep -i taint
kubectl taint nodes --all node.kubernetes.io/not-ready- # If present, not present for me
kubectl get pods --all-namespaces
kubectl -n kube-system delete pod coredns-576cbf47c7-vq5dz coredns-576cbf47c7-rn6v4 # Only if coredns- pods are stuck in ContainerCreating status
ip a # tunl0 -> tunnel interface and cali interfaces
```

### Lab 3.4. Deploy a Simple Application

From **master** node:

``` sh
kubectl create deployment nginx --image=nginx
kubectl get deployments
kubectl describe deployment nginx
kubectl get events
kubectl get deployment nginx -o yaml
kubectl get deployment nginx -o yaml > first.yaml
vim first.yaml # Remove creationTimestamp, resourceVersion, selfLink and uid lines, also status: block --> :+,$d to delete from status: with vim
kubectl delete deployment nginx
kubectl create -f first.yaml
kubectl get deployment nginx -o yaml > second.yaml
diff first.yaml second.yaml
kubectl create deployment two --image=nginx --dry-run=client -o yaml
kubectl get deployment
kubectl get deployments nginx -o yaml
kubectl get deployment nginx -o json
kubectl expose -h
kubectl expose deployment/nginx # Error due not port included
vim first.yaml # Add *ports: - containerPort: 80 protocol: TCP* on spec.template.spec.containers.nginx
kubectl replace -f first.yaml
kubectl get deploy,pod
kubectl expose deployment/nginx
kubectl get svc nginx
kubectl get ep nginx
kubectl get pods
kubectl describe pod nginx-7848d4b86f-z2dn5 | grep Node: # Node:         worker/10.2.0.3
```

From **worker** node:

``` sh
sudo tcpdump -i tunl0 # If not installed: apt-get install tcpdump
```

From **master** node:

``` sh
curl 10.107.223.107:80
curl 192.168.171.67:80 # The same output from the svc and the endpoint
kubectl get deployment nginx
kubectl scale deployment nginx --replicas=3
kubectl get deployment nginx
kubectl get ep nginx # 3 different endpoints
kubectl get pod -o wide
kubectl delete pod nginx-7848d4b86f-z2dn5 # Removing the old one
kubectl get po
kubectl get ep nginx
```

### Lab 3.5. Access from Outside the Cluster

You can access a Service from outside the cluster using a DNS add-on or environment variables. We will use environment variables to gain access to a Pod.

From **master** node:

``` sh
kubectl get po
kubectl exec nginx-7848d4b86f-gbwfk -- printenv | grep KUBERNETES
kubectl get svc
kubectl delete svc nginx
kubectl expose deployment nginx --type=LoadBalancer
kubectl get svc
```

Open a browser on your local system and use the public IP of your node and port shown in the output above.

``` sh
kubectl scale deployment nginx --replicas=0
kubectl get po
```

Test the web page again. Once all pods have finished terminating accessingthe web page should fail.

``` sh
kubectl scale deployment nginx --replicas=2
kubectl delete deployments nginx
kubectl delete ep nginx
kubectl delete svc nginx
```

## Kubernetes Architecture

### Main Components

* Master and worker nodes
* Controllers
* Services
* Pods of containers
* Namespaces and quotas
* Network and policies
* Storage

A Kubernetes cluster is made of a master node and a set of worker nodes. The cluster is all driven via API calls to controllers, both interior as well as exterior traffic. We will take a closer look at these components next.

![K8s architecture](../../images/k8s-arch2.png "K8s architecture")

### Master node

The Kubernetes master runs various server and manager processes for the cluster. Among the components of the master node are the kube-apiserver, the kube-scheduler, and the etcd database. As the software has matured, new components have been created to handle dedicated needs, such as the cloud-controller-manager; it handles tasks once handled by the kube-controller-manager to interact with other tools, such as Rancher or DigitalOcean for third-party cluster management and reporting.

As a concept, the various pods responsible for ensuring the current state of the cluster matches the desired state are called the control plane.

When building a cluster using kubeadm, the kubelet process is managed by systemd. Once running, it will start every pod found in */etc/kubernetes/manifests/*.

#### kube-apiserver

The kube-apiserver is central to the operation of the Kubernetes cluster. All calls, both internal and external traffic, are handled via this agent. All actions are accepted and validated by this agent, and it is the only connection to the etcd database. It validates and configures data for API objects, and services REST operations. As a result, it acts as a master process for the entire cluster, and acts as a frontend of the cluster's shared state.

Starting as an alpha feature in v1.16 is the ability to separate user-initiated traffic from server-initiated traffic. Until these features are developed, most network plugins commingle the traffic, which has performance, capacity, and security ramifications.

#### kube-scheduler

The kube-scheduler uses an algorithm to determine which node will host a Pod of containers. The scheduler will try to view available resources (such as volumes) to bind, and then try and retry to deploy the Pod based on availability and success. There are several ways you can affect the algorithm, or a custom scheduler could be used instead. You can also bind a Pod to a particular node, though the Pod may remain in a pending state due to other settings. One of the first settings referenced is if the Pod can be deployed within the current quota restrictions. If so, then the taints and tolerations, and labels of the Pods are used along with those of the nodes to determine the proper placement.

#### etcd database

The state of the cluster, networking, and other persistent information is kept in an etcd database, or, more accurately, a b+tree key-value store. Rather than finding and changing an entry, values are always appended to the end. Previous copies of the data are then marked for future removal by a compaction process. It works with curl and other HTTP libraries, and provides reliable watch queries.

Simultaneous requests to update a value all travel via the kube-apiserver, which then passes along the request to etcd in a series. The first request would update the database. The second request would no longer have the same version number, in which case the kube-apiserver would reply with an error 409 to the requester. There is no logic past that response on the server side, meaning the client needs to expect this and act upon the denial to update. 

There is a master database along with possible followers. They communicate with each other on an ongoing basis to determine which will be master, and determine another in the event of failure. While very fast and potentially durable, there have been some hiccups with new tools, such as kubeadm, and features like whole cluster upgrades.

While most Kubernetes objects are designed to be decoupled, a transient microservice which can be terminated without much concern etcd is the exception. As it is, the persistent state of the entire cluster must be protected and secured. Before upgrades or maintenance, you should plan on backing up etcd. The **etcdctl** command allows for snapshot save and snapshot restore.

#### kube-controller-manager

The kube-controller-manager is a core control loop daemon which interacts with the kube-apiserver to determine the state of the cluster. If the state does not match, the manager will contact the necessary controller to match the desired state. There are several controllers in use, such as endpoints, namespace, and replication. The full list has expanded as Kubernetes has matured.

#### cloud-controller-manager

Remaining in beta in v1.19, the cloud-controller-manager (ccm) interacts with agents outside of the cloud. It handles tasks once handled by kube-controller-manager. This allows faster changes without altering the core Kubernetes control process. Each kubelet must use the --cloud-provider-external settings passed to the binary. You can also develop your own ccm, which can be deployed as a daemonset as an in-tree deployment or as a free-standing out-of-tree installation. The cloud-controller-manager is an optional agent which takes a few steps to enable.

#### kube-dns

To handle DNS queries, Kubernetes service discovery, and other functions, the CoreDNS server has replaced kube-dns. Using chains of plugins, one of many provided or custom written, the server is easily extensible.

### Worker nodes

All nodes run the kubelet and kube-proxy, as well as the container engine, such as Docker or cri-o. Other management daemons are deployed to watch these agents or provide services not yet included with Kubernetes.

The kubelet interacts with the underlying Docker Engine also installed on all the nodes, and makes sure that the containers that need to run are actually running. The kube-proxy is in charge of managing the network connectivity to the containers. It does so through the use of iptables entries. It also has the userspace mode, in which it monitors Services and Endpoints using a random port to proxy traffic and an alpha feature of ipvs.

You can also run an alternative to the Docker engine: cri-o or some lesser used engines. To learn how you can do that, you should check the online documentation. In future releases, it is highly likely that Kubernetes will support additional container runtime engines.

Supervisord is a lightweight process monitor used in traditional Linux environments to monitor and notify about other processes. In non-systemd cluster, this daemon can be used to monitor both the kubelet and docker processes. It will try to restart them if they fail, and log events. While not part of a typical installation, some may add this monitor for added reporting.

Kubernetes does not have cluster-wide logging yet. Instead, another CNCF project is used, called Fluentd. When implemented, it provides a unified logging layer for the cluster, which filters, buffers, and routes messages.

Cluster-wide metrics is another area with limited functionality. The metrics-server SIG provides basic node and pod CPU and memory utilization. For more metrics, many use the Prometheus project.

#### kubelet

The kubelet agent is the heavy lifter for changes and configuration on worker nodes. It accepts the API calls for Pod specifications (a PodSpec is a JSON or YAML file that describes a pod). It will work to configure the local node until the specification has been met.

Should a Pod require access to storage, Secrets or ConfigMaps, the kubelet will ensure access or creation. It also sends back status to the kube-apiserver for eventual persistence. ​

* Uses **PodSpec**
* Mounts volumes to Pod
* Download secrets
* Passes request to local container engine
* Reports status of Pods and node to cluster

Kubelet calls other components such as the Topology Manager, which uses *hints* from other components to configure topology-aware resource assignments such as for CPU and hardware accelerators. As an alpha feature, it is not enabled by default.

### Services

With every object and agent decoupled we need a flexible and scalable agent which connects resources together and will reconnect, should something die and a replacement is spawned. Each Service is a microservice handling a particular bit of traffic, such as a single NodePort or a LoadBalancer to distribute inbound requests among many Pods.

A Service also handles access policies for inbound requests, useful for resource control, as well as for security. 

* Connect Pods together
* Expose Pods to Internet
* Decouple settings
* Define Pod access policy

![Service Network](../../images/service-network.png "Service Network")

This graphic shows a pod with a primary container, App, with an optional sidecar Logger. Also seen is the pause container, which is used by the cluster to reserve the IP address in the namespace prior to starting the other pods. This container is not seen from within Kubernetes, but can be seen using docker and crictl.

This graphic also shows a ClusterIP which is used to connect inside the cluster, not the IP of the cluster. As the graphic shows, this can be used to connect to a NodePort for outside the cluster, an IngressController or proxy, or another ”backend” pod or pods.

### Controllers

An important concept for orchestration is the use of controllers. Various controllers ship with Kubernetes, and you can create your own, as well. A simplified view of a controller is an agent, or Informer, and a downstream store. Using a DeltaFIFO queue, the source and downstream are compared. A loop process receives an obj or object, which is an array of deltas from the FIFO queue. As long as the delta is not of the type Deleted, the logic of the controller is used to create or modify some object until it matches the specification. 

The Informer which uses the API server as a source requests the state of an object via an API call. The data is cached to minimize API server transactions. A similar agent is the SharedInformer; objects are often used by multiple other objects. It creates a shared cache of the state for multiple requests. 

A Workqueue uses a key to hand out tasks to various workers. The standard Go work queues of rate limiting, delayed, and time queue are typically used. 

The endpoints, namespace, and serviceaccounts controllers each manage the eponymous resources for Pods.

### Pods

The whole point of Kubernetes is to orchestrate the lifecycle of a container. We do not interact with particular containers. Instead, the smallest unit we can work with is a Pod. Some would say a pod of whales or peas-in-a-pod. Due to shared resources, the design of a Pod typically follows a one-process-per-container architecture.

Containers in a Pod are started in parallel. As a result, there is no way to determine which container becomes available first inside a pod. The use of InitContainers can order startup, to some extent. To support a single process running in a container, you may need logging, a proxy, or special adapter. These tasks are often handled by other containers in the same pod.

There is only one IP address per Pod, for almost every network plugin. If there is more than one container in a pod, they must share the IP. To communicate with each other, they can either use IPC, the loopback interface, or a shared filesystem.

While Pods are often deployed with one application container in each, a common reason to have multiple containers in a Pod is for logging. You may find the term sidecar for a container dedicated to performing a helper task, like handling logs and responding to requests, as the primary application container may not have this ability. The term sidecar, like ambassador and adapter, does not have a special setting, but refers to the concept of what secondary pods are included to do.

### Containers

While Kubernetes orchestration does not allow direct manipulation on a container level, we can manage the resources containers are allowed to consume. 

In the resources section of the **PodSpec** you can pass parameters which will be passed to the container runtime on the scheduled node: 

``` yaml
resources:
  limits: 
    cpu: "1"
    memory: "4Gi" 
  requests:
    cpu: "0.5"
    memory: "500Mi"
```

Another way to manage resource usage of the containers is by creating a **ResourceQuota** object, which allows hard and soft limits to be set in a namespace. The quotas allow management of more resources than just CPU and memory and allows limiting several objects. 

A beta feature in v1.12 uses the **scopeSelector** field in the quota spec to run a pod at a specific priority if it has the appropriate priorityClassName in its pod spec.

### Init Containers

Not all containers are the same. Standard containers are sent to the container engine at the same time, and may start in any order. LivenessProbes, ReadinessProbes, and StatefulSets can be used to determine the order, but can add complexity. Another option can be an Init container, which must complete before app containers will be started. Should the init container fail, it will be restarted until completion, without the app container running. 

The init container can have a different view of the storage and security settings, which allows utilities and commands to be used, which the application would not be allowed to use.. Init containers can contain code or utilities that are not in an app. It also has an independent security from app containers.

The code below will run the init container until the ls command succeeds; then the database container will start.

``` yaml
spec:
  containers:

  + name: main-app

    image: databaseD 
  initContainers:

  + name: wait-database

    image: busybox
    command: ['sh', '-c', 'until ls /db/dir ; do sleep 5; done; '] 
```

### Component review

Now that we have seen some of the components, let's take another look with some of the connections shown. Not all connections are shown in the diagram below. Note that all of the components are communicating with kube-apiserver. Only kube-apiserver communicates with the etcd database.

![K8s Architectural Review](../../images/k8s-arch-review.png "K8s Architectural Review")

We also see some commands, which we may need to install separately to work with various components. There is an etcdctl command to interrogate the database and calicoctl to view more of how the network is configured. We can see Felix, which is the primary Calico agent on each machine. This agent, or daemon, is responsible for interface monitoring and management, route programming, ACL configuration and state reporting.

BIRD is a dynamic IP routing daemon used by Felix to read routing state and distribute that information to other nodes in the cluster. This allows a client to connect to any node, and eventually be connected to the workload on a container, even if not the node originally contacted.

### API Call Flow

Actor --> kube-apiserver --> etcd --> kube-apiserver --> kube-controller-manager --> kube-apiserver --> kube-scheduler --> kube-apiserver --> kubelet & kube-proxy --> Docker/cri-o engine --> containers

### Node

A node is an API object created outside the cluster representing an instance. While a master must be Linux, worker nodes can also be Microsoft Windows Server 2019. Once the node has the necessary software installed, it is ingested into the API server.

At the moment, you can create a master node with `kubeadm init` and worker nodes by passing `join` . In the near future, secondary master nodes and/or etcd nodes may be joined.

If the kube-apiserver cannot communicate with the kubelet on a node for 5 minutes, the default **NodeLease** will schedule the node for deletion and the **NodeStatus** will change from ready. The pods will be evicted once a connection is re-established. They are no longer forcibly removed and rescheduled by the cluster.

Each node object exists in the `kube-node-lease` namespace. To remove a node from the cluster, first use `kubectl delete node <node-name>` to remove it from the API server. This will cause pods to be evacuated. Then, use `kubeadm reset` to remove cluster-specific information. You may also need to remove iptables information, depending on if you plan on re-using the node.

To view CPU, memory, and other resource usage, requests and limits use the `kubectl describe node` command. The output will show capacity and pods allowed, as well as details on current pods and resource utilization.

### Single IP per Pod

A pod represents a group of co-located containers with some associated data volumes. All containers in a pod share the same network namespace.

![Pod Network](../../images/pod-network.png "Pod Network")

The graphic shows a pod with two containers, A and B, and two data volumes, 1 and 2. Containers A and B share the network namespace of a third container, known as the pause container. The pause container is used to get an IP address, then all the containers in the pod will use its network namespace. Volumes 1 and 2 are shown for completeness.

To communicate with each other, containers within pods can use the loopback interface, write to files on a common filesystem, or via inter-process communication (IPC). There is now a network plugin from HPE Labs which allows multiple IP addresses per pod, but this feature has not grown past this new plugin.

Starting as an alpha feature in 1.16 is the ability to use IPv4 and IPv6 for pods and services. In the current version, when creating a service, you need to create the network for each address family separately.

### Container to Outside Path

This graphic shows a node with a single, dual-container pod. A NodePort service connects the Pod to the outside network. 

![Container Network](../../images/container-network.png "Container Network")

Even though there are two containers, they share the same namespace and the same IP address, which would be configured by kubectl working with kube-proxy. The IP address is assigned before the containers are started, and will be inserted into the containers. The container will have an interface like **eth0@tun10**. This IP is set for the life of the pod.

The endpoint is created at the same time as the service. Note that it uses the pod IP address, but also includes a port. The service connects network traffic from a node high-number port to the endpoint using iptables with ipvs on the way. The kube-controller-manager handles the watch loops to monitor the need for endpoints and services, as well as any updates or deletions.

### Networking Setup

Getting all the previous components running is a common task for system administrators who are accustomed to configuration management. But, to get a fully functional Kubernetes cluster, the network will need to be set up properly, as well.

A detailed explanation about the Kubernetes networking model can be seen on the Cluster Networking page in the Kubernetes documentation.

If you have experience deploying virtual machines (VMs) based on IaaS solutions, this will sound familiar. The only caveat is that, in Kubernetes, the lowest compute unit is not a container, but what we call a pod. 

A pod is a group of co-located containers that share the same IP address. From a networking perspective, a pod can be seen as a virtual machine of physical hosts. The network needs to assign IP addresses to pods, and needs to provide traffic routes between all pods on any nodes. 

The three main networking challenges to solve in a container orchestration system are:

* Coupled container-to-container communication (solved by the pod concept).
* Pod-to-pod communication.
* External-to-pod communication (solved by the services concept, which we will discuss later).

Kubernetes expects the network configuration to enable pod-to-pod communication to be available; it will not do it for you.

Tim Hockin, one of the lead Kubernetes developers, has created a very useful slide deck to understand the Kubernetes networking: An Illustrated Guide to Kubernetes Networking.

### CNI Network Configuration File

To provide container networking, Kubernetes is standardizing on the Container Network Interface (CNI) specification. Since v1.6.0, the goal of kubeadm (the Kubernetes cluster bootstrapping tool) has been to use CNI, but you may need to recompile to do so.

CNI is an emerging specification with associated libraries to write plugins that configure container networking and remove allocated resources when the container is deleted. Its aim is to provide a common interface between the various networking solutions and container runtimes. As the CNI specification is language-agnostic, there are many plugins from Amazon ECS, to SR-IOV, to Cloud Foundry, and more.

With CNI, you can write a network configuration file:

``` json
{
    "cniVersion": "0.2.0",
    "name": "mynet",
    "type": "bridge",
    "bridge": "cni0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.22.0.0/16",
        "routes": [
            { "dst": "0.0.0.0/0" }
             ]
     }
}
```

This configuration defines a standard Linux bridge named cni0, which will give out IP addresses in the subnet 10.22.0.0./16. The bridge plugin will configure the network interfaces in the correct namespaces to define the container network properly.

### Pod-to-Pod Communication

While a CNI plugin can be used to configure the network of a pod and provide a single IP per pod, CNI does not help you with pod-to-pod communication across nodes.

The requirement from Kubernetes is the following:

* All pods con communicate with each other across nodes
* All nodes can communicate with all pods
* No Network Address Translation (NAT)

Basically, all IPs involved (nodes and pods) are routable without NAT. This can be achieved at the physical network infrastructure if you have access to it (e.g. GKE). Or, this can be achieved with a software defined overlay with solutions like:

* Weave
* Flannel
* Calico
* Romana

### Mesos

At a high level, there is nothing different between Kubernetes and other clustering systems.

A central manager exposes an API, a scheduler places the workloads on a set of nodes, and the state of the cluster is stored in a persistent layer.

For example, you could compare Kubernetes with Mesos, and you would see the similarities. In Kubernetes, however, the persistence layer is implemented with etcd, instead of Zookeeper for Mesos.

You should also consider systems like OpenStack and CloudStack. Think about what runs on their head node, and what runs on their worker nodes. How do they keep state? How do they handle networking? If you are familiar with those systems, Kubernetes will not seem that different.

What really sets Kubernetes apart is its features oriented towards fault-tolerance, self-discovery, and scaling, coupled with a mindset that is purely API-driven.

### Lab 4.1. Basic Node Maintenance

In this section we will backup the **etcd** database then update the version of Kubernetes used on control plane nodes and worker nodes.

#### etcd Backup

``` sh
sudo grep data-dir /etc/kubernetes/manifests/etcd.yaml
kubectl -n kube-system exec -it etcd-master -- sh
etcdctl -h # Inside the container
cd /etc/kubernetes/pki/etcd
pwd
echo * # ls is not present as image is minimized
exit
kubectl -n kube-system exec -it etcd-master -- sh -c "ETCDCTL_API=3 ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key etcdctl endpoint health"
kkubectl -n kube-system exec -it etcd-master -- sh -c "ETCDCTL_API=3 etcdctl --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key --cacert=/etc/kubernetes/pki/etcd/ca.crt --endpoints=https://127.0.0.1:2379 member list"
kubectl -n kube-system exec -it etcd-master -- sh -c "ETCDCTL_API=3 ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key etcdctl --endpoints=https://127.0.0.1:2379 --write-out=table member list"
kubectl -n kube-system exec -it etcd-master -- sh -c "ETCDCTL_API=3 ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key  etcdctl --endpoints=https://127.0.0.1:2379 snapshot save /var/lib/etcd/snapshot.db"
sudo ls -l /var/lib/etcd/
mkdir $HOME/backup
sudo cp /var/lib/etcd/snapshot.db $HOME/backup/snapshot.db-$(date +%m-%d-%y)
sudo cp /root/kubeadm-config.yaml $HOME/backup/
sudo cp -r /etc/kubernetes/pki/etcd $HOME/backup/
```

#### Upgrading the Cluster

From **master** node:

``` sh
sudo apt update
sudo apt-cache madison kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.20.1-00
sudo apt-mark hold kubeadm
sudo kubeadm version
kubectl drain master --ignore-daemonsets
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.20.1
kubectl get node # Previous version because the daemons are not restarted and all software not updated
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.20.1-00 kubectl=1.20.1-00
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl uncordon master
kubectl get node
```

From **worker** node:

``` sh
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.20.1-00
sudo apt-mark hold kubeadm

```

From **master** node:

``` sh
kubectl drain worker --ignore-daemonsets
```

From **worker** node:

``` sh
sudo kubeadm upgrade node
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.20.1-00 kubectl=1.20.1-00
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

From **master** node:

``` sh
kubectl get node
kubectl uncordon worker
kubectl get nodes
```

### Lab 4.2. Working with CPU and Memory Constraints

From **master** node:

``` sh
kubectl create deployment hog --image vish/stress
kubectl get deployments
kubectl describe deployment hog
kubectl get deployment hog -o yaml
kubectl get deployment hog -o yaml > hog.yaml
vim hog.yaml # Adding memory limits to 4Gi and memory requests to 2500Mi, also removing creationTimestamp and other settings
kubectl replace -f hog.yaml
kubectl get deployment hog -o yaml
kubectl get po
kubectl logs hog-775c7c858f-v9bjv
top
vim hog.yaml # Adding CPU limits to 1 and CPU requests to 0.5, memory requests to 500Mi and adding args: -cpus, "2", -mem-total, "950Mi", -mem-alloc-size, "100Mi", -mem-alloc-sleep, "1s"
kubectl delete deployment hog
kubectl create -f hog.yaml
```

### Lab 4.3. Resource Limits for a Namespace

From **master** node:

``` sh
kubectl create namespace low-usage-limit
kubectl get namespace
vim low-resource-range.yaml
```

``` yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: low-resource-range
spec:
  limits:

  + default:

      cpu: 1
      memory: 500Mi
    defaultRequest:
      cpu: 0.5
      memory: 100Mi
    type: Container
```

``` sh
kubectl --namespace=low-usage-limit create -f low-resource-range.yaml
kubectl get LimitRange # No resource found due namespace is not default
kubectl get LimitRange --all-namespaces
kubectl -n low-usage-limit create deployment limited-hog --image vish/stress
kubectl get deployments --all-namespaces
kubectl -n low-usage-limit get pods
kubectl -n low-usage-limit get pod limited-hog-7c5ddc8c74-84nbm -o yaml
cp hog.yaml hog2.yaml
vim hog2.yaml # Change namespace to low-usage-limit
kubectl create -f hog2.yaml
kubectl get deployments --all-namespaces
kubectl -n low-usage-limit delete deployment hog
kubectl delete deployment hog
```

## APIs and Access

### API Access

Kubernetes has a powerful REST-based API. The entire architecture is API-driven. Knowing where to find resource endpoints and understanding how the API changes between versions can be important to ongoing administrative tasks, as there is much ongoing change and growth. Starting with v1.16 deprecated objects are no longer honored by the API server.

As we learned in the Kubernetes Architecture chapter, the main agent for communication between cluster agents and from outside the cluster is the kube-apiserver. A curl query to the agent will expose the current API groups. Groups may have multiple versions, which evolve independently of other groups, and follow a domain-name format with several names reserved, such as single-word domains, the empty group, and any name ending in .k8s.io.

### RESTful

kubectl makes API calls on your behalf, responding to typical HTTP verbs (GET, POST, DELETE). You can also make calls externally, using curl or other program. With the appropriate certificates and keys, you can make requests, or pass JSON files to make configuration changes.   

``` sh
curl --cert userbob.pem --key userBob-key.pem --cacert /path/to/ca.pem https://k8sServer:6443/api/v1/pods
```

The ability to impersonate other users or groups, subject to RBAC configuration, allows a manual override authentication. This can be helpful for debugging authorization policies of other users.

### Checking Access

While there is more detail on security in a later chapter, it is helpful to check the current authorizations, both as an administrator, as well as another user. The following shows what user bob could do in the default namespace and the developer namespace, using the auth can-i subcommand to query: 

``` sh
kubectl auth can-i create deployments
kubectl auth can-i create deployments --as bob
kubectl auth can-i create deployments --as bob --namespace developer
```

There are currently three APIs which can be applied to set who and what can be queried:

* **SelfSubjectAccessReview**: Access review for any user, helpful for delegating to others. 
* **LocalSubjectAccessReview**: ​Review is restricted to a specific namespace.
* **SelfSubjectRulesReview**: A review which shows allowed actions for a user within a particular namespace. 

The use of **reconcile** allows a check of authorization necessary to create an object from a file. No output indicates the creation would be allowed.

### Optimistic Concurrency

The default serialization for API calls must be JSON. There is an effort to use Google's protobuf serialization, but this remains experimental. While we may work with files in a YAML format, they are converted to and from JSON. 

Kubernetes uses the **resourceVersion** value to determine API updates and implement optimistic concurrency. In other words, an object is not locked from the time it has been read until the object is written. 

Instead, upon an updated call to an object, the **resourceVersion** is checked, and a **409 CONFLICT** is returned, should the number have changed. The **resourceVersion** is currently backed via the **modifiedIndex** parameter in the etcd database, and is unique to the namespace, kind, and server. Operations which do not change an object, such as **WATCH** or **GET**, do not update this value.

### Using Annotations

Labels are used to work with objects or collections of objects; annotations are not.

Instead, annotations allow for metadata to be included with an object that may be helpful outside of the Kubernetes object interaction. Similar to labels, they are key to value maps. They are also able to hold more information, and more human-readable information than labels. 

Having this kind of metadata can be used to track information such as a timestamp, pointers to related objects from other ecosystems, or even an email from the developer responsible for that object's creation. 

The annotation data could otherwise be held in an exterior database, but that would limit the flexibility of the data. The more this metadata is included, the easier it is to integrate management and deployment tools or shared client libraries. 

For example, to annotate only Pods within a namespace, you can overwrite the annotation, and finally delete it:

``` sh
kubectl annotate pods --all description='Production Pods' -n prod
kubectl annotate --overwrite pod webpod description="Old Production Pods" -n prod
kubectl -n prod annotate pod webpod description-
```

### Simple Pod

As discussed earlier, a Pod is the lowest compute unit and individual object we can work with in Kubernetes. It can be a single container, but often, it will consist of a primary application container and one or more supporting containers. 

Below is an example of a simple pod manifest in YAML format. You can see the **apiVersion** (it must match the existing API group), the **kind** (the type of object to create), the **metadata** (at least a name), and its **spec** (what to create and parameters), which define the container that actually runs in this pod: 

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: firstpod
spec:
  containers:

  + image: nginx

    name: stan 
```

You can use the **kubectl create** command to create this pod in Kubernetes. Once it is created, you can check its status with **kubectl get pods**: 

``` sh
kubectl create -f simple.yaml
kubectl get pods
kubectl get pod firstpod -o yaml
kubectl get pod firstpod -o json
```

### Manage API Resources with kubectl

Kubernetes exposes resources via RESTful API calls, which allows all resources to be managed via HTTP, JSON or even XML, the typical protocol being HTTP. The state of the resources can be changed using standard HTTP verbs (e.g. GET, POST, PATCH, DELETE, etc.).

**kubectl** has a verbose mode argument which shows details from where the command gets and updates information. Other output includes **curl** commands you could use to obtain the same result. While the verbosity accepts levels from zero to any number, there is currently no verbosity value greater than ten. You can check this out for **kubectl get**:

``` sh
kubectl --v=10 get pods firstpod
```

If you delete this pod, you will see that the HTTP method changes from XGET to XDELETE.

``` sh
kubectl --v=10 delete pods firstpod
```

### Access from Outside the Cluster

The primary tool used from the command line will be **kubectl**, which calls **curl** on your behalf. You can also use the **curl** command from outside the cluster to view or make changes.

The basic server information, with redacted TLS certificate information, can be found in the output of:

``` sh
kubectl config view
```

If you view the verbose output from a previous page, you will note that the first line references a configuration file where this information is pulled from, *~/.kube/config*.

Without the certificate authority, key and certificate from this file, only insecure **curl** commands can be used, which will not expose much due to security settings. We will use **curl** to access our cluster using TLS in an upcoming lab.

### ~/.kube/config

``` yaml
apiVersion: v1
clusters:

* cluster:

    certificate-authority-data: LS0tLS1CRUdF.....
    server: https://10.128.0.3:6443 
    name: kubernetes
contexts:

* context:

    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:

* name: kubernetes-admin

  user:
    client-certificate-data: LS0tLS1CRUdJTib.....
    client-key-data: LS0tLS1CRUdJTi....
```

* **apiVersion**: As with other objects, this instructs the kube-apiserver where to assign the data.
* **clusters**: This key contains the name of the cluster, as well as where to send the API calls. The **certificate-authority-data** is passed to authenticate the curl request.
* **contexts**: This is a setting which allows easy access to multiple clusters, possibly as various users, from one configuration file. It can be used to set **namespace**, **user**, and **cluster**.
* **current-context**: This shows which cluster and user the **kubectl** command would use. These settings can also be passed on a per-command basis.
* **kind**: Every object within Kubernetes must have this setting; in this case, a declaration of object type **Config**.
* **preferences**: Currently not used, this is an optional settings for the **kubectl** command, such as colorizing output.
* **users**: A nickname associated with client credentials, which can be client key and certificate, username and password, and a token. Token and username/password are mutually exclusive. These can be configured via the **kubectl config set-credentials** command.

### Namespaces

The term namespace is used to reference both the kernel feature and the segregation of API objects by Kubernetes. Both are means to keep resources distinct. 

Every API call includes a namespace, using **default** if not otherwise declared: **https://10.128.0.3:6443/api/v1/namespaces/default/pods**. 

Namespaces, a Linux kernel feature that segregates system resources, are intended to isolate multiple groups and the resources they have access to work with via quotas. Eventually, access control policies will work on namespace boundaries, as well. One could use labels to group resources for administrative reasons. 

There are four namespaces when a cluster is first created:

* **default**: This is where all the resources are assumed, unless set otherwise.
* **kube-node-lease**: This is the namespace where worker node lease information is kept.
* **kube-public**: A namespace readable by all, even those not authenticated. General information is often included in this namespace.
* **kube-system**: This namespace contains infrastructure pods.

Should you want to see all the resources on a system, you must pass the **--all-namespaces** option to the **kubectl** command.

``` sh
kubectl get ns
kubectl create ns linuxcon
kubectl describe ns linuxcon
kubectl get ns/linuxcon -o yaml
kubectl delete ns/linuxcon
```

The above commands show how to view, create and delete namespaces. Note that the describe subcommand shows several settings, such as Labels, Annotations, resource quotas, and resource limits, which we will discus later in the course.

Once a namespace has been created, you can reference it via YAML when creating a resource: 

``` yaml
apiVersion: V1
kind: Pod
metadata:
    name: redis
    namespace: linuxcon
...
```

### API Resources with kubectl

All API resources exposed are available via **kubectl**. To get more information, do **kubectl help**.

``` sh
kubectl [command] [type] [Name] [flag]
```

![K8s API Resources](../../images/k8s-resources.png "K8s API Resources")

### Additional Resource Methods

In addition to basic resource management via REST, the API also provides some extremely useful endpoints for certain resources. 

For example, you can access the logs of a container, exec into it, and watch changes to it with the following endpoints: 

``` sh
curl --cert /tmp/client.pem --key /tmp/client-key.pem \
--cacert /tmp/ca.pem -v -XGET \ 
 https://10.128.0.3:6443/api/v1/namespaces/default/pods/firstpod/log
```

This would be the same as the following. If the container does not have any standard out, there would be no logs. 

``` sh
kubectl logs firstpod
```

There are other calls you could make, following the various API groups on your cluster: 

* GET /api/v1/namespaces/{namespace}/pods/{name}/exec
* GET /api/v1/namespaces/{namespace}/pods/{name}/log
* GET /api/v1/watch/namespaces/{namespace}/pods/{name}

### Swagger and OpenAPI

The entire Kubernetes API uses a Swagger specification. This is evolving towards the OpenAPI initiative. It is extremely useful, as it allows, for example, to auto-generate client code. All the stable resources definitions are available on the documentation site.

You can browse some of the API groups via a Swagger UI on the OpenAPI [Specification web page](https://swagger.io/specification/).

### API Maturity

The use of API groups and different versions allows for development to advance without changes to an existing group of APIs. This allows for easier growth and separation of work among separate teams. While there is an attempt to maintain some consistency between API and software versions, they are only indirectly linked.

The use of JSON and Google's Protobuf serialization scheme will follow the same release guidelines.

* Alpha: An Alpha level release, noted with alpha in the name, may be buggy and is disabled by default. Features could change or disappear at any time. Only use these features on a test cluster which is often rebuilt.
* Beta: The Beta level, found with beta in the name, has more well-tested code and is enabled by default. It also ensures that, as changes move forward, they will be tested for backwards compatibility between versions. It has not been adopted and tested enough to be called stable. You can expect some bugs and issues.
* Stable: Use of the Stable version, denoted by only an integer which may be preceded by the letter v, is for stable APIs.

### Lab 5.1. Configuring TLS Access

``` sh
less $HOME/.kube/config
export client=$(grep client-cert $HOME/.kube/config | cut -d" " -f 6)
echo $client
export key=$(grep client-key-data $HOME/.kube/config | cut -d " " -f 6)
echo $key
export auth=$(grep certificate-authority-data $HOME/.kube/config | cut -d " " -f 6)
echo $auth
echo $client | base64 -d - > ./client.pem
echo $key | base64 -d - > ./client-key.pem
echo $auth | base64 -d - > ./ca.pem
kubectl config view | grep server
curl --cert ./client.pem --key ./client-key.pem --cacert ./ca.pem https://k8smaster:6443/api/v1/pods
vim curlpod.json
```

``` json
{
  "kind": "Pod",
  "apiVersion": "v1",
  "metadata": {
    "name": "curlpod",
    "namespace": "default",
    "labels": {
      "name": "examplepod"
      }
  },
  "spec": {
    "containers": [{
      "name": "nginx",
      "image": "nginx",
      "ports": [{"containerPort": 80}]
    }]
  }
}
```

``` sh
curl --cert ./client.pem --key ./client-key.pem --cacert ./ca.pem https://k8smaster:6443/api/v1/namespaces/default/pods -XPOST -H'Content-Type: application/json' -d@curlpod.json
kubectl get pods
```

### Lab 5.2. Explore API Calls

``` sh
sudo apt-get install -y strace
kubectl get endpoints
strace kubectl get endpoints
cd /home/guillermov/.kube/cache/discovery/
ls
cd k8smaster_6443/
ls
find .
python3 -m json.tool v1/serverresources.json
python3 -m json.tool v1/serverresources.json | less
kubectl get ep
python3 -m json.tool v1/serverresources.json | grep kind
python3 -m json.tool apps/v1/serverresources.json | grep kind
kubectl delete po curlpod
```

## API Objects

### v1 API Group

The v1 API group is no longer a single group, but rather a collection of groups for each main object category. For example, there is a v1 group, a storage.k8s.io/v1 group, and an rbac.authorization.k8s.io/v1, etc. Currently, there are eight v1 groups.

* **Node**: Represents a machine - physical or virtual - that is part of your Kubernetes cluster. You can get more information about nodes with the **kubectl get nodes** command. You can turn on and off the scheduling to a node with the **kubectl cordon/uncordon** commands.
* **Service Account**: Provides an identifier for processes running in a pod to access the API server and performs actions that it is authorized to do.
* **Resource Quota**: It is an extremely useful tool, allowing you to define quotas per namespace. For example, if you want to limit a specific namespace to only run a given number of pods, you can write a **resourcequota** manifest, create it with **kubectl** and the quota will be enforced.
* **Endpoint**: Generally, you do not manage endpoints. They represent the set of IPs for pods that match a particular service. They are handy when you want to check that a service actually matches some running pods. If an endpoint is empty, then it means that there are no matching pods and something is most likely wrong with your service definition.

### Discovering API Groups

We can take a closer look at the output of the request for current APIs. Each of the name values can be appended to the URL to see details of that group. For example, you could drill down to find included objects at this URL: **https://localhost:6443/apis/apiregistration.k8s.io/v1beta1**.

If you follow this URL, you will find only one resource, with a name of apiservices. If it seems to be listed twice, the lower output is for status. You'll notice that there are different verbs or actions for each. Another entry is if this object is namespaced, or restricted to only one namespace. In this case, it is not. 

``` sh
curl https://localhost:6443/apis --header "Authorization: Bearer $token" -k
```

``` json
{
  "kind": "APIGroupList",
  "apiVersion": "v1",
  "groups": [
    {
      "name": "apiregistration.k8s.io",
      "versions": [
        {
          "groupVersion": "apiregistration.k8s.io/v1",
          "version": "v1"
        }
      ],
      "preferredVersion": {
        "groupVersion": "apiregistration.k8s.io/v1",
        "version": "v1"
      }
....
    }
  ]
}
```

### Deploying an Application

Using the **kubectl create** command, we can quickly deploy an application. We have looked at the Pods created running the application, like nginx. Looking closer, you will find that a Deployment was created, which manages a ReplicaSet, which then deploys the Pod.

* **Deployment**: It is a controller which manages the state of ReplicaSets and the pods within. The higher level control allows for more flexibility with upgrades and administration. Unless you have a good reason, use a deployment.
* **ReplicaSet**: Orchestrates individual pod lifecycle and updates. These are newer versions of Replication Controllers, which differ only in selector support.
* **Pod**: As we've already mentioned, it is the lowest unit we can manage; it runs the application container, and possibly support containers.

### DaemonSets

Should you want to have a logging application on every node, a DaemonSet may be a good choice. The controller ensures that a single pod, of the same type, runs on every node in the cluster. When a new node is added to the cluster, a Pod, same as deployed on the other nodes, is started. When the node is removed, the DaemonSet makes sure the local Pod is deleted. DaemonSets are often used for logging, metrics and security pods, and can be configured to avoid nodes.

As usual, you get all the CRUD operations via **kubectl**: ​

``` sh
kubectl get daemonsets
kubectl get ds
```

### StatefulSets

According to Kubernetes documentation, a StatefulSet is the workload API object used to manage stateful applications. Pods deployed using a StatefulSet use the same Pod specification. How this is different than a Deployment is that a StatefulSet considers each Pod as unique and provides ordering to Pod deployment.

In order to track each Pod as a unique object, the controllers uses an identity composed of stable storage, stable network identity, and an ordinal. This identity remains with the node, regardless of which node the Pod is running on at any one time.

The default deployment scheme is sequential, starting with 0, such as **app-0**, **app-1**, **app-2**, etc. A following Pod will not launch until the current Pod reaches a running and ready state. They are not deployed in parallel.

StatefulSets are stable as of Kubernetes v1.9.​

### Autoscaling

In the autoscaling group we find the **Horizontal Pod Autoscalers (HPA)**. This is a stable resource. HPAs automatically scale Replication Controllers, ReplicaSets, or Deployments based on a target of 50% CPU usage by default. The usage is checked by the kubelet every 30 seconds, and retrieved by the Metrics Server API call every minute. HPA checks with the Metrics Server every 30 seconds. Should a Pod be added or removed, HPA waits 180 seconds before further action. 

Other metrics can be used and queried via REST. The autoscaler does not collect the metrics, it only makes a request for the aggregated information and increases or decreases the number of replicas to match the configuration. 

The **Cluster Autoscaler (CA)** adds or removes nodes to the cluster, based on the inability to deploy a Pod or having nodes with low utilization for at least 10 minutes. This allows dynamic requests of resources from the cloud provider and minimizes expenses for unused nodes. If you are using CA, nodes should be added and removed through **cluster-autoscaler-** commands. Scale-up and down of nodes is checked every 10 seconds, but decisions are made on a node every 10 minutes. Should a scale-down fail, the group will be rechecked in 3 minutes, with the failing node being eligible in five minutes. The total time to allocate a new node is largely dependent on the cloud provider. 

Another project still under development is the Vertical Pod Autoscaler. This component will adjust the amount of CPU and memory requested by Pods.

### Jobs

Jobs are part of the **batch** API group. They are used to run a set number of pods to completion. If a pod fails, it will be restarted until the number of completion is reached.

While they can be seen as a way to do batch processing in Kubernetes, they can also be used to run one-off pods. A Job specification will have a parallelism and a completion key. If omitted, they will be set to one. If they are present, the parallelism number will set the number of pods that can run concurrently, and the completion number will set how many pods need to run successfully for the Job itself to be considered done. Several Job patterns can be implemented, like a traditional work queue.

Cronjobs work in a similar manner to Linux jobs, with the same time syntax. There are some cases where a job would not be run during a time period or could run twice; as a result, the requested Pod should be idempotent.

An option spec field is **.spec.concurrencyPolicy**, which determines how to handle existing jobs, should the time segment expire. If set to **Allow**, the default, another concurrent job will be run. If set to **Forbid**, the current job continues and the new job is skipped. A value of **Replace** cancels the current job and starts a new job in its place.

### RBAC

The last API resources that we will look at are in the **rbac.authorization.k8s.io** group. We actually have four resources: ClusterRole, Role, ClusterRoleBinding, and RoleBinding. They are used for Role Based Access Control (RBAC) to Kubernetes.

``` sh
curl localhost:8080/apis/rbac.authorization.k8s.io/v1
```

These resources allow us to define Roles within a cluster and associate users to these Roles. For example, we can define a Role for someone who can only read pods in a specific namespace, or a Role that can create deployments, but no services. We will talk more about RBAC later in the course, in the Security chapter.

### Lab 6.1. RESTful API Access

``` sh
kubectl config view
kubectl get secrets --all-namespaces
kubectl get secrets
kubectl describe secret default-token-749mf
export token=$(kubectl describe secret default-token-749mf | grep ^token | cut -f7 -d' ')
curl https://k8smaster:6443/apis --header "Authorization: Bearer $token" -k
curl https://k8smaster:6443/api/v1 --header "Authorization: Bearer $token" -k
curl https://k8smaster:6443/api/v1/namespaces --header "Authorization: Bearer $token" -k # Forbidden
kubectl run -i -t busybox --image=busybox --restart=Never
ls /var/run/secrets/kubernetes.io/serviceaccount/
exit
kubectl delete pod busybox
```

### Lab 6.2. Using the Proxy

``` sh
kubectl proxy -h
kubectl proxy --api-prefix=/ & # 30489
curl http://127.0.0.1:8001/api/
curl http://127.0.0.1:8001/api/v1/namespaces
kill 30489
```

### Lab 6.3. Working with Jobs

``` sh
vim job.yaml
```

``` yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sleepy
spec:
  template:
    spec:
      containers:

      - name: resting

        image: busybox
        command: ["/bin/sleep"]
        args: ["3"]
      restartPolicy: Never
```

``` sh
kubectl create -f job.yaml
kubectl get job
kubectl describe jobs.batch sleepy
kubectl get job
kubectl get jobs.batch sleepy -o yaml
kubectl delete jobs.batch sleepy
vim job.yaml
```

``` yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sleepy
spec:
  completions: 5
  template:
    spec:
      containers:

      - name: resting

        image: busybox
        command: ["/bin/sleep"]
        args: ["3"]
      restartPolicy: Never
```

``` sh
kubectl create -f job.yaml
kubectl get jobs.batch
kubectl get pods
kubectl get jobs
kubectl delete jobs.batch sleepy
vim job.yaml
```

``` yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sleepy
spec:
  completions: 5
  parallelism: 2
  template:
    spec:
      containers:

      - name: resting

        image: busybox
        command: ["/bin/sleep"]
        args: ["3"]
      restartPolicy: Never
```

``` sh
kubectl create -f job.yaml
kubectl get pods
kubectl get jobs
vim job.yaml
```

``` yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sleepy
spec:
  completions: 5
  parallelism: 2
  activeDeadlineSeconds: 15
  template:
    spec:
      containers:

      - name: resting

        image: busybox
        command: ["/bin/sleep"]
        args: ["5"]
      restartPolicy: Never
```

``` sh
kubectl delete jobs.batch sleepy
kubectl create -f job.yaml
kubectl get pods
kubectl get jobs
kubectl get job sleepy -o yaml
kubectl delete jobs.batch sleepy
cp job.yaml cronjob.yaml
vim cronjob.yaml
```

``` yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: sleepy
spec:
  schedule: "*/2 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:

          - name: resting

            image: busybox
            command: ["/bin/sleep"]
            args: ["5"]
          restartPolicy: Never
```

``` sh
kubectl create -f cronjob.yaml
kubectl get cronjobs.batch
kubectl get jobs.batch
kubectl get cronjobs.batch
kubectl get jobs.batch # 2 minutes later
kubectl get jobs.batch # 2 minutes later
vim cronjob.yaml
```

``` yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: sleepy
spec:
  schedule: "*/2 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          activeDeadlineSeconds: 10
          containers:

          - name: resting

            image: busybox
            command: ["/bin/sleep"]
            args: ["30"]
          restartPolicy: Never
```

``` sh
kubectl delete cronjobs.batch sleepy
kubectl create -f cronjob.yaml
kubectl get jobs
kubectl get cronjobs.batch
kubectl get jobs
kubectl get jobs
kubectl get cronjobs.batch
kubectl delete cronjobs.batch sleepy
```

## Managing State with Deployments

The default controller for a container deployed via **kubectl run** is a Deployment. While we have been working with them already, we will take a closer look at configuration options.

As with other objects, a deployment can be made from a YAML or JSON spec file. When added to the cluster, the controller will create a ReplicaSet and a Pod automatically. The containers, their settings and applications can be modified via an update, which generates a new ReplicaSet, which, in turn, generates new Pods.

The updated objects can be staged to replace previous objects as a block or as a rolling update, which is determined as part of the deployment specification. Most updates can be configured by editing a YAML file and running **kubectl apply**. You can also use **kubectl edit** to modify the in-use configuration. Previous versions of the ReplicaSets are kept, allowing a rollback to return to a previous configuration.

We will also talk more about labels. Labels are essential to administration in Kubernetes, but are not an API resource. They are user-defined key-value pairs which can be attached to any resource, and are stored in the metadata. Labels are used to query or select resources in your cluster, allowing for flexible and complex management of the cluster. 

As a label is arbitrary, you could select all resources used by developers, or belonging to a user, or any attached string, without having to figure out what kind or how many of such resources exist.

### Deployments

**ReplicationControllers** (RC) ensure that a specified number of pod replicas is running at any one time. ReplicationControllers also give you the ability to perform rolling updates. However, those updates are managed on the client side. This is problematic if the client loses connectivity, and can leave the cluster in an unplanned state. To avoid problems when scaling the ReplicationControllers on the client side, a new resource was introduced in the **apps/v1** API group: Deployments. s

Deployments allow server-side updates to pods at a specified rate. They are used for canary and other deployment patterns. Deployments generate ReplicaSets, which offer more selection features than ReplicationControllers, such as **matchExpressions**. 

``` sh
kubectl create deployment dev-web --image=nginx:1.13.7-alpine
```

### Object Relationship

Here you can see the relationship between objects from the container, which Kubernetes does not directly manage, up to the deployment.

![Nested Objects](../../images/nested-objects.png "Nested Objects")

The boxes and shapes are logical, in that they represent the controllers, or watch loops, running as a thread of the kube-controller-manager. Each controller queries the kube-apiserver for the current state of the object they track. The state of each object on a worker node is sent back from the local kubelet.

The graphic in the upper left represents a container running nginx 1.11. Kubernetes does not directly manage the container. Instead, the kubelet daemon checks the pod specifications by asking the container engine, which could be Docker or cri-o, for the current status. The graphic to the right of the container shows a pod which represents a watch loop checking the container status. kubelet compares the current pod spec against what the container engine replies and will terminate and restart the pod if necessary.

A multi-container pod is shown next. While there are several names used, such as *sidecar* or *ambassador*, these are all multi-container pods. The names are used to indicate the particular reason to have a second container in the pod, instead of denoting a new kind of pod.

On the lower left we see a **replicaSet**. This controller will ensure you have a certain number of pods running. The pods are all deployed with the same **podSpec**, which is why they are called replicas. Should a pod terminate or a new pod be found, the replicaSet will create or terminate pods until the current number of running pods matches the specifications. Any of the current pods could be terminated should the spec demand fewer pods running.

The graphic in the lower right shows a deployment. This controller allows us to manage the versions of images deployed in the pods. Should an edit be made to the deployment, a new **replicaSet** is created, which will deploy pods using the new **podSpec**. The deployment will then direct the old **replicaSet** to shut down pods as the new **replicaSet** pods become available. Once the old pods are all terminated, the deployment terminates the old **replicaSet** and the deployment returns to having only one **replicaSet** running.

### Deployment Details

``` sh
kubectl get deployments,rs,pods -o yaml
kubectl get deployments,rs,pods -o json
```

``` yaml
apiVersion: v1
items:

* apiVersion: apps/v1

  kind: Deployment
```

* **apiVersion**: A value of **v1** shows that this object is considered to be a stable resource. In this case, it is not the deployment. It is a reference to the **List** type. 
* **items**: As the previous line is a **List**, this declares the list of items the command is showing.
* **apiVersion**: The dash is a YAML indication of the first item, which declares the **apiVersion** of the object as **apps/v1**. This indicates the object is considered stable. Deployments are controller used in most cases.
* **kind**: This is where the type of object to create is declared, in this case, a deployment.

### Deployment Configuration Metadata

Continuing with the YAML output, we see the next general block of output concerns the metadata of the deployment. This is where we would find labels, annotations, and other non-configuration information. Note that this output will not show all possible configuration. Many settings which are set to false by default are not shown, like **podAffinity** or **nodeAffinity**.

``` yaml
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: 2017-12-21T13:57:07Z
  generation: 1
  labels:
    app: dev-web
  name: dev-web
  namespace: default
  resourceVersion: "774003"
  selfLink: /apis/apps/v1/namespaces/default/deployments/dev-web
  uid: d52d3a63-e656-11e7-9319-42010a800003
```

* **annotations**: These values do not configure the object, but provide further information that could be helpful to third-party applications or administrative tracking. Unlike labels, they cannot be used to select an object with **kubectl**.
* **creationTimestamp**: Shows when the object was originally created. Does not update if the object is edited.
* **generation**: How many times this object has been edited, such as changing the number of replicas, for example.
* **labels**: Arbitrary strings used to select or exclude objects for use with **kubectl**, or other API calls. Helpful for administrators to select objects outside of typical object boundaries.
* **name**: This is a *required* string, which we passed from the command line. The name must be unique to the namespace.
* **resourceVersion**: A value tied to the etcd database to help with concurrency of objects. Any changes to the database will cause this number to change.
* **selfLink**: References how the kube-apiserver will ingest this information into the API.
* **uid**: Remains a unique ID for the life of the object.

### Deployment Configuration Spec

There are two spec declarations for the deployment. The first will modify the ReplicaSet created, while the second will pass along the Pod configuration. 

``` yaml
spec:  
  progressDeadlineSeconds: 600   
  replicas: 1  
  revisionHistoryLimit: 10   
  selector:     
    matchLabels:       
      app: dev-web  
  strategy:     
    rollingUpdate:       
      maxSurge: 25%        
      maxUnavailable: 25%     
    type: RollingUpdate
```

* **spec**: A declaration that the following items will configure the object being created.
* **progressDeadlineSeconds**: Time in seconds until a progress error is reported during a change. Reasons could be quotas, image issues, or limit ranges.
* **replicas**: As the object being created is a ReplicaSet, this parameter determines how many Pods should be created. If you were to use kubectl edit and change this value to two, a second Pod would be generated.
* **revisionHistoryLimit**: How many old ReplicaSet specifications to retain for rollback.
* **selector**: A collection of values ANDed together. All must be satisfied for the replica to match. Do not create Pods which match these selectors, as the deployment controller may try to control the resource, leading to issues.
* **matchLabels**: Set-based requirements of the Pod selector. Often found with the **matchExpressions** statement, to further designate where the resource should be scheduled.
* **strategy**: A header for values having to do with updating Pods. Works with the later listed **type**. Could also be set to **Recreate**, which would delete all existing pods before new pods are created. With **RollingUpdate**, you can control how many Pods are deleted at a time with the following parameters.
* **maxsurge**: Maximum number of Pods over desired number of Pods to create. Can be a percentage, default of 25%, or an absolute number. This creates a certain number of new Pods before deleting old ones, for continued access.
* **maxUnavailable**: A number or percentage of Pods which can be in a state other than **Ready** during the update process.
* **type**: Even though listed last in the section, due to the level of white space indentation, it is read as the type of object being configured. (e.g. RollingUpdate).

### Deployment Configuration Pod Template

``` yaml
template:
  metadata:
  creationTimestamp: null
    labels:
      app: dev-web
  spec:
    containers:

    - image: nginx:1.13.7-alpine

      imagePullPolicy: IfNotPresent
      name: dev-web
      resources: {}
      terminationMessagePath: /dev/termination-log
      terminationMessagePolicy: File
    dnsPolicy: ClusterFirst
    restartPolicy: Always
    schedulerName: default-scheduler
    securityContext: {}
    terminationGracePeriodSeconds: 30
```

* **template**: Data being passed to the ReplicaSet to determine how to deploy an object (in this case, containers).
* **containers**: Key word indicating that the following items of this indentation are for a container.
* **image**: This is the image name passed to the container engine, typically Docker. The engine will pull the image and create the Pod.
* **imagePullPolicy**: Policy settings passed along to the container engine, about when and if an image should be downloaded or used from a local cache.
* **name**: The leading stub of the Pod names. A unique string will be appended.
* **resources**: By default, empty. This is where you would set resource restrictions and settings, such as a limit on CPU or memory for the containers.
* **terminationMessagePath**: A customizable location of where to output success or failure information of a container.
* **terminationMessagePolicy**: The default value is **File**, which holds the termination method. It could also be set to **FallbackToLogsOnError**, which will use the last chunk of container log if the message file is empty and the container shows an error.
* **dnsPolicy**: Determines if DNS queries should go to **coredns** or, if set to **Default**, use the node's DNS resolution configuration.
* **restartPolicy**: Should the container be restarted if killed? Automatic restarts are part of the typical strength of Kubernetes.
* **scheduleName**: Allows for the use of a custom scheduler, instead of the Kubernetes default.
* **securityContext**: Flexible setting to pass one or more security settings, such as SELinux context, AppArmor values, users and UIDs for the containers to use.
* **terminationGracePeriodSeconds**: The amount of time to wait for a **SIGTERM** to run until a **SIGKILL** is used to terminate the container.

### Deployment Configuration Status

The status output is generated when the information is requested:

``` yaml
status:
  availableReplicas: 2
  conditions:

  + lastTransitionTime: 2017-12-21T13:57:07Z

    lastUpdateTime: 2017-12-21T13:57:07Z
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  observedGeneration: 2
  readyReplicas: 2
  replicas: 2
  updatedReplicas: 2
```

* **availabelReplicas**: Indicates how many were configured by the ReplicaSet. This would be compared to the later value of **readyReplicas**, which would be used to determine if all replicas have been fully generated and without error.
* **observedGeneration**: Shows how often the deployment has been updated. This information can be used to understand the rollout and rollback situation of the deployment.

### Scaling and Rolling Updates

The API server allows for the configurations settings to be updated for most values. There are some immutable values, which may be different depending on the version of Kubernetes you have deployed. 

A common update is to change the number of replicas running. If this number is set to zero, there would be no containers, but there would still be a ReplicaSet and Deployment. This is the backend process when a Deployment is deleted.

``` sh
kubectl scale deploy/dev-web --replicas=4
kubectl get deployments
kubectl edit deployment nginx
```

Non-immutable values can be edited via a text editor, as well. Use edit to trigger an update. For example, to change the deployed version of the nginx web server to an older version: 

``` yaml
....
      containers:

      - image: nginx:1.8 #<<---Set to an older version 

        imagePullPolicy: IfNotPresent
                name: dev-web
....
```

This would trigger a rolling update of the deployment. While the deployment would show an older age, a review of the Pods would show a recent update and older version of the web server application deployed.

### Deployment Rollbacks

With some of the previous ReplicaSets of a Deployment being kept, you can also roll back to a previous revision by scaling up and down. The number of previous configurations kept is configurable, and has changed from version to version. Next, we will have a closer look at rollbacks, using the **--record** option of the **kubectl create** command, which allows annotation in the resource definition.

``` sh
kubectl create deploy ghost --image=ghost
kubectl get deployments ghost -o yaml
kubectl set image deployment/ghost ghost=ghost:09 --all
kubectl rollout history deployment/ghost
kubectl rollout undo deployment/ghost
kubectl get pods
kubectl rollout history deployment/ghost --revision=2
kubectl rollout pause deployment/ghost
kubectl rollout resume deployment/ghost
```

Please note that you can still do a rolling update on ReplicationControllers with the **kubectl rolling-update** command, but this is done on the client side. Hence, if you close your client, the rolling update will stop.

### Using DaemonSets

A newer object to work with is the DaemonSet. This controller ensures that a single pod exists on each node in the cluster. Every Pod uses the same image. Should a new node be added, the DaemonSet controller will deploy a new Pod on your behalf. Should a node be removed, the controller will delete the Pod also. 

The use of a DaemonSet allows for ensuring a particular container is always running. In a large and dynamic environment, it can be helpful to have a logging or metric generation application on every node without an administrator remembering to deploy that application. 

Use **kind: DaemonSet**.​

There are ways of effecting the kube-apischeduler such that some nodes will not run a DaemonSet.

### Labels

Part of the metadata of an object is a label. Though labels are not API objects, they are an important tool for cluster administration. They can be used to select an object based on an arbitrary string, regardless of the object type. Labels are immutable as of API version apps/v1.

Every resource can contain labels in its metadata. By default, creating a Deployment with kubectl create adds a label, as we saw in:

``` yaml
.... 
    labels:
        pod-template-hash: "3378155678"
        run: ghost ....
```

``` sh
kubectl get pods -l run=ghost
kubectl get pods -L run
```

While you typically define labels in pod templates and in the specifications of Deployments, you can also add labels on the fly:

``` sh
kubectl label pods ghost-3378155678-eq5i6 foo=bar
kubectl get pods --show-labels
```

For example, if you want to force the scheduling of a pod on a specific node, you can use a nodeSelector in a pod definition, add specific labels to certain nodes in your cluster and use those labels in the pod. 

``` yaml
....
spec:
    containers:

    - image: nginx

    nodeSelector:
        disktype: ssd
```

### Lab 7.1. Working with ReplicaSets

``` sh
kubectl get rs
vim rs.yaml
```

``` yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-one
spec:
  replicas: 2
  selector:
    matchLabels:
      system: ReplicaOne
  template:
    metadata:
      labels:
        system: ReplicaOne
    spec:
      containers:

      - name: nginx

        image: nginx:1.15.1
        ports:

        - containerPort: 80

```

``` sh
kubectl create -f rs.yaml
kubectl describe rs rs-one
kubectl get pods
kubectl delete rs rs-one --cascade=false
kubectl describe rs rs-one # Error
kubectl get pods
kubectl create -f rs.yaml
kubectl get rs
kubectl get pods
kubectl edit pod rs-one-h2pml # label.system to IsolatedPod
kubectl get rs
kubectl get po -L system
kubectl delete rs rs-one
kubectl get po
kubectl get rs
kubectl delete pod -l system=IsolatedPod
```

### Lab 7.2. Working with DaemonSets

``` sh
cp rs.yaml ds.yaml
vim ds.yaml
```

``` yaml
....
kind: DaemonSet
....
name: ds-one
....
replicas: 2 #<<<----Remove this line
....
system: DaemonSetOne #<<-- Edit both references
....
```

``` sh
kubectl create -f ds.yaml
kubectl get ds
kubectl get pod
kubectl describe pod ds-one-b1dcv | grep Image:
kubectl get pod -o wide
```

### Lab 7.3. Rolling Updates and Rollbacks

``` sh
kubectl get ds ds-one -o yaml | grep -A 3 Strategy
kubectl edit ds ds-one
```

``` yaml
....
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
    type: OnDelete               #<-- Edit to be this line
status:
....
```

``` sh
kubectl set image ds ds-one nginx=nginx:1.16.1-alpine
kubectl describe pod ds-one-5kd8z | grep Image:
kubectl delete po ds-one-5kd8z
kubectl get pod
kubectl describe pod ds-one-djjk9 |grep Image:
kubectl describe pod ds-one-w5bjl |grep Image: # Old one
kubectl rollout history ds ds-one
kubectl rollout history ds ds-one --revision=1
kubectl rollout history ds ds-one --revision=2
kubectl rollout undo ds ds-one --to-revision=1
kubectl describe pod ds-one-djjk9 | grep Image:
kubectl delete pod ds-one-djjk9
kubectl get pod
kubectl describe po ds-one-954xx | grep Image:
kubectl describe ds | grep Image:
kubectl get ds ds-one -o yaml
kubectl get ds ds-one -o yaml  > ds2.yaml
vim ds2.yaml # name: ds-two; type: RollingUpdate
kubectl create -f ds2.yaml
kubectl get pod
kubectl describe po ds-two-6s9tj | grep Image:
kubectl edit ds ds-two --record # - image: nginx:1.16.1-alpine
kubectl get ds ds-two
kubectl get pod
kubectl describe po ds-two-js78d | grep Image:
kubectl rollout status ds ds-two
kubectl rollout history ds ds-two
kubectl rollout history ds ds-two --revision=2
kubectl delete ds ds-one ds-two
```

## Volumes and Data

Container engines have traditionally not offered storage that outlives the container. As containers are considered transient, this could lead to a loss of data, or complex exterior storage options. A Kubernetes **volume** shares the Pod lifetime, not the containers within. Should a container terminate, the data would continue to be available to the new container. 

A volume is a directory, possibly pre-populated, made available to containers in a Pod. The creation of the directory, the backend storage of the data and the contents depend on the volume type. As of v1.13, there were 27 different volume types ranging from rbd to gain access to Ceph, to NFS, to dynamic volumes from a cloud provider like Google's gcePersistentDisk. Each has particular configuration options and dependencies. 

The Container Storage Interface (CSI) adoption enables the goal of an industry standard interface for container orchestration to allow access to arbitrary storage systems. Currently, volume plugins are "in-tree", meaning they are compiled and built with the core Kubernetes binaries. This "out-of-tree" object will allow storage vendors to develop a single driver and allow the plugin to be containerized. This will replace the existing Flex plugin which requires elevated access to the host node, a large security concern. 

Should you want your storage lifetime to be distinct from a Pod, you can use Persistent Volumes. These allow for empty or pre-populated volumes to be claimed by a Pod using a Persistent Volume Claim, then outlive the Pod. Data inside the volume could then be used by another Pod, or as a means of retrieving data. 

There are two API objects which exist to provide data to a Pod already. Encoded data can be passed using a Secret and non-encoded data can be passed with a ConfigMap. These can be used to pass important data like SSH keys, passwords, or even a configuration file like */etc/hosts*.

### Introducing Volumes

A Pod specification can declare one or more volumes and where they are made available. Each requires a name, a type, and a mount point. The same volume can be made available to multiple containers within a Pod, which can be a method of container-to-container communication. A volume can be made available to multiple Pods, with each given an access mode to write. There is no concurrency checking, which means data corruption is probable, unless outside locking takes place. 

![Pod Volumes](../../images/pod-volumes.png "Pod Volumes")

A particular access mode is part of a Pod request. As a request, the user may be granted more, but not less access, though a direct match is attempted first. The cluster groups volumes with the same mode together, then sorts volumes by size, from smallest to largest. The claim is checked against each in that access mode group, until a volume of sufficient size matches. The three access modes are:

* **ReadWriteOnce**: which allows read-write by a single node
* **ReadOnlyMany**: which allows read-only by multiple nodes
* **ReadWriteMany**: which allows read-write by many nodes

Thus two pods on the same node can write to a ReadWriteOnce, but a third pod on a different node would not become ready due to a FailedAttachVolume error.

When a volume is requested, the local kubelet uses the **kubelet_pods.go** script to map the raw devices, determine and make the mount point for the container, then create the symbolic link on the host node filesystem to associate the storage to the container. The API server makes a request for the storage to the **StorageClass** plugin, but the specifics of the requests to the backend storage depend on the plugin in use. 

If a request for a particular **StorageClass** was not made, then the only parameters used will be access mode and size. The volume could come from any of the storage types available, and there is no configuration to determine which of the available ones will be used.

### Volume Spec

One of the many types of storage available is an **emptyDir**. The kubelet will create the directory in the container, but not mount any storage. Any data created is written to the shared container space. As a result, it would not be persistent storage. When the Pod is destroyed, the directory would be deleted along with the container.

``` yaml
apiVersion: v1
kind: Pod
metadata:
    name: fordpinto 
    namespace: default
spec:
    containers:

    - image: simpleapp 

      name: gastank 
      command:

        - sleep
        - "3600"

      volumeMounts:

      - mountPath: /scratch

        name: scratch-volume
    volumes:

    - name: scratch-volume

            emptyDir: {}
```

The YAML file above would create a Pod with a single container with a volume named **scratch-volume** created, which would create the */scratch* directory inside the container.

### Volume Types

There are several types that you can use to define volumes, each with their pros and cons. Some are local, and many make use of network-based resources.

In GCE or AWS, you can use volumes of type **GCEpersistentDisk** or **awsElasticBlockStore**, which allows you to mount GCE and EBS disks in your Pods, assuming you have already set up accounts and privileges.

**emptyDir** and **hostPath** volumes are easy to use. As mentioned, **emptyDir** is an empty directory that gets erased when the Pod dies, but is recreated when the container restarts. The **hostPath** volume mounts a resource from the host node filesystem. The resource could be a directory, file socket, character, or block device. These resources must already exist on the host to be used. There are two types, **DirectoryOrCreate** and **FileOrCreate**, which create the resources on the host, and use them if they don't already exist.

NFS (Network File System) and iSCSI (Internet Small Computer System Interface) are straightforward choices for multiple readers scenarios.

rbd for block storage or CephFS and GlusterFS, if available in your Kubernetes cluster, can be a good choice for multiple writer needs.

Besides the volume types we just mentioned, there are many other possible, with more being added: **azureDisk**, **azureFile**, **csi**, **downwardAPI**, **fc** (fibre channel), **flocker**, **gitRepo**, **local**, **projected**, **portworxVolume**, **quobyte**, **scaleIO**, **secret**, **storageos**, **vsphereVolume**, **persistentVolumeClaim**, **CSIPersistentVolumeSource**, etc.​

CSI allows for even more flexibility and decoupling plugins without the need to edit the core Kubernetes code. It was developed as a standard for exposing arbitrary plugins in the future. 

### Shared Volume Example

The following YAML file creates a pod, **exampleA**, with two containers, both with access to a shared volume:

``` yaml
....
   containers:
   - name: alphacont
     image: busybox
     volumeMounts:
     - mountPath: /alphadir
       name: sharevol
   - name: betacont
     image: busybox
     volumeMounts:
     - mountPath: /betadir
       name: sharevol
   volumes:
   - name: sharevol
     emptyDir: {}   
```

``` sh
kubectl exec -ti exampleA -c betacont -- touch /betadir/foobar
kubectl exec -ti exampleA -c alphacont -- ls -l /alphadir
```

You could use **emptyDir** or **hostPath** easily, since those types do not require any additional setup, and will work in your Kubernetes cluster. 

Note that one container (**betacont**) wrote, and the other container (**alphacont**) had immediate access to the data. There is nothing to keep the containers from overwriting the other's data. Locking or versioning considerations must be part of the containerized application to avoid corruption.

### Persistent Volumes and Claims

A **persistent volume** (pv) is a storage abstraction used to retain data longer then the Pod using it. Pods define a volume of type **persistentVolumeClaim** (**pvc**) with various parameters for size and possibly the type of backend storage known as its **StorageClass**. The cluster then attaches the persistentVolume.

Kubernetes will dynamically use volumes that are available, irrespective of its storage type, allowing claims to any backend storage.

#### Persistent Storage Phases

* Provision: Provisioning can be from PVs created in advance by the cluster administrator, or requested from a dynamic source, such as the cloud provider.
* Bind: Binding occurs when a control loop on the master notices the PVC, containing an amount of storage, access request, and optionally, a particular **StorageClass**. The watcher locates a matching PV or waits for the **StorageClass** provisioner to create one. The PV must match at least the storage amount requested, but may provide more.
* Use: The use phase begins when the bound volume is mounted for the Pod to use, which continues as long as the Pod requires.
* Release: Releasing happens when the Pod is done with the volume and an API request is sent, deleting the PVC. The volume remains in the state from when the claim is deleted until available to a new claim. The resident data remains depending on the **persistentVolumeReclaimPolicy**.
* Reclaim: The reclaim phase has three options:
  + Retain, which keeps the data intact, allowing for an administrator to handle the storage and data.
  + Delete tells the volume plugin to delete the API object, as well as the storage behind it. 
  + The Recycle option runs an rm -rf /mountpoint and then makes it available to a new claim. With the stability of dynamic provisioning, the Recycle option is planned to be deprecated.

``` sh
kubectl get pv
kubectl get pvc
```

### Persistent Volume

The following example shows a basic declaration of a Persistent Volume using the **hostPath** type.

``` yaml
kind: PersistentVolume
apiVersion: v1
metadata:
    name: 10Gpv01
    labels: 
        type: local 
spec:
    capacity: 
        storage: 10Gi
    accessModes:

        - ReadWriteOnce

    hostPath:
        path: "/somepath/data01"
```

Each type will have its own configuration settings. For example, an already created Ceph or GCE Persistent Disk would not need to be configured, but could be claimed from the provider.

Persistent volumes are not a namespaces object, but persistent volume claims are. A beta feature of v1.13 allows for static provisioning of Raw Block Volumes, which currently support the Fibre Channel plugin, AWS EBS, Azure Disk and RBD plugins among others.

The use of locally attached storage has been graduated to a stable feature. This feature is often used as part of distributed filesystems and databases.

### Persistence Volume Claim

With a persistent volume created in your cluster, you can then write a manifest for a claim, and use that claim in your pod definition. In the Pod, the volume uses the **persistentVolumeClaim**.

``` yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
    name: myclaim
spec:
    accessModes:

        - ReadWriteOnce

    resources:
        requests:
                storage: 8GI
```

In the pod:

``` yaml
spec:
    containers:
....
    volumes:

        - name: test-volume

          persistentVolumeClaim:
                claimName: myclaim
```

The Pod configuration could also be as complex as this:

``` yaml
volumeMounts:

      - name: Cephpd

        mountPath: /data/rbd
  volumes:

    - name: rbdpd

      rbd:
        monitors:

        - '10.19.14.22:6789'
        - '10.19.14.23:6789'
        - '10.19.14.24:6789'

        pool: k8s
        image: client
        fsType: ext4
        readOnly: true
        user: admin
        keyring: /etc/ceph/keyring
        imageformat: "2"
        imagefeatures: "layering"
```

### Dynamic Provisioning

While handling volumes with a persistent volume definition and abstracting the storage provider using a claim is powerful, a cluster administrator still needs to create those volumes in the first place. Starting with Kubernetes v1.4, Dynamic Provisioning allowed for the cluster to request storage from an exterior, pre-configured source. API calls made by the appropriate plugin allow for a wide range of dynamic storage use. 

The **StorageClass** API resource allows an administrator to define a persistent volume provisioner of a certain type, passing storage-specific parameters. 

With a **StorageClass** created, a user can request a claim, which the API Server fills via auto-provisioning. The resource will also be reclaimed as configured by the provider. AWS and GCE are common choices for dynamic storage, but other options exist, such as a Ceph cluster or iSCSI. Single, default class is possible via annotation.

Here is an example of a **StorageClass** using GCE: 

``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast      # Could be any name
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```

### Using Rook for Storage Orchestration

In keeping with the decoupled and distributed nature of the Cloud technology, the [Rook](https://rook.io/) project allows orchestration of storage using multiple storage providers.

As with other agents of the cluster, Rook uses custom resource definitions (CRD) and a custom operator to provision storage according to the backend storage type, upon API call.

Several storage providers are supported:

* Ceph
* Cassandra
* CockroachDB
* EdgeFS Geo-Transparant Storage
* Minio Object Store
* Network File System (NFS)
* YugabyteDB.

### Secrets

Pods can access local data using volumes, but there is some data you don't want readable to the naked eye. Passwords may be an example. Using the Secret API resource, the same password could be encoded or encrypted.

You can create, get, or delete secrets. Secrets can be encoded manually or via **kubectl create secret**:​

``` sh
kubectl get secrets
kubectl create secret generic --help
kubectl create secret generic mysql --from-literal=password=root
```

A secret is not encrypted, only base64-encoded, by default. You must create an **EncryptionConfiguration** with a key and proper identity. Then, the kube-apiserver needs the **--encryption-provider-config** flag set to a previously configured provider, such as **aescbc** or **ksm**. Once this is enabled, you need to recreate every secret, as they are encrypted upon write.

Multiple keys are possible. Each key for a provider is tried during decryption. The first key of the first provider is used for encryption. To rotate keys, first create a new key, restart (all) kube-apiserver processes, then recreate every secret. 

You can see the encoded string inside the secret with **kubectl**. The secret will be decoded and be presented as a string saved to a file. The file can be used as an environmental variable or in a new directory, similar to the presentation of a volume.

``` sh
$ echo LFTr@1n | base64
TEZUckAxbgo=

$ vim secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: lf-secret
data:
  password: TEZUckAxbgo=
```

### Using Secrets via Environment Variables

A secret can be used as an environmental variable in a Pod. You can see one being configured in the following example: 

``` yaml
...
spec:
    containers:

    - image: mysql:5.5

      env:

      - name: MYSQL_ROOT_PASSWORD

        valueFrom:
            secretKeyRef:
                name: mysql
                key: password
          name: mysql
```

There is no limit to the number of Secrets used, but there is a 1MB limit to their size. Each secret occupies memory, along with other API objects, so very large numbers of secrets could deplete memory on a host.

They are stored in the **tmpfs** storage on the host node, and are only sent to the host running Pod. All volumes requested by a Pod must be mounted before the containers within the Pod are started. So, a secret must exist prior to being requested.

### Mounting Secrets as Volumes

You can also mount secrets as files using a volume definition in a pod manifest. The mount path will contain a file whose name will be the key of the secret created with the **kubectl create secret** step earlier.

``` yaml
...
spec:
    containers:

    - image: busybox

      command:

        - sleep
        - "3600"

      volumeMounts:

      - mountPath: /mysqlpassword

        name: mysql
      name: busy
    volumes:

    - name: mysql

        secret:
          secretName: mysql
```

Once the pod is running, you can verify that the secret is indeed accessible in the container:

``` sh
kubectl exec -ti busybox -- cat /mysqlpassword/password
```

### Portable Data with ConfigMaps

A similar API resource to Secrets is the ConfigMap, except the data is not encoded. In keeping with the concept of decoupling in Kubernetes, using a ConfigMap decouples a container image from configuration artifacts. 

They store data as sets of key-value pairs or plain configuration files in any format. The data can come from a collection of files or all files in a directory. It can also be populated from a literal value. 

A ConfigMap can be used in several different ways. A container can use the data as environmental variables from one or more sources. The values contained inside can be passed to commands inside the pod. A Volume or a file in a Volume can be created, including different names and particular access modes. In addition, cluster components like controllers can use the data. ​

Let's say you have a file on your local filesystem called **config.js**. You can create a ConfigMap that contains this file. The **configmap** object will have a data section containing the content of the file:

``` sh
kubectl get configmap foobar -o yaml
```

``` yaml
kind: ConfigMap
apiVersion: v1
metadata:
    name: foobar
data:
    config.js: |
         {
...
```

ConfigMaps can be consumed in various ways:

* Pod environmental variables from single or multiple ConfigMaps
* Use ConfigMap values in Pod commands
* Populate Volume from ConfigMap
* Add ConfigMap data to specific path in Volume
* Set file names and access mode in Volume from ConfigMap data
* Can be used by system components and controllers.

### Using ConfigMaps

Like secrets, you can use ConfigMaps as environment variables or using a volume mount. They must exist prior to being used by a Pod, unless marked as optional. They also reside in a specific namespace.

In the case of environment variables, your pod manifest will use the **valueFrom** key and the **configMapKeyRef** value to read the values. For instance:

``` yaml
env:

* name: SPECIAL_LEVEL_KEY

  valueFrom:
    configMapKeyRef:
      name: special-config
      key: special.how
```

With volumes, you define a volume with the configMap type in your pod and mount it where it needs to be used.

``` yaml
volumes:

    - name: config-volume

      configMap:
        name: special-config
```

### Lab 8.1. Create a ConfigMap

``` sh
mkdir primary
echo c > primary/cyan
echo m > primary/magenta
echo y > primary/yellow
echo k > primary/black
echo "known as key" >> primary/black
echo blue > favorite
kubectl create configmap colors --from-literal=text=black  --from-file=./favorite  --from-file=./primary/
kubectl get configmap colors
kubectl get configmap colors -o yaml
vim simpleshell.yaml
```

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: shell-demo
spec:
  containers:

  + name: nginx

    image: nginx
    env:

    - name: ilike

      valueFrom:
        configMapKeyRef:
          name: colors
          key: favorite
```

``` sh
kubectl create -f simpleshell.yaml
kubectl exec shell-demo -- /bin/bash -c 'echo $ilike'
kubectl delete pod shell-demo
vim simpleshell.yaml
```

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: shell-demo
spec:
  containers:

  + name: nginx

    image: nginx
    # env:
    # - name: ilike
    #   valueFrom:
    #     configMapKeyRef:
    #       name: colors
    #       key: favorite
    envFrom:                  

    - configMapRef:

        name: colors
```

``` sh
kubectl create -f simpleshell.yaml
kubectl exec shell-demo -- /bin/bash -c 'env'
kubectl delete pod shell-demo
vim car-map.yaml
```

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fast-car
  namespace: default
data:
  car.make: Ford
  car.model: Mustang
  car.trim: Shelby
```

``` sh
kubectl create -f car-map.yaml
kubectl get configmap fast-car -o yaml
vim simpleshell.yaml
```

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: shell-demo
spec:
  containers:

  + name: nginx

    image: nginx
    # env:
    # - name: ilike
    #   valueFrom:
    #     configMapKeyRef:
    #       name: colors
    #       key: favorite
    # envFrom:                  
    # - configMapRef:
    #     name: colors
    volumeMounts:

    - name: car-vol

      mountPath: /etc/cars
  volumes:

  + name: car-vol

    configMap:
      name: fast-car
```

``` sh
kubectl create -f simpleshell.yaml
kubectl exec shell-demo -- /bin/bash -c 'df -ha |grep car'
kubectl exec shell-demo -- /bin/bash -c 'cat /etc/cars/car.trim'
kubectl delete pods shell-demo
kubectl delete configmap fast-car colors
```

### Lab 8.2. Create a Persistent NFS Volume (PV)

From **master** node:

``` sh
sudo apt-get update && sudo apt-get install -y nfs-kernel-server
sudo mkdir /opt/sfw
sudo chmod 1777 /opt/sfw/
sudo bash -c 'echo software > /opt/sfw/hello.txt'
sudo vim /etc/exports # /opt/sfw/ *(rw,sync,no_root_squash,subtree_check)
sudo exportfs -ra
```

From **worker** node:

``` sh
sudo apt-get -y install nfs-common
showmount -e k8smaster
sudo mount k8smaster:/opt/sfw /mnt
ls -l /mnt
```

From **master** node:

``` sh
vim PVol.yaml
```

``` yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pvvol-1
spec:
  capacity:
    storage: 1Gi
  accessModes:

  + ReadWriteMany

  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /opt/sfw
    server: k8smaster   #<-- Edit to match master node
    readOnly: false
```

``` sh
kubectl create -f PVol.yaml
kubectl get pv
```

### Lab 8.3. Creating a Persistent Volume Claim (PVC)

``` sh
kubectl get pvc
vim pvc.yaml
```

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-one
spec:
  accessModes:

  + ReadWriteMany

  resources:
    requests:
      storage: 200Mi
```

``` sh
kubectl create -f pvc.yaml
kubectl get pvc
kubectl get pv
cp first.yaml nfs-pod.yaml
vim nfs-pod.yaml
```

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  generation: 1
  labels:
    run: nginx
  name: nginx-nfs
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: nginx
    spec:
      containers:

      - image: nginx

        imagePullPolicy: Always
        name: nginx
        volumeMounts:

        - name: nfs-vol

          mountPath: /opt
        ports:

        - containerPort: 80

          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      volumes:

      - name: nfs-vol

        persistentVolumeClaim:
          claimName: pvc-one
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

``` sh
kubectl create -f nfs-pod.yaml
kubectl get pods
kubectl describe pod nginx-nfs-5f58fd64fd-2rrfq
kubectl get pvc
```

### Lab 8.4. Use a ResourceQuota to Limit PVC Count and Usage

``` sh
kubectl delete deploy nginx-nfs
kubectl delete pvc pvc-one
kubectl delete pv pvvol-1
vim storage-quota.yaml
```

``` yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storagequota
spec:
  hard:
    persistentvolumeclaims: "10"
    requests.storage: "500Mi"
```

``` sh
kubectl create namespace small
kubectl describe ns small
kubectl -n small create -f PVol.yaml
kubectl -n small create -f pvc.yaml
kubectl -n small create -f storage-quota.yaml
kubectl describe ns small
vim nfs-pod.yaml # Removing namespace line
kubectl -n small create -f nfs-pod.yaml
kubectl -n small get deploy
kubectl -n small describe deploy nginx-nfs
kubectl -n small get pod
kubectl -n small describe pod nginx-nfs-5f58fd64fd-bwm8c
kubectl describe ns small
sudo dd if=/dev/zero of=/opt/sfw/bigfile bs=1M count=300
kubectl describe ns small
sudo du -h /opt/
kubectl -n small get deploy
kubectl -n small delete deploy nginx-nfs
kubectl describe ns small
kubectl -n small get pvc
kubectl -n small delete pvc pvc-one
kubectl -n small get pv
kubectl get pv/pvvol-1 -o yaml
kubectl delete pv/pvvol-1
grep Retain PVol.yaml
kubectl create -f PVol.yaml
kubectl patch pv pvvol-1 -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'
kubectl get pv/pvvol-1
kubectl describe ns small
kubectl -n small create -f pvc.yaml
kubectl describe ns small
kubectl -n small get resourcequota
kubectl -n small delete resourcequota storagequota
vim storage-quota.yaml # 100Mi instead of 500Mi
kubectl -n small create -f storage-quota.yaml
kubectl describe ns small
kubectl -n small create -f nfs-pod.yaml
kubectl -n small describe deploy/nginx-nfs
kubectl -n small get po
kubectl -n small delete deploy nginx-nfs
kubectl -n small delete pvc/pvc-one
kubectl -n small get pv
kubectl delete pv/pvvol-1
vim PVol.yaml # persistentVolumeReclaimPolicy: Recycle
kubectl -n small create -f low-resource-range.yaml
kubectl describe ns small
kubectl -n small create -f PVol.yaml
kubectl get pv
kubectl -n small create -f pvc.yaml # Error
kubectl -n small edit resourcequota # requests.storageto 500Mi
kubectl -n small create -f pvc.yaml
kubectl -n small create -f nfs-pod.yaml
kubectl describe ns small
kubectl -n small delete deploy nginx-nfs
kubectl -n small get pvc
kubectl -n small get pv
kubectl -n small delete pvc pvc-one
kubectl -n small get pv
kubectl delete pv pvvol-1
```

## Services

As touched on previously, the Kubernetes architecture is built on the concept of transient, decoupled objects connected together. Services are the agents which connect Pods together, or provide access outside of the cluster, with the idea that any particular Pod could be terminated and rebuilt. Typically using Labels, the refreshed Pod is connected and the microservice continues to provide the expected resource via an **Endpoint** object. Google has been working on Extensible Service Proxy (ESP), based off the nginx HTTP reverse proxy server, to provide a more flexible and powerful object than Endpoints, but ESP has not been adopted much outside of the Google App Engine or GKE environments. 

There are several different service types, with the flexibility to add more, as necessary. Each service can be exposed internally or externally to the cluster. A service can also connect internal resources to an external resource, such as a third-party database. 

The kube-proxy agent watches the Kubernetes API for new services and endpoints being created on each node. It opens random ports and listens for traffic to the **ClusterIP: Port**, and redirects the traffic to the randomly generated service endpoints.

Services provide automatic load-balancing, matching a label query. While there is no configuration of this option, there is the possibility of session affinity via IP. Also, a headless service, one without a fixed IP nor load-balancing, can be configured. 

Unique IP addresses are assigned and configured via the etcd database, so that Services implement iptables to route traffic, but could leverage other technologies to provide access to resources in the future.

### Service Update Pattern

Labels are used to determine which Pods should receive traffic from a service. As we have learned, labels can be dynamically updated for an object, which may affect which Pods continue to connect to a service. 

The default update pattern is for a rolling deployment, where new Pods are added, with different versions of an application, and due to automatic load balancing, receive traffic along with previous versions of the application. 

Should there be a difference in applications deployed, such that clients would have issues communicating with different versions, you may consider a more specific label for the deployment, which includes a version number. When the deployment creates a new replication controller for the update, the label would not match. Once the new Pods have been created, and perhaps allowed to fully initialize, we would edit the labels for which the Service connects. Traffic would shift to the new and ready version, minimizing client version confusion.

### Accessing an Application with a Service

The basic step to access a new service is to use **kubectl**. 

``` sh
kubectl expose deployment/nginx --port=80 --type=NodePort
kubectl get svc
kubectl get svc nginx -o yaml
```

Open browser **http://Public-IP:31230**.

The **kubectl expose** command created a service for the **nginx** deployment. This service used port 80 and generated a random port on all the nodes. A particular **port** and **targetPort** can also be passed during object creation to avoid random values. The **targetPort** defaults to the port, but could be set to any value, including a string referring to a port on a backend Pod. Each Pod could have a different port, but traffic is still passed via the name. Switching traffic to a different port would maintain a client connection, while changing versions of software, for example.

The **kubectl get svc** command gave you a list of all the existing services, and we saw the **nginx** service, which was created with an internal cluster IP.

The range of cluster IPs and the range of ports used for the random NodePort are configurable in the API server startup options.

Services can also be used to point to a service in a different namespace, or even a resource outside the cluster, such as a legacy application not yet in Kubernetes.

### Service Types

* **ClusterIP**: The ClusterIP service type is the default, and only provides access internally (except if manually creating an external endpoint). The range of ClusterIP used is defined via an API server startup option.
* **NodePort**: The NodePort type is great for debugging, or when a static IP address is necessary, such as opening a particular address through a firewall. The NodePort range is defined in the cluster configuration.
* **LoadBalancer**: The LoadBalancer service was created to pass requests to a cloud provider like GKE or AWS. Private cloud solutions also may implement this service type if there is a cloud provider plugin, such as with CloudStack and OpenStack. Even without a cloud provider, the address is made available to public traffic, and packets are spread among the Pods in the deployment automatically.
* **ExternalName**: A newer service is ExternalName, which is a bit different. It has no selectors, nor does it define ports or endpoints. It allows the return of an alias to an external service. The redirection happens at the DNS level, not via a proxy or forward. This object can be useful for services not yet brought into the Kubernetes cluster. A simple change of the type in the future would redirect traffic to the internal objects.

The **kubectl proxy** command creates a local service to access a ClusterIP. This can be useful for troubleshooting or development work.

### Services Diagram

![Traffic from ClusterIP to Pod](../../images/traffic-from-clusterip-to-pod.png "Traffic from ClusterIP to Pod")

The kube-proxy running on cluster nodes watches the API server service resources. It presents a type of virtual IP address for services other than **ExternalName**. The mode for this process has changed over versions of Kubernetes.

In v1.0, services ran in userspace mode as TCP/UDP over IP or Layer 4. In the v1.1 release, the iptables proxy was added and became the default mode starting with v1.2.

In the iptables proxy mode, kube-proxy continues to monitor the API server for changes in Service and Endpoint objects, and updates rules for each object when created or removed. One limitation to the new mode is an inability to connect to a Pod should the original request fail, so it uses a Readiness Probe to ensure all containers are functional prior to connection. This mode allows for up to approximately 5000 nodes. Assuming multiple Services and Pods per node, this leads to a bottleneck in the kernel.

Another mode beginning in v1.9 is ipvs. While in beta, and expected to change, it works in the kernel space for greater speed, and allows for a configurable load-balancing algorithm, such as round-robin, shortest expected delay, least connection and several others. This can be helpful for large clusters, much past the previous 5000 node limitation. This mode assumes IPVS kernel modules are installed and running prior to kube-proxy.

The kube-proxy mode is configured via a flag sent during initialization, such as **mode=iptables** and could also be IPVS or userspace.

### Local Proxy for Development

When developing an application or service, one quick way to check your service is to run a local proxy with kubectl. It will capture the shell, unless you place it in the background. When running, you can make calls to the Kubernetes API on localhost and also reach the ClusterIP services on their API URL. The IP and port where the proxy listens can be configured with command arguments. 

Run a proxy:

``` sh
kubectl proxy
```

Next, to access a **ghost** service using the local proxy, we could use the following URL, for example, at **http://localhost:8001/api/v1/namespaces/default/services/ghost**.

If the service port has a name, the path will be **http://localhost:8001/api/v1/namespaces/default/services/ghost:<port_name>**.

### DNS

DNS has been provided as CoreDNS by default as of v1.13. The use of CoreDNS allows for a great amount of flexibility. Once the container starts, it will run a Server for the zones it has been configured to serve. Then, each server can load one or more plugin chains to provide other functionality. As with other microservices, clients would it access using a service, **kube-dns**.

The thirty or so in-tree plugins provide most common functionality, with an easy process to write and enable other plugins as necessary.

Common plugins can provide metrics for consumption by Prometheus, error logging, health reporting, and TLS to configure certificates for TLS and gRPC servers.

More can be found on the [CoreDNS Plugins web page](https://coredns.io/plugins/).

### Verifying DNS Registration

To make sure that your DNS setup works well and that services get registered, the easiest way to do it is to run a pod with a shell and network tools in the cluster, create a service to connect to the pod, then exec in it to do a DNS lookup.

Troubleshooting of DNS uses typical tools such as **nslookup**, **dig**, **nc**, **wireshark** and more. The difference is that we leverage a service to access the DNS server, so we need to check labels and selectors in addition to standard network concerns.

Other steps, similar to any DNS troubleshooting, would be to check the */etc/resolv.conf* file of the container, as well as Network Policies and firewalls. We will cover more on Network Policies in the Security chapter.

### Lab 9.1. Deploy a New Service

``` sh
vim nginx-one.yaml
```

``` yaml
apiVersion: apps/v1
# Determines YAML versioned schema.
kind: Deployment
# Describes the resource defined in this file.
metadata:
  name: nginx-one
  labels:
    system: secondary
# Required string which defines object within namespace.
  namespace: accounting
# Existing namespace resource will be deployed into.
spec:
  selector:
    matchLabels:
      system: secondary
# Declaration of the label for the deployment to manage
  replicas: 2
# How many Pods of following containers to deploy
  template:
    metadata:
      labels:
        system: secondary
# Some string meaningful to users, not cluster. Keys
# must be unique for each object. Allows for mapping
# to customer needs.
    spec:
      containers:
# Array of objects describing containerized application with a Pod.
# Referenced with shorthand spec.template.spec.containers

      - image: nginx:1.16.1

# The Docker image to deploy
        imagePullPolicy: Always
        name: nginx
# Unique name for each container, use local or Docker repo image
        ports:

        - containerPort: 8080

          protocol: TCP
# Optional resources this container may need to function.
      nodeSelector:
        system: secondOne
# One method of node affinity. 
```

``` sh
kubectl get nodes --show-labels
kubectl create -f nginx-one.yaml # Error
kubectl create ns accounting
kubectl create -f nginx-one.yaml
kubectl -n accounting get pods
kubectl -n accounting describe pod nginx-one-fb4bdb45d-9z67r
kubectl label node worker system=secondOne
kubectl get nodes --show-labels
kubectl get pods -l system=secondary --all-namespaces
kubectl -n accounting expose deployment nginx-one
kubectl -n accounting get ep nginx-one
curl 192.168.171.68:8080 # Error
curl 192.168.171.68:80
kubectl -n accounting delete deploy nginx-one
vim nginx-one.yaml # containerPort: 80
kubectl create -f nginx-one.yaml
```

### Lab 9.2. Configure a NodePort

``` sh
kubectl -n accounting expose deployment nginx-one --type=NodePort --name=service-lab
kubectl -n accounting describe services
kubectl cluster-info
curl http://k8smaster:30725
curl ifconfig.io # In a browser: 35.188.124.221:30725
```

### Lab 9.3. Working with CoreDNS

``` sh
vim nettool.yaml
```

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu
spec:
  containers:

  + name: ubuntu

    image: ubuntu:latest
    command: [ "sleep" ]
    args: [ "infinity" ]
```

``` sh
kubectl create -f nettool.yaml
kubectl exec -it ubuntu -- /bin/bash
apt-get update ; apt-get install curl dnsutils -y # 8 and 29 for Europe/Madrid
dig
cat /etc/resolv.conf
dig @10.96.0.10 -x 10.96.0.10
curl service-lab.accounting.svc.cluster.local.
curl service-lab # Error
curl service-lab.accounting
exit
kubectl -n kube-system get svc
kubectl -n kube-system get svc kube-dns -o yaml
kubectl get pod -l k8s-app --all-namespaces
kubectl -n kube-system get pod coredns-74ff55c5b-8s2lg -o yaml
kubectl -n kube-system get configmaps
kubectl -n kube-system get configmaps coredns -o yaml
kubectl -n kube-system edit configmaps coredns # rewrite name regex (.*)\.test\.io {1}.default.svc.cluster.local --> before errors line
kubectl -n kube-system delete pod coredns-74ff55c5b-8s2lg coredns-74ff55c5b-npmcv
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --type=ClusterIP --port=80
kubectl get svc
kubectl exec -it ubuntu -- /bin/bash
dig -x 10.98.8.86
dig nginx.default.svc.cluster.local.
dig nginx.test.io
exit
kubectl -n kube-system edit configmaps coredns # Replace the rewrite line for those:
# rewrite stop {
#            name regex (.*)\.test\.io {1}.default.svc.cluster.local
#            answer name (.*)\.default\.svc\.cluster\.local {1}.test.io
#         }
kubectl -n kube-system delete pod coredns-74ff55c5b-68892 coredns-74ff55c5b-q6h5h
kubectl exec -it ubuntu -- /bin/bash
dig nginx.test.io
exit
kubectl delete -f nettool.yaml
```

### Lab 9.4. Use Labels to Manage Resources

``` sh
kubectl delete pods -l system=secondary --all-namespaces
kubectl -n accounting get pods
kubectl -n accounting get deploy --show-labels
kubectl -n accounting delete deploy -l system=secondary
kubectl label node worker system-
```

## Ingress

In an earlier chapter, we learned about using a Service to expose a containerized application outside of the cluster. We use Ingress Controllers and Rules to do the same function. The difference is efficiency. Instead of using lots of services, such as LoadBalancer, you can route traffic based on the request host or path. This allows for centralization of many services to a single point. 

An Ingress Controller is different than most controllers, as it does not run as part of the kube-controller-manager binary. You can deploy multiple controllers, each with unique configurations. A controller uses Ingress Rules to handle traffic to and from outside the cluster. 

There are many ingress controllers such as GKE, nginx, Traefik, Contour and Envoy to name a few. Any tool capable of reverse proxy should work. These agents consume rules and listen for associated traffic. An Ingress Rule is an API resource that you can create with **kubectl**. When you create that resource, it reprograms and reconfigures your Ingress Controller to allow traffic to flow from the outside to an internal service. You can leave a service as a ClusterIP type and define how the traffic gets routed to that internal service using an Ingress Rule.

### Ingress Controller

An Ingress Controller is a daemon running in a Pod which watches the **/ingresses** endpoint on the API server, which is found under the **networking.k8s.io/v1beta1** group for new objects. When a new endpoint is created, the daemon uses the configured set of rules to allow inbound connection to a service, most often HTTP traffic. This allows easy access to a service through an edge router to Pods, regardless of where the Pod is deployed. 

Multiple Ingress Controllers can be deployed. Traffic should use annotations to select the proper controller. The lack of a matching annotation will cause every controller to attempt to satisfy the ingress traffic.

![Ingress Controller](../../images/ingress-controller.png "Ingress Controller")

### nginx

Deploying an nginx controller has been made easy through the use of provided YAML files, which can be found in the [ingress-nginx/deploy GitHub repository](https://github.com/kubernetes/ingress-nginx/tree/master/deploy).

This page has configuration files to configure nginx on several platforms, such as AWS, GKE, Azure, and bare metal, among others.

As with any Ingress Controller, there are some configuration requirements for proper deployment. Customization can be done via a ConfigMap, Annotations, or, for detailed configuration, a custom template:

* Easy integration with RBAC
* Uses the annotation **kubernetes.io/ingress.class: "nginx"**
* L7 traffic requires the **proxy-real-ip-cidr** setting
* Bypasses kube-proxy to allow session affinity
* Does not use **conntrack** entries for iptables DNAT
* TLS requires the host field to be defined.

### Google Load Balancer Controller (GLBC)

There are several objects which need to be created to deploy the GCE Ingress Controller. YAML files are available to make the process easy. Be aware that several objects would be created for each service, and currently, quotas are not evaluated prior to creation. 

The GLBC Controller must be created and started first. Also, you must create a ReplicationController with a single replica, three services for the application Pod, and an Ingress with two hostnames and three endpoints for each service. The backend is a group of virtual machine instances, Instance Group. ​

Each path for traffic uses a group of like objects referred to as a pool. Each pool regularly checks the next hop up to ensure connectivity. 

The multi-pool path is:

**Global Forwarding Rule -> Target HTTP Proxy -> URL map -> Backend Service -> Instance Group**

Currently, the TLS Ingress only supports port 443 and assumes TLS termination. It does not support SNI, only using the first certificate. The TLS secret must contain keys named **tls.crt** and **tls.key**.

### Ingress API Resources

Ingress objects are now part of the **networking.k8s.io** API, but still a beta object. A typical Ingress object that you can POST to the API server is:

``` yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ghost
spec:
  rules:

    - host: ghost.192.168.99.100.nip.io

http:
paths:

    - backend:

            serviceName: ghost
            servicePort: 2368
```

You can manage ingress resources like you do pods, deployments, services etc:

``` sh
kubectl get ingress
kubectl delete ingress <ingress_name>
kubectl edit ingress <ingress_name>
```

### Deploying the Ingress Controller

To deploy an Ingress Controller, it can be as simple as creating it with **kubectl**. The source for a sample controller deployment is available on [GitHub](https://github.com/kubernetes/ingress-nginx/tree/master/deploy).

The result will be a set of pods managed by a replication controller and some internal services. You will notice a default HTTP backend which serves 404 pages.

``` sh
kubectl create -f backend.yaml
kubectl get pods,rc,svc
```

### Creating an Ingress Rule

To get exposed with ingress quickly, you can go ahead and try to create a similar rule as mentioned on the previous page. First, start a **ghost** deployment and expose it with an internal ClusterIP service. With the deployment exposed and the Ingress rules in place, you should be able to access the application from outside the cluster:

``` sh
kubectl run ghost --image=ghost
kubectl expose deployments ghost --port=2368
```

``` yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
    name: ghost
spec:
    rules:

    - host: ghost.192.168.99.100.nip.io

      http:
      paths:

      - backend:

            serviceName: ghost
            servicePort: 2368
```

### Multiple Rules

On the previous page, we defined a single rule. If you have multiple services, you can define multiple rules in the same Ingress, each rule forwarding traffic to a specific service. 

``` yaml
rules:

* host: ghost.192.168.99.100.nip.io

  http:
    paths:

    - backend:

        serviceName: ghost
        servicePort: 2368

* host: nginx.192.168.99.100.nip.io

  http:
    paths:

    - backend:

        serviceName: nginx
        servicePort: 80
```

### Intelligent Connected Proxies

For more complex connections or resources such as service discovery, rate limiting, traffic management and advanced metrics, you may want to implement a service mesh.

A *service mesh* consists of edge and embedded proxies communicating with each other and handling traffic based on rules from a control plane. Various options are available, including Envoy, Istio, and linkerd.

![Istio Service Mesh](../../images/istio.png "Istio Service Mesh")

* [Envoy](https://www.envoyproxy.io/) is a modular and extensible proxy favored due to its modular construction, open architecture and dedication to remaining unmonetized. It is often used as a data plane under other tools of a service mesh.
* [Istio](https://istio.io/docs/concepts/what-is-istio/) is a powerful tool set which leverages Envoy proxies via a multi-component control plane. It is built to be platform-independent, and it can be used to make the service mesh flexible and feature-filled.
* [linderd](https://linkerd.io/) is another service mesh, purposely built to be easy to deploy, fast, and ultralight.

### Lab 10.1. Advanced Service Exposure

``` sh
kubectl create deployment secondapp --image=nginx
kubectl get deployments secondapp -o yaml | grep 'labels' -A2
kubectl expose deployment secondapp --type=NodePort --port=80
vim ingress.rbac.yaml
```

``` yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-ingress-controller
rules:

  + apiGroups:
      - ""

    resources:

      - services
      - endpoints
      - secrets

    verbs:

      - get
      - list
      - watch
  + apiGroups:
      - extensions

    resources:

      - ingresses

    verbs:

      - get
      - list
      - watch

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:

* kind: ServiceAccount

  name: traefik-ingress-controller
  namespace: kube-system
```

``` sh
kubectl create -f ingress.rbac.yaml
vim traefik-ds.yaml
```

``` yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
kind: DaemonSet
apiVersion: apps/v1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  selector:
    matchLabels:
      name: traefik-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      hostNetwork: True
      containers:

      - image: traefik:1.7.13

        name: traefik-ingress-lb
        ports:

        - name: http

          containerPort: 80
          hostPort: 80

        - name: admin

          containerPort: 8080
          hostPort: 8080
        args:

        - --api
        - --kubernetes
        - --logLevel=INFO

---
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:

    - protocol: TCP

      port: 80
      name: web

    - protocol: TCP

      port: 8080
      name: admin
```

``` sh
kubectl create -f traefik-ds.yaml
vim ingress.rule.yaml
```

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-test
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:

  + host: www.example.com

    http:
      paths:

      - backend:

          service:
            name: secondapp
            port: 
              number: 80
        path: /
        pathType: ImplementationSpecific
```

``` sh
kubectl create -f ingress.rule.yaml
ip a
curl -H "Host: www.example.com" http://k8smaster/
curl -H "Host: www.example.com" http://10.2.0.2
kubectl create deployment thirdpage --image=nginx
kubectl get deployment thirdpage -o yaml | grep -A2 'labels'
kubectl expose deployment thirdpage --type=NodePort --port=80
kubectl exec -it thirdpage-5867bc9dfd-j7ndw -- /bin/bash
apt-get update
apt-get install vim -y
vim /usr/share/nginx/html/index.html # <title>Third Page</title>
exit
vim ingress.rule.yaml
```

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-test
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:

  + host: www.example.com

    http:
      paths:

      - backend:

          service:
            name: secondapp
            port: 
              number: 80
        path: /
        pathType: ImplementationSpecific

  + host: thirdpage.org

    http:
      paths:

      - backend:

          service:
            name: thirdpage
            port: 
              number: 80
        path: /
        pathType: ImplementationSpecific
```

``` sh
kubectl apply -f ingress.rule.yaml
kubectl get ingress ingress-test -o yaml
curl -H "Host: thirdpage.org" http://k8smaster/
```

The **Traefik.io** ingress controller also presents a dashboard which allows you to monitor basic traffic.  From your localsystem open a browser and navigate to the public IP of your master node like this **<YOURPUBLICIP>:8080/dashboard/**. The trailing slash makes a difference. Follow the *HEALTH* and *PROVIDERS* links at the top, as well as the the node IP links to view traffic when you reference the pages, from inside or outside the node. Mistype the domain names inside the **curl** commands and you can also see 404 error traffic. Explore as time permits.

## Scheduling

### kube-scheduler

The larger and more diverse a Kubernetes deployment becomes, the more administration of scheduling can be important. The kube-scheduler determines which nodes will run a Pod, using a topology-aware algorithm. 

Users can set the priority of a pod, which will allow preemption of lower priority pods. The eviction of lower priority pods would then allow the higher priority pod to be scheduled.

The scheduler tracks the set of nodes in your cluster, filters them based on a set of predicates, then uses priority functions to determine on which node each Pod should be scheduled. The Pod specification as part of a request is sent to the kubelet on the node for creation. 

The default scheduling decision can be affected through the use of Labels on nodes or Pods. Labels of podAffinity, taints, and pod bindings allow for configuration from the Pod or the node perspective. Some, like tolerations, allow a Pod to work with a node, even when the node has a taint that would otherwise preclude a Pod being scheduled. 

Not all labels are drastic. Affinity settings may encourage a Pod to be deployed on a node, but would deploy the Pod elsewhere if the node was not available. Sometimes, documentation may use the term require, but practice shows the setting to be more of a request. As beta features, expect the specifics to change. Some settings will evict Pods from a node should the required condition no longer be true, such as **requiredDuringScheduling**, **RequiredDuringExecution**. 

Other options, like a custom scheduler, need to be programmed and deployed into your Kubernetes cluster.

### Predicates

The scheduler goes through a set of filters, or predicates, to find available nodes, then ranks each node using priority functions. The node with the highest rank is selected to run the Pod. 

**predicatesOrdering = []string{CheckNodeConditionPred, GeneralPred, HostNamePred, PodFitsHostPortsPred, MatchNodeSelectorPred, PodFitsResourcesPred, NoDiskConflictPred, PodToleratesNodeTaintsPred, PodToleratesNodeNoExecuteTaintsPred, CheckNodeLabelPresencePred, checkServiceAffinityPred, MaxEBSVolumeCountPred, MaxGCEPDVolumeCountPred, MaxAzureDiskVolumeCountPred, CheckVolumeBindingPred, NoVolumeZoneConflictPred, CheckNodeMemoryPressurePred, CheckNodeDiskPressurePred, MatchInterPodAffinityPred}**​

The predicates, such as **PodFitsHost** or **NoDiskConflict**, are evaluated in a particular and configurable order. In this way, a node has the least amount of checks for new Pod deployment, which can be useful to exclude a node from unnecessary checks if the node is not in the proper condition. 

For example, there is a filter called **HostNamePred**, which is also known as **HostName**, which filters out nodes that do not match the node name specified in the pod specification. Another predicate is **PodFitsResources** to make sure that the available CPU and memory can fit the resources required by the Pod. 

The scheduler can be updated by passing a configuration of **kind: Policy**, which can order predicates, give special weights to priorities, and even **hardPodAffinitySymmetricWeight**, which deploys Pods such that if we set Pod A to run with Pod B, then Pod B should automatically be run with Pod A.

### Priorities

**Priorities** are functions used to weight resources. Unless Pod and node affinity has been configured to the SelectorSpreadPriority setting, which ranks nodes based on the number of existing running pods, they will select the node with the least amount of Pods. This is a basic way to spread Pods across the cluster. 

Other priorities can be used for particular cluster needs. The **ImageLocalityPriorityMap** favors nodes which already have downloaded container images. The total sum of image size is compared with the largest having the highest priority, but does not check the image about to be used. 

Currently, there are more than ten included priorities, which range from checking the existence of a label to choosing a node with the most requested CPU and memory usage. You can view a list of priorities at **master/pkg/scheduler/algorithm/priorities**.

A stable feature as of v1.14 allows the setting of a **PriorityClass** and assigning pods via the use of **PriorityClassName** settings. This allows users to preempt, or evict, lower priority pods so that their higher priority pods can be scheduled. The kube-scheduler determines a node where the pending pod could run if one or more existing pods were evicted. If a node is found, the low priority pod(s) are evicted and the higher priority pod is scheduled. The use of a Pod Disruption Budget (PDB) is a way to limit the number of pods preemption evicts to ensure enough pods remain running. The scheduler will remove pods even if the PDB is violated if no other options are available.

### Scheduling Policies

The default scheduler contains a number of predicates and priorities; however, these can be changed via a scheduler policy file. 

A short version is shown below:

``` json
{
  "kind" : "Policy",
  "apiVersion" : "v1",
  "predicates" : [
          {"name" : "MatchNodeSelector", "order": 6},
          {"name" : "PodFitsHostPorts", "order": 2},
          {"name" : "PodFitsResources", "order": 3},
          {"name" : "NoDiskConflict", "order": 4},
          {"name" : "PodToleratesNodeTaints", "order": 5},
          {"name" : "PodFitsHost", "order": 1}
  ],
  "priorities" : [
          {"name" : "LeastRequestedPriority", "weight" : 1},
          {"name" : "BalancedResourceAllocation", "weight" : 1},       
          {"name" : "ServiceSpreadingPriority", "weight" : 2},
          {"name" : "EqualPriority", "weight" : 1}   
  ],
  "hardPodAffinitySymmetricWeight" : 10
}
```

Typically, you will configure a scheduler with this policy using the **--policy-config-file** parameter and define a name for this scheduler using the **--scheduler-name** parameter. You will then have two schedulers running and will be able to specify which scheduler to use in the pod specification.

With multiple schedulers, there could be conflict in the Pod allocation. Each Pod should declare which scheduler should be used. But, if separate schedulers determine that a node is eligible because of available resources and both attempt to deploy, causing the resource to no longer be available, a conflict would occur. The current solution is for the local kubelet to return the Pods to the scheduler for reassignment. Eventually, one Pod will succeed and the other will be scheduled elsewhere.

### Pod Specification

Most scheduling decisions can be made as part of the Pod specification. A pod specification contains several fields that inform scheduling, namely:

* The **nodeName** and **nodeSelector** options allow a Pod to be assigned to a single node or a group of nodes with particular labels.
* **Affinity** and **anti-affinity** can be used to require or prefer which node is used by the scheduler. If using a preference instead, a matching node is chosen first, but other nodes would be used if no match is present.
* The use of **taints** allows a node to be labeled such that Pods would not be scheduled for some reason, such as the master node after initialization. A **toleration** allows a Pod to ignore the taint and be scheduled assuming other requirements are met.
* Should none of the options above meet the needs of the cluster, there is also the ability to deploy a custom scheduler. Each Pod could then include a **schedulerName** to choose which schedule to use.

### Specifying the Node Label

The **nodeSelector** field in a pod specification provides a straightforward way to target a node or a set of nodes, using one or more key-value pairs.

``` yaml
spec:
  containers:

  + name: redis

    image: redis
  nodeSelector:
    net: fast
```

Setting the **nodeSelector** tells the scheduler to place the pod on a node that matches the labels. All listed selectors must be met, but the node could have more labels. In the example above, any node with a key of **net** set to **fast** would be a candidate for scheduling. Remember that labels are administrator-created tags, with no tie to actual resources. This node could have a slow network. 

The pod would remain Pending until a node is found with the matching labels.

The use of affinity/anti-affinity should be able to express every feature as nodeSelector.

### Pod Affinity Rules

Pods which may communicate a lot or share data may operate best if co-located, which would be a form of affinity. For greater fault tolerance, you may want Pods to be as separate as possible, which would be anti-affinity. These settings are used by the scheduler based on the labels of Pods that are already running. As a result, the scheduler must interrogate each node and track the labels of running Pods. Clusters larger than several hundred nodes may see significant performance loss. Pod affinity rules use **In**, **NotIn**, **Exists**, and **DoesNotExist** operators.

* The use of **requiredDuringSchedulingIgnoredDuringExecution** means that the Pod will not be scheduled on a node unless the following operator is true. If the operator changes to become false in the future, the Pod will continue to run. This could be seen as a hard rule.
* Similarly, **preferredDuringSchedulingIgnoredDuringExecution** will choose a node with the desired setting before those without. If no properly-labeled nodes are available, the Pod will execute anyway. This is more of a soft setting, which declares a preference instead of a requirement.
* With the use of **podAffinity**, the scheduler will try to schedule Pods together.
* The use of **podAntiAffinity** would cause the scheduler to keep Pods on different nodes.
* The **topologyKey** allows a general grouping of Pod deployments. Affinity (or the inverse anti-affinity) will try to run on nodes with the declared topology key and running Pods with a particular label. The **topologyKey** could be any legal key, with some important considerations.
  + If using **requiredDuringScheduling** and the admission controller **LimitPodHardAntiAffinityTopology** setting, the **topologyKey** must be set to **kubernetes.io/hostname**. 
  + If using **PreferredDuringScheduling**, an empty **topologyKey** is assumed to be all, or the combination of **kubernetes.io/hostname**, **topology.kubernetes.io/zone** and **topology.kubernetes.io/region**.

### podAffinity Example

An example of **affinity** and **podAffinity** settings can be seen below. This also requires a particular label to be matched when the Pod starts, but not required if the label is later removed. 

``` yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:

      - labelSelector:

          matchExpressions:

          - key: security

            operator: In
            values:

            - S1

        topologyKey: topology.kubernetes.io/zone
```

Inside the declared topology zone, the Pod can be scheduled on a node running a Pod with a key label of **security** and a value of **S1**. If this requirement is not met, the Pod will remain in a **Pending** state.

### podAntiAffinity Example

With **podAntiAffinity**, we can prefer to avoid nodes with a particular label. In this case, the scheduler will prefer to avoid a node with a key set to **security** and value of **S2**. 

``` yaml
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:

  + weight: 100

    podAffinityTerm:
      labelSelector:
        matchExpressions:

        - key: security

          operator: In
          values:

          - S2

    topologyKey: kubernetes.io/hostname 
```

In a large, varied environment, there may be multiple situations to be avoided. As a preference, this setting tries to avoid certain labels, but will still schedule the Pod on some node. As the Pod will still run, we can provide a weight to a particular rule. The weights can be declared as a value from 1 to 100. The scheduler then tries to choose, or avoid the node with the greatest combined value.

### Node Affinity Rules

Where Pod affinity/anti-affinity has to do with other Pods, the use of **nodeAffinity** allows Pod scheduling based on node labels. This is similar and will some day replace the use of the **nodeSelector** setting. The scheduler will not look at other Pods on the system, but the labels of the nodes. This should have much less performance impact on the cluster, even with a large number of nodes.

* Uses **In**, **NotIn**, **Exists**, **DoesNotExist** operators
* **requiredDuringSchedulingIgnoredDuringExecution**
* **preferredDuringSchedulingIgnoredDuringExecution**
* Planned for future: **requiredDuringSchedulingRequiredDuringExecution**.

Until **nodeSelector** has been fully deprecated, both the selector and required labels must be met for a Pod to be scheduled.

### Node Affinity Example

``` yaml
spec:
  affinity:
    nodeAffinity: 
      requiredDuringSchedulingIgnoredDuringExecution:  
        nodeSelectorTerms:

        - matchExpressions:
          - key: kubernetes.io/colo-tx-name

            operator: In
            values:

            - tx-aus
            - tx-dal

      preferredDuringSchedulingIgnoredDuringExecution:

      - weight: 1

        preference:
          matchExpressions:

          - key: disk-speed

            operator: In
            values:

            - fast
            - quick 

```

The first **nodeAffinity** rule requires a node with a key of **kubernetes.io/colo-tx-name** which has one of two possible values: **tx-aus** or **tx-dal**. 

The second rule gives extra weight to nodes with a key of **disk-speed** with a value of **fast** or **quick**. The Pod will be scheduled on some node - in any case, this just prefers a particular label.

### Taints

A node with a particular taint will repel Pods without tolerations for that taint. A taint is expressed as **key=value:effect**. The key and the value are created by the administrator.

The key and value used can be any legal string, and this allows flexibility to prevent Pods from running on nodes based off of any need. If a Pod does not have an existing toleration, the scheduler will not consider the tainted node.

There are three effects, or ways to handle Pod scheduling:

* **NoSchedule**: The scheduler will not schedule a Pod on this node, unless the Pod has this toleration. Existing Pods continue to run, regardless of toleration.
* **PreferNoSchedule**: The scheduler will avoid using this node, unless there are no untainted nodes for the Pods toleration. Existing Pods are unaffected.
* **NoExecute**: This taint will cause existing Pods to be evacuated and no future Pods scheduled. Should an existing Pod have a toleration, it will continue to run. If the Pod **tolerationSeconds** is set, they will remain for that many seconds, then be evicted. Certain node issues will cause the kubelet to add 300 second tolerations to avoid unnecessary evictions.

If a node has multiple taints, the scheduler ignores those with matching tolerations. The remaining unignored taints have their typical effect. 

The use of **TaintBasedEvictions** is still an alpha feature. The kubelet uses taints to rate-limit evictions when the node has problems.

### Tolerations

Setting tolerations on a node are used to schedule Pods on tainted nodes. This provides an easy way to avoid Pods using the node. Only those with a particular toleration would be scheduled.

An operator can be included in a Pod specification, defaulting to **Equal** if not declared. The use of the operator **Equal** requires a value to match. The **Exists** operator should not be specified. If an empty key uses the **Exists** operator, it will tolerate every taint. If there is no effect, but a key and operator are declared, all effects are matched with the declared key.

``` yaml
tolerations:

* key: "server"

  operator: "Equal"
  value: "ap-east"
  effect: "NoExecute"
  tolerationSeconds: 3600
```

In the above example, the Pod will remain on the server with a key of server and a value of ap-east for 3600 seconds after the node has been tainted with NoExecute. When the time runs out, the Pod will be evicted.

### Custom Scheduler

If the default scheduling mechanisms (affinity, taints, policies) are not flexible enough for your needs, you can write your own scheduler. The programming of a custom scheduler is outside the scope of this course, but you may want to start with the existing scheduler code, which can be found in the [Scheduler repository on GitHub](https://github.com/kubernetes/kubernetes/tree/master/pkg/scheduler).

If a Pod specification does not declare which scheduler to use, the standard scheduler is used by default. If the Pod declares a scheduler, and that container is not running, the Pod would remain in a **Pending** state forever. 

The end result of the scheduling process is that a pod gets a binding that specifies which node it should run on. A binding is a Kubernetes API primitive in the **api/v1** group. Technically, without any scheduler running, you could still schedule a pod on a node, by specifying a binding for that pod. 

You can also run multiple schedulers simultaneously.

You can view the scheduler and other information with:

``` sh
kubectl get events
```

### Lab 11.1. Assign Pods Using Labels

``` sh
kubectl get nodes
kubectl describe nodes | grep -A5 -i label
kubectl describe nodes | grep -i taint
kubectl get deployments --all-namespaces
sudo docker ps | wc -l
kubectl label nodes master status=vip
kubectl label nodes worker status=other
kubectl get nodes --show-labels
vim vip.yaml
```

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: vip
spec:
  containers:

  + name: vip1

    image: busybox
    args:

    - sleep
    - "1000000"
  + name: vip2

    image: busybox
    args:

    - sleep
    - "1000000"
  + name: vip3

    image: busybox
    args:

    - sleep
    - "1000000"
  + name: vip4

    image: busybox
    args:

    - sleep
    - "1000000"

  nodeSelector:
    status: vip
```

``` sh
kubectl create -f vip.yaml
sudo docker ps | wc -l
kubectl delete pod vip
vim vip.yaml # Comment nodeSelector
kubectl get pods
kubectl create -f vip.yaml
sudo docker ps | wc -l
cp vip.yaml other.yaml
sed -i s/vip/other/g other.yaml
vim other.yaml # Add nodeSelector: status: other
kubectl create -f other.yaml
sudo docker ps | wc -l
kubectl get pod -o wide
kubectl delete pods vip other
kubectl get pods
```

### Lab 11.2. Using Taints to Control Pod Deployment

``` sh
kubectl delete deployment secondapp thirdpage nginx
vim taint.yaml
```

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: taint-deployment
spec:
  replicas: 8
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:

      - name:  nginx

        image: nginx:1.16.1
        ports:

        - containerPort: 80

```

``` sh
kubectl apply -f taint.yaml
kubectl get pods -o wide
sudo docker ps | grep nginx
sudo docker ps | wc -l
kubectl delete deployment taint-deployment
sudo docker ps | wc -l
kubectl taint nodes worker bubba=value:PreferNoSchedule
kubectl describe node | grep Taint
kubectl apply -f taint.yaml
kubectl get pods -o wide
kubectl delete deployment taint-deployment
kubectl taint nodes worker bubba-
kubectl describe node | grep Taint
kubectl taint nodes worker bubba=value:NoSchedule
kubectl apply -f taint.yaml
kubectl get pods -o wide
kubectl delete deployment taint-deployment
kubectl taint nodes worker bubba-
kubectl get pods -o wide
kubectl apply -f taint.yaml
kubectl get pods -o wide
kubectl taint nodes worker bubba=value:NoExecute
kubectl get pods -o wide
kubectl taint nodes worker bubba-
kubectl delete deployment taint-deployment
```

## Logging and Troubleshooting

Kubernetes relies on API calls and is sensitive to network issues. Standard Linux tools and processes are the best method for troubleshooting your cluster. If a shell, such as bash, is not available in an affected Pod, consider deploying another similar pod with a shell, like **busybox**. DNS configuration files and tools like **dig** are a good place to start. For more difficult challenges, you may need to install other tools, like **tcpdump**.

Large and diverse workloads can be difficult to track, so monitoring of usage is essential. Monitoring is about collecting key metrics, such as CPU, memory, and disk usage, and network bandwidth on your nodes, as well as monitoring key metrics in your applications. These features are being ingested into Kubernetes with the Metric Server, which is a cut-down version of the now deprecated Heapster. Once installed, the Metrics Server exposes a standard API which can be consumed by other agents, such as autoscalers. Once installed, this endpoint can be found here on the master server: */apis/metrics/k8s.io/*.

Logging activity across all the nodes is another feature not part of Kubernetes. Using Fluentd can be a useful data collector for a unified logging layer. Having aggregated logs can help visualize the issues, and provides the ability to search all logs. It is a good place to start when local network troubleshooting does not expose the root cause. It can be downloaded from the [Fluentd website](https://www.fluentd.org/).

Another project from CNCF combines logging, monitoring, and alerting and is called Prometheus - you can learn more from the [Prometheus website](https://prometheus.io/). It provides a time-series database, as well as integration with Grafana for visualization and dashboards. 

We are going to review some of the basic **kubectl** commands that you can use to debug what is happening, and we will walk you through the basic steps to be able to debug your containers, your pending containers, and also the systems in Kubernetes.

### Basic Troubleshooting Steps

The troubleshooting flow should start with the obvious. If there are errors from the command line, investigate them first. The symptoms of the issue will probably determine the next step to check. Working from the application running inside a container to the cluster as a whole may be a good idea. The application may have a shell you can use, for example: 

``` sh
kubectl create deploy busybox --image=busybox -- sleep 3600
kubectl exec -ti <busybox_pod> -- /bin/sh
```

If the Pod is running, use **kubectl logs pod-name** to view the standard out of the container. Without logs, you may consider deploying a sidecar container in the Pod to generate and handle logging. The next place to check is networking, including DNS, firewalls and general connectivity, using standard Linux commands and tools. 

Security settings can also be a challenge. RBAC, covered in the security chapter, provides mandatory or discretionary access control in a granular manner. SELinux and AppArmor are also common issues, especially with network-centric applications. 

A newer feature of Kubernetes is the ability to enable auditing for the kube-apiserver, which can allow a view into actions after the API call has been accepted.

The issues found with a decoupled system like Kubernetes are similar to those of a traditional datacenter, plus the added layers of Kubernetes controllers: 

* Errors from the command line
* Pod logs and state of Pods
* Use shell to troubleshoot Pod DNS and network
* Check node logs for errors, make sure there are enough resources allocated
* RBAC, SELinux or AppArmor for security settings​
* API calls to and from controllers to kube-apiserver
* Enable auditing
* Inter-node network issues, DNS and firewall
* Master server controllers (control Pods in pending or error state, errors in log files, sufficient resources, etc).

### Ephemeral Containers

A feature new to the 1.16 version is the ability to add a container to a running pod. This would allow a feature-filled container to be added to an existing pod without having to terminate and re-create. Intermittent and difficult to determine problems may take a while to reproduce, or not exist with the addition of another container.

As an alpha stability feature, it may change or be removed at any time. As well, they will not be restarted automatically, and several resources such as ports or resources are not allowed.

These containers are added via the **ephemeralcontainers** handler via an API call, not via the **podSpec**. As a result, the use of **kubectl edit** is not possible.

You may be able to use the **kubectl attach** command to join an existing process within the container. This can be helpful instead of **kubectl exec**, which executes a new process. The functionality of the attached process depends entirely on what you are attaching to.

``` sh
kubectl debug buggypod --image debian --attach
```

### Cluster Start Sequence

The cluster startup sequence begins with systemd if you built the cluster using **kubeadm**. Other tools may leverage a different method. Use **systemctl status kubelet.service** to see the current state and configuration files used to run the kubelet binary.

* Uses */etc/systemd/system/kubelet.service.d/10-kubeadm.conf*

Inside of the *config.yaml* file you will find several settings for the binary, including the **staticPodPath** which indicates the directory where kubelet will read every yaml file and start every pod. If you put a yaml file in this directory, it is a way to troubleshoot the scheduler, as the pod is created with any requests to the scheduler.

* Uses */var/lib/kubelet/config.yaml* configuration file
* **staticPodPath** is set to */etc/kubernetes/manifests/*

The four default yaml files will start the base pods necessary to run the cluster:

* kubelet creates all pods from *.yaml in directory: kube-apiserver, etcd, kube-controller-manager, kube-scheduler.

Once the watch loops and controllers from kube-controller-manager run using etcd data, the rest of the configured objects will be created.

### Monitoring

Monitoring is about collecting metrics from the infrastructure, as well as applications. 

The long used and now deprecated Heapster has been replaced with an integrated Metrics Server. Once installed and configured, the server exposes a standard API which other agents can use to determine usage. It can also be configured to expose custom metrics, which then could also be used by autoscalers to determine if an action should take place.

Prometheus is part of the Cloud Native Computing Foundation (CNCF). As a Kubernetes plugin, it allows one to scrape resource usage metrics from Kubernetes objects across the entire cluster. It also has several client libraries which allow you to instrument your application code in order to collect application-level metrics.

### Using krew

We have been using the **kubectl** command throughout the course. The basic commands can be used together in a more complex manner extending what can be done. There are over seventy and growing plugins available to interact with Kubernetes objects and components.

At the time this course was written, plugins cannot overwrite existing **kubectl** commands, nor can it add sub-commands to existing commands. Writing new plugins should take into account the command line runtime package and a Go library for plugin authors.

As a plugin the declaration of options such as namespace or container to use must come after the command.

``` sh
kubectl sniff bigpod-abcd-123 -c mainapp -n accounting
```

Plugins can be distributed in many ways. The use of **krew** (the **kubectl** plugin manager) allows for cross-platform packaging and a helpful plugin index, which makes finding new plugins easy.

Install the software using steps available in [krew's GitHub repository](https://github.com/kubernetes-sigs/krew/).

``` sh
kubectl krew help
```

You can invoke krew through kubectl:

``` sh
kubectl krew [command]...
```

Usage:

``` sh
krew [command]
```

More information can be found in the Kubernetes Documentation, [extend kubectl with plugins](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/).

![Krew Commands](../../images/krew.png "Krew Commands")

### Managing Plugins

The **help** option explains basic operation. After installation ensure the $PATH includes the plugins. krew should allow easy installation and use after that.

``` sh
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
kubectl krew search
kubectl krew install tail
kubectl plugin list
kubectl krew search
```

Once installed use as **kubectl** sub-command. You can also upgrade and uninstall.

### Sniffing Traffic With Wireshark

Cluster network traffic is encrypted making troubleshooting of possible network issues more complex. Using the sniff plugin you can view the traffic from within. sniff requires Wireshark and ability to export graphical display.

The **sniff** command will use the first found container unless you pass the -c option to declare which container in the pod to use for traffic monitoring.

``` sh
kubectl krew install sniff nginx-123456-abcd -c webcont
```

### Logging Tools

Logging, like monitoring, is a vast subject in IT. It has many tools that you can use as part of your arsenal.

Typically, logs are collected locally and aggregated before being ingested by a search engine and displayed via a dashboard which can use the search syntax. While there are many software stacks that you can use for logging, the [Elasticsearch, Logstash, and Kibana Stack](https://www.elastic.co/videos/introduction-to-the-elk-stack) (ELK) has become quite common.

In Kubernetes, the kubelet writes container logs to local files (via the Docker logging driver). The **kubectl logs** command allows you to retrieve these logs.

Cluster-wide, you can use [Fluentd](https://www.fluentd.org/) to aggregate logs. Check out the [cluster administration logging concepts](https://kubernetes.io/docs/concepts/cluster-administration/logging/) for a detailed description.

Fluentd is part of the Cloud Native Computing Foundation and, together with Prometheus, they make a nice combination for monitoring and logging. You can find a [detailed walk-through of running Fluentd on Kubernetes](https://kubernetes.io/docs/tasks/debug-application-cluster/logging-elasticsearch-kibana/) in the Kubernetes documentation.

Setting up Fluentd for Kubernetes logging is a good exercise in understanding DaemonSets. Fluentd agents run on each node via a DaemonSet, they aggregate the logs, and feed them to an Elasticsearch instance prior to visualization in a Kibana dashboard.

### Lab 12.1. Review Log File Locations

``` sh
journalctl -u kubelet | less
sudo find / -name "*apiserver*log"
sudo less /var/log/containers/kube-apiserver-master_kube-system_kube-apiserver-d8b4274a8ce5c2e39d31b7c2dd09500f696acc788093e5766fc9c12bdb36954d.log
sudo find / -name "*coredns*log"
sudo find / -name "*kube-proxy*log"
sudo find / -name "*kube-scheduler*log"
sudo find / -name "*kube-controller*log"
```

### Lab 12.2. Viewing Log Outputs

``` sh
kubectl get po --all-namespaces
kubectl -n kube-system logs kube-apiserver-master
```

### Lab 12.3. Adding Tools for Monitoring and Metrics

``` sh
git clone https://github.com/kubernetes-incubator/metrics-server.git
cd metrics-server/ ; less README.md
kubectl create -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.7/components.yaml
kubectl -n kube-system get pods
kubectl -n kube-system edit deployment metrics-server # - --kubelet-insecure-tls                    #<-- Add this line8- --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname #<--May be needed
kubectl -n kube-system logs metrics-server-67cd7bd5f6-6mv6s
sleep 120 ; kubectl top pod --all-namespaces
kubectl top nodes
cd
curl --cert ./client.pem --key ./client-key.pem --cacert ./ca.pem https://k8smaster:6443/apis/metrics.k8s.io/v1beta1/nodes
kubectl create -f https://bit.ly/2OFQRMy
kubectl get svc --all-namespaces
kubectl -n kubernetes-dashboard edit svc kubernetes-dashboard # NodePort
kubectl create clusterrolebinding dashaccess --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:kubernetes-dashboard
curl ifconfig.io
kubectl -n kubernetes-dashboard describe secrets kubernetes-dashboard-token-5n6gs
```

Navigate to [Dashboard](https://35.202.145.17:30650) on Firefox and paste the Token to Sign In.

## Custom Resource Definitions

### Custom Resources

We have been working with built-in resources, or API endpoints. The flexibility of Kubernetes allows for the dynamic addition of new resources as well. Once these Custom Resources have been added, the objects can be created and accessed using standard calls and commands, like **kubectl**. The creation of a new object stores new structured data in the etcd database and allows access via kube-apiserver. 

To make a new custom resource part of a declarative API, there needs to be a controller to retrieve the structured data continually and act to meet and maintain the declared state. This controller, or operator, is an agent that creates and manages one or more instances of a specific stateful application. We have worked with built-in controllers such as Deployments, DaemonSets and other resources. 

The functions encoded into a custom operator should be all the tasks a human would need to perform if deploying the application outside of Kubernetes. The details of building a custom controller are outside the scope of this course, and thus, not included. 

There are two ways to add custom resources to your Kubernetes cluster. The easiest way, but less flexible, is by adding a Custom Resource Definition (CRD) to the cluster. The second way, which is more flexible, is the use of Aggregated APIs (AA), which requires a new API server to be written and added to the cluster. 

Either way of adding a new object to the cluster, as distinct from a built-in resource, is called a Custom Resource.

If you are using RBAC for authorization, you probably will need to grant access to the new CRD resource and controller. If using an Aggregated API, you can use the same or a different authentication process.

### Custom Resource Definitions

As we have already learnt, the decoupled nature of Kubernetes depends on a collection of watcher loops, or controllers, interrogating the kube-apiserver to determine if a particular configuration is true. If the current state does not match the declared state, the controller makes API calls to modify the state until they do match. If you add a new API object and controller, you can use the existing kube-apiserver to monitor and control the object. The addition of a Custom Resource Definition will be added to the cluster API path, currently under **apiextensions.k8s.io/v1**.

While this is the easiest way to add a new object to the cluster, it may not be flexible enough for your needs. Only the existing API functionality can be used. Objects must respond to REST requests and have their configuration state validated and stored in the same manner as built-in objects. They would also need to exist with the protection rules of built-in objects.

A CRD allows the resource to be deployed in a namespace or be available in the entire cluster. The YAML file sets this with the **scope:** parameter, which can be set to **Namespaced** or **Cluster**.

Prior to v1.8, there was a resource type called ThirdPartyResource (TPR). This has been deprecated and is no longer available. All resources will need to be rebuilt as CRD. After upgrading, existing TPRs will need to be removed and replaced by CRDs such that the API URL points to functional objects.

### Configuration example

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.stable.linux.com
spec:
  group: stable.linux.com
  version: v1
  scope: Namespaced
  names:
    plural: backups
    singular: backup
    shortNames:
    - bks
    kind: BackUp
```

- apiVersion: This should match the current level of stability, which is **apiextensions.k8s.io/v1**.
- kind: CustomResourceDefinition. The object type being inserted by the kube-apiserver.
- name: backups.stable.linux.com. The name must match the **spec** field declared later. The syntax must be **<plural name>.<group>**.
- group: stable.linux.com. The group name will become part of the REST API under **/apis/<group>/<version>** or **/apis/stable/v1** in this case with the version set to v1.
- scope: Determines if the object exists in a single namespace or is cluster-wide.
- plural: Defines the last part of the API URL, such as **apis/stable/v1/backups**.
- singular and shortNames: They represent the name displayed and make CLI usage easier.
- kind: A CamelCased singular type used in resource manifests.

### New Object Configuration

```yaml
apiVersion: "stable.linux.com/v1"
kind: BackUp
metadata:
  name: a-backup-object
spec:
  timeSpec: "* * * * */5"
  image: linux-backup-image
replicas: 5
```

Note that the **apiVersion** and **kind** match the CRD we created in a previous step. The **spec** parameters depend on the controller.

The object will be evaluated by the controller. If the syntax, such as **timeSpec**, does not match the expected value, you will receive and error, should validation be configured. Without validation, only the existence of the variable is checked, not its details.

### Optional Hooks

Just as with built-in objects, you can use an asynchronous pre-delete hook known as a **Finalizer**. If an API **delete** request is received, the object metadata field **metadata.deletionTimestamp** is updated. The controller then triggers whichever finalizer has been configured. When the finalizer completes, it is removed from the list. The controller continues to complete and remove finalizers until the string is empty. Then, the object itself is deleted.

#### Finalizer

```yaml
metadata:
  finalizers:
  - finalizer.stable.linux.com
```

#### Validation

```yaml
validation:
    openAPIV3Schema:
      properties:
        spec:
          properties:
            timeSpec:
              type: string 
              pattern: '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
            replicas:
              type: integer
              minimum: 1
              maximum: 10
```

A feature in beta starting with v1.9 allows for validation of custom objects via the OpenAPI v3 schema. This will check various properties of the object configuration being passed by the API server. In the example above, the **timeSpec** must be a string matching a particular pattern and the number of allowed replicas is between 1 and 10. If the validation does not match, the error returned is the failed line of validation.

### Understanding Aggregated APIs

The use of Aggregated APIs allows adding additional Kubernetes-type API servers to the cluster. The added server acts as a subordinate to kube-apiserver, which, as of v1.7, runs the aggregation layer in-process. When an extension resource is registered, the aggregation layer watches a passed URL path and proxies any requests to the newly registered API service. 

The aggregation layer is easy to enable. Edit the flags passed during startup of the kube-apiserver to include **--enable-aggregator-routing=true**. Some vendors enable this feature by default. 

The creation of the exterior can be done via YAML configuration files or APIs. Configuring TLS authorization between components and RBAC rules for various new objects is also required. A [sample API server](https://github.com/kubernetes/sample-apiserver) is available on GitHub. A project currently in the incubation stage is an [API server builder](https://github.com/kubernetes-sigs/apiserver-builder-alpha) which should handle much of the security and connection configuration.

### Lab 13.1. Create a Custom Resource Definition

```sh
kubectl get crd --all-namespaces
vim crd.yaml
```

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.training.lfs458.com
         # This name must match names below.
         # <plural>.<group> syntax
spec:
  scope: Cluster    #Could also be Namespaced
  group: training.lfs458.com
  version: v1
  names:
    kind: CronTab      #Typically CamelCased for resource manifest
    plural: crontabs   #Shown in URL
    singular: crontab  #Short name for CLI alias
    shortNames:
    - ct               #CLI short name
```

```sh
kubectl create -f crd.yaml
kubectl get crd
kubectl describe crd crontabs.training.lfs458.com
vim new-crontab.yaml
```

```yaml
apiVersion: "training.lfs458.com/v1"
    # This is from the group and version of new CRD
kind: CronTab
    # The kind from the new CRD
metadata:
  name: new-cron-object
spec:
  cronSpec: "*/5 * * * *"
  image: some-cron-image
    #Does not exist
```

```sh
kubectl create -f new-crontab.yaml
kubectl get CronTab
kubectl get ct
kubectl describe ct
kubectl delete -f crd.yaml
kubectl get ct # Error
```

## Helm

### Deploying Complex Applications

We have used Kubernetes tools to deploy simple Docker applications. Starting with the v1.4 release, the goal was to have a canonical location for software. Helm is similar to a package manager like **yum** or **apt**, with a chart being similar to a package. Helm v3 is significantly different than v2.

A typical containerized application will have several manifests. Manifests for deployments, services, and ConfigMaps. You will probably also create some secrets, Ingress, and other objects. Each of these will need a manifest. 

With Helm, you can package all those manifests and make them available as a single tarball. You can put the tarball in a repository, search that repository, discover an application, and then, with a single command, deploy and start the entire application. 

The server runs in your Kubernetes cluster, and your client is local, even a local laptop. With your client, you can connect to multiple repositories of applications. 

You will also be able to upgrade or roll back an application easily from the command line.

### Helm v2 and Tiller

The helm tool packages a Kubernetes application using a series of YAML files into a chart, or package. This allows for simple sharing between users, tuning using a templating scheme, as well as provenance tracking, among other things.

![Basic Helm and Tiller Flow](../../images/helm-tiller.png "Basic Helm and Tiller Flow")

Helm v2 is made of two components:
- A server called Tiller, which runs inside your Kubernetes cluster.
- A client called Helm, which runs on your local machine.

Helm version 2 uses a Tiller pod to deploy in the cluster. This has led to a lot of issues with security and cluster permissions. The new Helm v3 does not deploy a pod.

With the Helm client you can browse package repositories (containing published Charts), and deploy those Charts on your Kubernetes cluster. The Helm will download the chart and pass a request to Tiller to create a release, otherwise known as an instance of a chart. The release will be made of various resources running in the Kubernetes cluster.

### Helm v3

With the near complete overhaul of Helm, the processes and commands have changed quite a bit. Expect to spend some time updating and integrating these changes if you are currently using Helm v2.

One of the most noticeable changes is the removal of the Tiller pod. This was an ongoing security issue, as the pod needed elevated permissions to deploy charts. The functionality is in the command alone, and no longer requires initialization to use.

In version 2, an update to a chart and deployment used a 2-way strategic merge for patching. This compared the previous manifest to the intended manifest, but not the possible edits done outside of helm commands. The third way now checked is the live state of objects.

Among other changes, software installation no longer generates a name automatically. One must be provided, or the **--generated-name** option must be passed.

### Chart Contents

A **chart** is an archived set of Kubernetes resource manifests that make up a distributed application. You can learn more from the [Helm 3 documentation](https://helm.sh/docs/topics/charts/). Others exist and can be easily created, for example by a vendor providing software. Charts are similar to the use of independent YUM repositories.

```tree
├── Chart.yaml
├── README.md
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── pvc.yaml
│   ├── secrets.yaml
│   └── svc.yaml
└── values.yaml
```

- **Chart.yaml**: contains some metadata about the Chart, like its name, version, keywords, and so on, in this case, for MariaDB.
- **values.yaml**: contains keys and values that are used to generate the release in your cluster. These values are replaced in the resource manifests using the Go templating syntax.
- **templates**: directory contains the resource manifests that make up this MariaDB application.

### Templates

The templates are resource manifests that use the Go templating syntax. Variables defined in the **values.yaml** file, for example, get injected in the template when a release is created. In the MariaDB example we provided, the database passwords are stored in a Kubernetes secret, and the database configuration is stored in a Kubernetes ConfigMap.

We can see that a set of labels are defined in the Secret metadata using the Chart name, Release name, etc. The actual values of the passwords are read from the **values.yaml** file.

```yaml
apiVersion: v1
kind: Secret
metadata:
    name: {{ template "fullname" . }}
    labels:
        app: {{ template "fullname" . }}
        chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
        release: "{{ .Release.Name }}"
        heritage: "{{ .Release.Service }}"
type: Opaque
data:
    mariadb-root-password: {{ default "" .Values.mariadbRootPassword | b64enc | quote }}
    mariadb-password: {{ default "" .Values.mariadbPassword | b64enc | quote }}
```

### Initializing Helm v2

Helm v3 does not need to be initialized.

As always, you can build Helm from source, or download a tarball. We expect to see Linux packages for the stable release soon. The current RBAC security requirements to deploy helm require the creation of a new **serviceaccount** and assigning of permissions and roles. There are several optional settings which can be passed to the **helm init** command, typically for particular security concerns, storage options and also a dry-run option.

```sh
helm init
kubectl get deployments --namespace=kube-system # tiller-deploy pod
```

The helm v2 initialization should have created a new **tiller-deploy** pod in your cluster. Please note that this will create a deployment in the **kube-system** namespace.

The client will be able to communicate with the tiller pod using port forwarding. Hence, you will not see any service exposing tiller.

### Chart Repositories

A default repository is included when initializing helm, but it's common to add other repositories. Repositories are currently simple HTTP servers that contain an index file and a tarball of all the Charts present. 

You can interact with a repository using the **helm repo** commands. Once you have a repository available, you can search for Charts based on keywords. Below, we search for a **redis** Chart:

```sh
helm repo add testing http://storage.googleapis.com/kubernetes-charts-testing
helm repo list
helm search redis
```

Once you find the chart within a repository, you can deploy it on your cluster.

### Deploying a Chart

To deploy a Chart, you can just use the **helm install** command. There may be several required resources for the installation to be successful, such as available PVs to match chart PVC. Currently, the only way to discover which resources need to exist is by reading the **READMEs** for each chart:

```sh
helm install testing/redis-standalone
helm list
```

You will be able to list the release, delete it, even upgrade it and roll back.

A unique, colorful name will be created for each helm instance deployed. You can also use **kubectl** to view new resources Helm created in your cluster. 

The output of the deployment should be carefully reviewed. It often includes information on access to the applications within. If your cluster did not have a required cluster resource, the output is often the first place to begin troubleshooting.

### Lab 14.1. Working with Helm and Charts

```sh
wget https://get.helm.sh/helm-v3.0.0-linux-amd64.tar.gz
tar -xvf helm-v3.0.0-linux-amd64.tar.gz
sudo cp linux-amd64/helm /usr/local/bin/helm3
helm3 search hub database
helm3 repo add stable https://charts.helm.sh/stable
helm3 repo update
helm3 --debug install firstdb stable/mariadb --set master.persistence.enabled=false --set slave.persistence.enabled=false
kubectl get secret --namespace default firstdb-mariadb -o jsonpath="{.data.mariadb-root-password}" | base64 --decode
kubectl run firstdb-mariadb-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mariadb:10.3.22-debian-10-r27 --namespace default --command -- bash
mysql -h firstdb-mariadb.default.svc.cluster.local -uroot -p my_database # Enter pass
SHOW DATABASES;
quit
exit
helm3 list
helm3 uninstall firstdb
helm3 list
find $HOME -name *mariadb*
cd $HOME/.cache/helm/repository ; tar -xvf mariadb-*
cp mariadb/values.yaml $HOME/custom.yaml ; cd
less custom.yaml
vim custom.yaml # Add password on rootUser.passsword and change persistance.enabled
helm3 install -f custom.yaml seconddb stable/mariadb
kubectl run seconddb-mariadb-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mariadb:10.3.22-debian-10-r27 --namespace default --command -- bash
mysql -h seconddb-mariadb.default.svc.cluster.local -uroot -p my_database # Enter new pass
SHOW DATABASES;
quit
exit
helm3 uninstall seconddb
```

## Security

Security is a big and complex topic, especially in a distributed system like Kubernetes. Thus, we are just going to cover some of the concepts that deal with security in the context of Kubernetes.

Then, we are going to focus on the authentication aspect of the API server and we will dive into authorization, looking at things like ABAC and RBAC, which is now the default configuration when you bootstrap a Kubernetes cluster with **kubeadm**.

We are going to look at the **admission control** system, which lets you look at and possibly modify the requests that are coming in, and do a final deny or accept on those requests.

Following that, we're going to look at a few other concepts, including how you can secure your Pods more tightly using security contexts and pod security policies, which are full-fledged API objects in Kubernetes.

Finally, we will look at network policies. By default, we tend not to turn on network policies, which let any traffic flow through all of our pods, in all the different namespaces. Using network policies, we can actually define Ingress rules so that we can restrict the Ingress traffic between the different namespaces. The network tool in use, such as Flannel or Calico will determine if a network policy can be implemented. As Kubernetes becomes more mature, this will become a strongly suggested configuration.

### Cloud Security Considerations

Keeping a cloud environment secure is an ongoing, and wide-ranging task. As more moves to the cloud, we must look at more than just Kubernetes towards the hardware, software, and configuration options for the entire environment. Starting in the design phase, care must be taken to secure safe hardware, firmware and operating system binaries.

Once the platform is hardened, the kube-apiserver has a list of considerations, tools, and settings to limit access and formalize access in an easy-to-understand manner.

As a network-intensive environment, it becomes important to secure the network both inside Kubernetes, as done with a NetworkPolicy, as well as traditional firewall tools and pod-to-pod encryption.

Minimizing base images, insisting on container immutability, and static and runtime analysis of tools is also an important part of security, which often begins with developers and is implemented in the CI/CD pipeline prior to an image being used in a production cluster. Tools like AppArmor and SELinux should also be used to further protect the environment from malicious containers.

Security is more than just settings and configuration. It is an ongoing process of issue detection using intrusion detection tools and behavioral analytics. There needs to be an ongoing process of assessment, prevention, detection, and reaction following written and often updated policies.

### Accessing the API

To perform any action in a Kubernetes cluster, you need to access the API and go through three main steps:

- Authentication
- Authorization (ABAC or RBAC)
- Admission Control

These steps are described in more detail in the official documentation about [controlling access to the Kubernetes API](https://kubernetes.io/docs/reference/access-authn-authz/controlling-access/) and illustrated by the diagram below.

![Accessing the API](../../images/accessing-api.png "Accessing the API")

Once a request reaches the API server securely, it will first go through any authentication module that has been configured. The request can be rejected if authentication fails or it gets authenticated and passed to the authorization step.

At the authorization step, the request will be checked against existing policies. It will be authorized if the user has the permissions to perform the requested actions. Then, the requests will go through the last step of admission. In general, admission controllers will check the actual content of the objects being created and validate them before admitting the request.

In addition to these steps, the requests reaching the API server over the network are encrypted using TLS. This needs to be properly configured using SSL certificates. If you use **kubeadm**, this configuration is done for you; otherwise, follow [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) by Kelsey Hightower, or review the [API server configuration options](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/).

### Authentication

There are three main points to remember with authentication in Kubernetes:

- In its straightforward form, authentication is done with certificates, tokens or basic authentication (i.e. username and password).
- Users are not created by the API, but should be managed by an external system.
- System accounts are used by processes to access the API (to learn more read Configure Service Accounts for Pods).

If you want to learn more on how system accounts are used by processes to access the API:

There are two more advanced authentication mechanisms. Webhooks can be used to verify bearer tokens, and connection with an external OpenID provider.

The type of authentication used is defined in the kube-apiserver startup options. Below are four examples of a subset of configuration options that would need to be set depending on what choice of authentication mechanism you choose:

- **--basic-auth-file**
- **--oidc-issuer-url**
- **--token-auth-file**
- **--authorization-webhook-config-file**

One or more Authenticator Modules are used: 

- x509 Client Certs
- static token, bearer or bootstrap token
- static password file
- service account
- OpenID connect tokens

Each is tried until successful, and the order is not guaranteed. Anonymous access can also be enabled, otherwise you will get a 401 response. Users are not created by the API, and should be managed by an external system.

### Authorization

Once a request is authenticated, it needs to be authorized to be able to proceed through the Kubernetes system and perform its intended action.

There are three main authorization modes and two global Deny/Allow settings. The three main modes are:

- ABAC
- RBAC
- Webhook

They can be configured as kube-apiserver startup options:

- **--authorization-mode=ABAC**
- **--authorization-mode=RBAC**
- **--authorization-mode=Webhook**
- **--authorization-mode=AlwaysDeny**
- **--authorization-mode=AlwaysAllow**

The authorization modes implement policies to allow requests. Attributes of the requests are checked against the policies (e.g. user, group, namespace, verb).

#### ABAC

[ABAC](https://kubernetes.io/docs/reference/access-authn-authz/abac/) stands for Attribute Based Access Control. It was the first authorization model in Kubernetes that allowed administrators to implement the right policies. Today, RBAC is becoming the default authorization mode.

Policies are defined in a JSON file and referenced to by a kube-apiserver startup option **--authorization-policy-file=my_policy.json**.

For example, the policy file shown below authorizes user Bob to read pods in the namespace foobar:

```json
{
    "apiVersion": "abac.authorization.kubernetes.io/v1beta1",
    "kind": "Policy",
    "spec": {
        "user": "bob",
        "namespace": "foobar",
        "resource": "pods",
        "readonly": true     
    }
}
```

#### RBAC

[RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) stands for Role Based Access Control.

All resources are modeled API objects in Kubernetes, from Pods to Namespaces. They also belong to API Groups, such as **core** and **apps**. These resources allow operations such as Create, Read, Update, and Delete (CRUD), which we have been working with so far. Operations are called **verbs** inside YAML files. Adding to these basic components, we will add more elements of the API, which can then be managed via RBAC.

Rules are operations which can act upon an API group. Roles are a group of rules which affect, or scope, a single namespace, whereas **ClusterRoles** have a scope of the entire cluster.

Each operation can act upon one of three subjects, which are **User Accounts** which don't exist as API objects, **Service Accounts**, and **Groups** which are known as **clusterrolebinding** when using kubectl.

RBAC is then writing rules to allow or deny operations by users, roles or groups upon resources.

While RBAC can be complex, the basic flow is to create a certificate for a user. As a user is not an API object of Kubernetes, we are requiring outside authentication, such as OpenSSL certificates. After generating the certificate against the cluster certificate authority, we can set that credential for the user using a context.

Roles can then be used to configure an association of **apiGroups**, **resources**, and the **verbs** allowed to them. The user can then be bound to a role limiting what and where they can work in the cluster.

Here is a summary of the RBAC process:

- Determine or create namespace
- Create certificate credentials for user
- Set the credentials for the user to the namespace using a context
- Create a role for the expected task set
- Bind the user to the role
- Verify the user has limited access.

#### Webhook

A Webhook is an HTTP callback, an HTTP POST that occurs when something happens; a simple event-notification via HTTP POST. A web application implementing Webhooks will POST a message to a URL when certain things happen.

To learn more about using the Webhook mode, see the [Webhook Mode](https://kubernetes.io/docs/reference/access-authn-authz/webhook/) section of the Kubernetes Documentation.

### Admission Controller

The last step in letting an API request into Kubernetes is admission control.

Admission controllers are pieces of software that can access the content of the objects being created by the requests. They can modify the content or validate it, and potentially deny the request.

Admission controllers are needed for certain features to work properly. Controllers have been added as Kubernetes matured. Starting with the 1.13.1 release of the **kube-apiserver**, the admission controllers are now compiled into the binary, instead of a list passed during execution. To enable or disable, you can pass the following options, changing out the plugins you want to enable or disable:

- **--enable-admission-plugins=Initializers,NamespaceLifecycle,LimitRanger**
- **--disable-admission-plugins=PodNodeSelector**

The first controller is **Initializers** which will allow the dynamic modification of the API request, providing great flexibility. Each admission controller functionality is explained in the documentation. For example, the **ResourceQuota** controller will ensure that the object created does not violate any of the existing quotas.

### Security Contexts

Pods and containers within pods can be given specific security constraints to limit what processes running in containers can do. For example, the UID of the process, the Linux capabilities, and the filesystem group can be limited.

This security limitation is called a security context. It can be defined for the entire pod or per container, and is represented as additional sections in the resources manifests. The notable difference is that Linux capabilities are set at the container level.

For example, if you want to enforce a policy that containers cannot run their process as the root user, you can add a pod security context like the one below:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  securityContext:
    runAsNonRoot: true
  containers:
  - image: nginx
    name: nginx
```

Then, when you create this Pod, you will see a warning that the container is trying to run as root and that it is not allowed. Hence, the Pod will never run.

### Pod Security Policies

To automate the enforcement of security contexts, you can define PodSecurityPolicies (PSP). A PSP is defined via a standard Kubernetes manifest following the PSP API schema. An example is presented below.

These policies are cluster-level rules that govern what a pod can do, what they can access, what user they run as, etc.

For instance, if you do not want any of the containers in your cluster to run as the root user, you can define a PSP to that effect. You can also prevent containers from being privileged or use the host network namespace, or the host PID namespace.

You can see an example of a PSP below:

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: MustRunAsNonRoot
  fsGroup:
    rule: RunAsAny
```

For Pod Security Policies to be enabled, you need to configure the admission controller of the controller-manager to contain PodSecurityPolicy. These policies make even more sense when coupled with the RBAC configuration in your cluster. This will allow you to finely tune what your users are allowed to run and what capabilities and low level privileges their containers will have.

See the [PSP RBAC example](https://github.com/kubernetes/examples/tree/master/staging/podsecuritypolicy/rbac/) on GitHub for more details.

### Network Security Policies

By default, all pods can reach each other; all ingress and egress traffic is allowed. This has been a high-level networking requirement in Kubernetes. However, network isolation can be configured and traffic to pods can be blocked. In newer versions of Kubernetes, egress traffic can also be blocked. This is done by configuring a **NetworkPolicy**. As all traffic is allowed, you may want to implement a policy that drops all traffic, then, other policies which allow desired ingress and egress traffic.

The **spec** of the policy can narrow down the effect to a particular namespace, which can be handy. Further settings include a **podSelector**, or label, to narrow down which Pods are affected. Further ingress and egress settings declare traffic to and from IP addresses and ports.

Not all network providers support the **NetworkPolicies** kind. A non-exhaustive list of providers with support includes Calico, Romana, Cilium, Kube-router, and WeaveNet.

In previous versions of Kubernetes, there was a requirement to annotate a namespace as part of network isolation, specifically the **net.beta.kubernetes.io/network-policy=value**. Some network plugins may still require this setting.

The use of policies has become stable, noted with the **v1 apiVersion**. The example below narrows down the policy to affect the default namespace.

Only Pods with the label of **role:db** will be affected by this policy, and the policy has both Ingress and Egress settings.

The **ingress** setting includes a *172.17* network, with a smaller range of *172.17.1.0* IPs being excluded from this traffic.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-egress-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
  - namespaceSelector:
      matchLabels:
        project: myproject
  - podSelector:
      matchLabels:
        role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```

These rules change the namespace for the following settings to be labeled **project:myproject**. The affected Pods also would need to match the label **role:frontend**. Finally, TCP traffic on port 6379 would be allowed from these Pods.

The egress rules have the **to** settings, in this case the *10.0.0.0/24* range TCP traffic to port 5978.

The use of empty ingress or egress rules denies all type of traffic for the included Pods, though this is not suggested. Use another dedicated **NetworkPolicy** instead.

Note that there can also be complex **matchExpressions** statements in the spec, but this may change as **NetworkPolicy** matures.

```yaml
podSelector:
  matchExpressions:
    - {key: inns, operator: In, values: ["yes"]}
```

### Default Policy Example

The empty braces will match all Pods not selected by other **NetworkPolicy** and will not allow ingress traffic. Egress traffic would be unaffected by this policy.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

With the potential for complex ingress and egress rules, it may be helpful to create multiple objects which include simple isolation rules and use easy to understand names and labels.

Some network plugins, such as WeaveNet, may require annotation of the Namespace. The following shows the setting of a **DefaultDeny** for the **myns** namespace:

```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: myns
  annotations:
    net.beta.kubernetes.io/network-policy: |
     {
        "ingress": {
          "isolation": "DefaultDeny"
        }
     }
```

### Lab 15.1. Working with TLS

```sh
systemctl status kubelet.service
sudo less /var/lib/kubelet/config.yaml
sudo ls /etc/kubernetes/manifests/
sudo less /etc/kubernetes/manifests/kube-controller-manager.yaml
kubectl -n kube-system get secrets
kubectl -n kube-system get secrets certificate-controller-token-9t9jc -o yaml
kubectl config set-credentials -h
cp $HOME/.kube/config $HOME/cluster-api-config
kubectl config <Tab><Tab>
sudo kubeadm token -h
sudo kubeadm config -h
sudo kubeadm config print init-defaults
```

### Lab 15.2. Authentication and Authorization

```sh
kubectl create ns development
kubectl create ns production
kubectl config get-contexts
sudo useradd -s /bin/bash DevDan
sudo passwd DevDan # lfs458
openssl genrsa -out DevDan.key 2048
touch $HOME/.rnd
openssl req -new -key DevDan.key -out DevDan.csr -subj "/CN=DevDan/O=development"
sudo openssl x509 -req -in DevDan.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out DevDan.crt -days 45
kubectl config set-credentials DevDan --client-certificate=/home/guillermov/DevDan.crt --client-key=/home/guillermov/DevDan.key
diff cluster-api-config .kube/config
kubectl config set-context DevDan-context --cluster=kubernetes --namespace=development --user=DevDan
kubectl --context=DevDan-context get pods # Error, Forbidden
kubectl config get-contexts
diff cluster-api-config .kube/config
vim role-dev.yaml
```

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: development
  name: developer
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["deployments", "replicasets", "pods"]
  verbs: ["list", "get", "watch", "create", "update", "patch", "delete"]
# You can use ["*"] for all verbs
```

```sh
kubectl create -f role-dev.yaml
vim rolebind.yaml
```

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: developer-role-binding
  namespace: development
subjects:
- kind: User
  name: DevDan
  apiGroup: ""
roleRef:
  kind: Role
  name: developer
  apiGroup: ""
```

```sh
kubectl create -f rolebind.yaml
kubectl --context=DevDan-context get pods
kubectl --context=DevDan-context create deployment nginx --image=nginx
kubectl --context=DevDan-context get pods
kubectl --context=DevDan-context delete deploy nginx
vim role-prod.yaml
```

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: production
  name: dev-prod
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["deployments", "replicasets", "pods"]
  verbs: ["get", "list", "watch"] # You can also use ["*"]
```

```sh
vim rolebindprod.yaml
```

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: production-role-binding  #<-- Edit to production
  namespace: production          #<-- Also here
subjects:
- kind: User
  name: DevDan
  apiGroup: ""
roleRef:
  kind: Role
  name: dev-prod                 #<-- Also this
  apiGroup: ""
```

```sh
kubectl create -f role-prod.yaml
kubectl create -f rolebindprod.yaml
kubectl config set-context ProdDan-context --cluster=kubernetes --namespace=production --user=DevDan
kubectl --context=ProdDan-context create deployment nginx --image=nginx
kubectl -n production describe role dev-prod
```

### Lab 15.3. Admission Controllers

```sh
sudo grep admission /etc/kubernetes/manifests/kube-apiserver.yaml
```

## High Availability

### Cluster High Availability

A newer feature of **kubeadm** is the integrated ability to join multiple master nodes with collocated etcd databases. This allows for higher redundancy and fault tolerance. As long as the database services the cluster will continue to run and catch up with kubelet information should the master node go down and be brought back online. 

Three instances are required for etcd to be able to determine quorum if the data is accurate, or if the data is corrupt, the database could become unavailable. Once etcd is able to determine quorum, it will elect a leader and return to functioning as it had before failure. 

One can either collocate the database with control planes or use an external etcd database cluster. The **kubeadm** command makes the collocated deployment easier to use. 

To ensure that workers and other control planes continue to have access, it is a good idea to use a load balancer. The default configuration leverages SSL, so you may need to configure the load balancer as a TCP pass through unless you want the extra work of certificate configuration. As the certificates will be decoded only for particular node names, it is a good idea to use a FQDN instead of an IP address, although there are many possible ways to handle access.

### Collocated Databases

The easiest way to gain higher availability is to use the **kubeadm** command and join at least two more master servers to the cluster. The command is almost the same as a worker join except an additional **--control-plane** flag and a **certificate-key**. The key will probably need to be generated unless the other master nodes are added within two hours of the cluster initialization.

Should a node fail, you would lose both a control plane and a database. As the database is the one object that cannot be rebuilt, this may not be an important issue.

### Non-Collocated Databases

Using an external cluster of etcd allows for less interruption should a node fail. Creating a cluster in this manner requires a lot more equipment to properly spread out services and takes more work to configure. 

The external etcd cluster needs to be configured first. The **kubeadm** command has options to configure this cluster, or other options are available. Once the etcd cluster is running, the certificates need to be manually copied to the intended first control plane node. 

The **kubeadm-config.yaml** file needs to be populated with the etcd set to external, endpoints, and the certificate locations. Once the first control plane is fully initialized, the redundant control planes need to be added one at a time, each fully initialized before the next is added.

### Lab 16.1. High Availability Steps and Lab 16.2. Detailed Steps

Add 3 more instances. One to act as a load balancer, the other two will act as master nodes for quorum.

From **ha-proxy** node:

```sh
sudo apt-get update ; sudo apt-get install -y haproxy vim
sudo vim /etc/haproxy/haproxy.cfg
```

```
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# Default ciphers to use on SSL-enabled listening sockets.
	# For more information, see ciphers(1SSL). This list is from:
	#  https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
	ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS
	ssl-default-bind-options no-sslv3

defaults
	log	global
	mode	tcp
	option	tcplog
	option	dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
	errorfile 400 /etc/haproxy/errors/400.http
	errorfile 403 /etc/haproxy/errors/403.http
	errorfile 408 /etc/haproxy/errors/408.http
	errorfile 500 /etc/haproxy/errors/500.http
	errorfile 502 /etc/haproxy/errors/502.http
	errorfile 503 /etc/haproxy/errors/503.http
	errorfile 504 /etc/haproxy/errors/504.http

frontend proxynode
   bind *:80
   bind *:6443
   stats uri /proxystats
   default_backend k8sServers

backend k8sServers
   balance roundrobin
   server master1  10.128.0.24:6443 check  #<-- Edit with your IP addresses.
#   server master2  10.128.0.30:6443 check
#   server master3  10.128.0.66:6443 check

listen stats
     bind :9999
     mode http
     stats enable
     stats hide-version
     stats uri /stats
```

```sh
sudo systemctl restart haproxy.service
sudo systemctl status haproxy.service
```

From **master** node:

```sh
sudo vim /etc/hosts # comment out the old and add a new k8smaster alias to the IP address of the proxy server
```

Get HAProxy IP and navigate to http://34.66.66.0:9999/stats in a browser. Use `curl ifconfig.io` to get public IP.

From **master2** node:

```sh
sudo -i
apt-get update && apt-get upgrade -y
apt-get install -y vim
apt-get install -y docker.io
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" >> /etc/apt/sources.list.d/kubernetes.list
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
apt-get update
apt-get install -y kubeadm=1.20.1-00 kubelet=1.20.1-00 kubectl=1.20.1-00
apt-mark hold kubelet kubeadm kubectl
exit
```

The same steps for **master3** node.

From **all nodes**:

```sh
sudo vim /etc/hosts # comment out the old and add a new k8smaster alias to the IP address of the proxy server
```

From **master** node:

```sh
sudo kubeadm token create
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/ˆ.* //'
sudo kubeadm init phase upload-certs --upload-certs
```

From **master2** and **master3** nodes:

```sh
sudo kubeadm join k8smaster:6443 --token g92ew8.suac6wd72pzi56sq --discovery-token-ca-cert-hash sha256:b34138b00fafa8ebe124c3bde3c464642962b9d5de6e707995009b4fc269f8ba --control-plane --certificate-key 1a55b5c4df7b0662d7438d67340ae891e8be24c4d1036e7aa69d9eec2df3c1ef
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

From **ha-proxy** node:

```sh
sudo vim /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy.service
```

From **master** node:

```sh
kubectl -n kube-system get pods | grep etcd
kubectl -n kube-system logs -f etcd-master2
kubectl -n kube-system exec -it etcd-master -- /bin/sh
ETCDCTL_API=3 etcdctl -w table --endpoints 10.2.0.2:2379,10.2.0.5:2379,10.2.0.6:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key endpoint status
exit
sudo systemctl stop docker.service
kubectl -n kube-system logs -f etcd-master2
```

From **master2** node:

```sh
kubectl -n kube-system exec -it etcd-master2 -- /bin/sh
ETCDCTL_API=3 etcdctl -w table --endpoints 10.2.0.2:2379,10.2.0.5:2379,10.2.0.6:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key endpoint status
exit
```

From **master** node:

```sh
sudo systemctl start docker.service
kubectl -n kube-system logs -f etcd-master3
```
