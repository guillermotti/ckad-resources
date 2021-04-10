# Containers Fundamentals LFS253

## Virtualization Fundamentals

### Control Groups (cgroups)

Feature of the Linux kernel which allows the limitation, accounting and isolation of resources used by groups of processes and their subgroups. Cgroups allow the limitation of memory, disk I/O and network usage for a group of processes. Also may set usage quotas and prioritize a process group to receive more CPU or memory than other groups.

Containers benefit from cgroups because they allow system resources to be limited for processes grouped by a container. In addition, the processes of a container are treated and managed as a whole unit.

### Namespaces

Feature of the Linux kernel allowing groups of processes to have limited visibility of the host system resources. May limit the visibility of: 

- cgroups
- hostname
- process IDs
- IPC mechanisms
- network interfaces and routes
- users
- mounted file systems

In the context of containers, namespaces isolate processes from one container to processes runnning in other containers. Same PIDs can coexist on a host system in separate namespaces, while at the global level of the host system they are each assigned different PIDs. Multiple root processes are allowed to run on the same host system, isolated in separate namespaces.

### Unification File System (UnionFS)

Feature found in Linux, FreeBSD and NetBSD kernels which allows the overlay of separate trasnparent file systems to produce an apparent single unified file system. Several file systems, called branches are virtually stacked in a separate state but their contens appear to be merged phisically.

For containers, unionfs allows for changes to be made to the container image at runtime. The unified file system gives impression that the actual container image is being modified, however, these changes are saved onto a real writable file system, part of unionfs, while leaving the base container image file intact.

### Lab 2.1 - Cgroups

```sh
sudo -i
apt update
apt install -y cgroup-tools
lscgroup
cat /proc/1/cgroup
cd /sys/fs/cgroup/freezer/
mkdir mycgroup
cd mycgroup/
ls
cat tasks # Open the second terminal
echo 2213 >> tasks
cat tasks
echo FROZEN > freezer.state
cat freezer.state
echo THAWED > freezer.state
cat freezer.state
```

```sh
sudo -i
ps
date # Don't display anything as its frozen, after thawed will show it
```

### Lab 2.2 - Namespaces

```sh
ls -l /proc/1/ns/
ip netns add namespace1
ip netns add namespace2
ip netns list
ip link add veth1 type veth peer name veth2
ip link set veth1 netns namespace1
ip link set veth2 netns namespace2
ip netns exec namespace1 ip link set dev veth1 up
ip netns exec namespace2 ip link set dev veth2 up
ip netns exec namespace1 ifconfig veth1 192.168.1.1 up
ip netns exec namespace2 ifconfig veth2 192.168.1.2 up
ip netns exec namespace1 ping 192.168.1.2
ip netns exec namespace2 ping 192.168.1.1
ip netns delete namespace1
ip netns delete namespace2
ip netns list
```

### Lab 2.3 - UnionFS

```sh
apt install -y unionfs-fuse
mkdir /root/dir1
touch /root/dir1/f1
touch /root/dir1/f2
mkdir /root/dir2
touch /root/dir2/f3
touch /root/dir2/f4
mkdir /root/union
unionfs /root/dir1/:/root/dir2/ /root/union/
ls /root/union/
```

## Virtualization Mechanisms

### Full Virtualization vs. Operating System-Level Virtualization

Although Containers are not considered to be Virtual Machines (VMs), not even light-weight VMs, their similarities cannot be overlooked. Both VMs and Containers are products of virtualization methods and provide resources isolation for running applications.

### Operating System-Level Virtualization

Kernel’s capability to allow the creation and existence of multiple isolated virtual environments on the same host. Programs running inside a virtual environment are limited to its content and assigned devices. 

### Mechanisms Implementing Operating System-Level Virtualization

Some of the virtualization mechanisms that paved the way for today’s containers, presented in chronological order, and not in order of importance.

#### Chroot

Mechanism implementing OS-level virtualization. It was first introduced on UNIX Version 7 in 1979, then in 1982 it was added to BSD. Chroot operates on a UNIX-like operating system by changing the apparent root directory of a process and its children. This apparent root directory is no longer the real root directory of the operating system, it is a virtual root directory - commonly referred to as a chrooted directory.

#### FreeBSD Jails

Mechanism implementing OS-level virtualization with very little overhead. It was first introduced in 2000 on FreeBSD systems. It allows for the partitioning of a FreeBSD system into many independent systems, called jails. They share the same kernel, but virtualize the system’s files and resources for improved security and administration through clean isolation between services.

#### Solaris Zones

Mechanism implementing OS-level virtualization. It was first introduced in 2004 on Solaris 10 systems. It represent securely isolated Virtual Machines on a single host system. Zones may host single or multiple applications, services, and their children. Each zone on a host system virtualizes its hostname, network, IP address, and it has assigned storage. Zones may have allocated dedicated physical resources such as CPU, memory, and physical network, but they require minimal storage for their configuration data.

#### OpenVZ

Mechanism implementing OS-level virtualization; it was first introduced in 2005 on Linux systems. OpenVZ allows a physical host to run multiple isolated virtual instances - called containers, virtual environments or virtual private servers. OpenVZ containers share the same kernel, and can only run Linux.

#### Linux Containers (LXC)

Mechanism implementing OS-level virtualization; it was first introduced in 2008 on Linux systems. LXC allows multiple isolated systems to run on a single Linux host, using chroot and cgroups, together with namespace isolation features of the Linux kernel to limit resources, set priorities, and isolate processes, the filesystem, network and users from the host operating system.

#### Systemd-nspawn

Mechanism implementing OS-level virtualization; it was first introduced in 2010 on Linux systems. Systemd-nspawn may be used to run a simple script or boot an entire Linux-like operating system in a container. Systemd-nspawn fully isolates containers from each other and from the host system, therefore processes running in a container are not able to communicate with processes from other containers. It fully virtualizes the process tree, filesystem, users, host and domain name.

### Lab 3.1 - Chroot

```sh
sudo -i
apt install -y debootstrap
mkdir /mnt/chroot-ubuntu-xenial
debootstrap xenial /mnt/chroot-ubuntu-xenial/
cat /etc/os-release
chroot /mnt/chroot-ubuntu-xenial/ /bin/bash
cat /etc/os-release
exit
```

### Lab 3.2 - LXC

```sh
sudo apt install -y lxc
cat /etc/subuid
cat /etc/subgid
sudo bash -c 'echo guillermovigilr veth lxcbr0 10 >> /etc/lxc/lxc-usernet'
cat /etc/lxc/lxc-usernet
ls -a ~/ | grep config
mkdir -p ~/.config/lxc
ls -a ~/.config/
cp /etc/lxc/default.conf ~/.config/lxc/default.conf
cat ~/.config/lxc/default.conf
echo lxc.idmap = u 0 231072 65536 >> ~/.config/lxc/default.conf
echo lxc.idmap = g 0 231072 65536 >> ~/.config/lxc/default.conf
cat ~/.config/lxc/default.conf
sudo reboot
lxc-create -t download -n unpriv-cont-user # ubuntu, xenial, amd64
lxc-start -n unpriv-cont-user -d
lxc-ls -f
lxc-info -n unpriv-cont-user
lxc-attach -n unpriv-cont-user
hostname
ps
cat /etc/os-release
exit
lxc-stop -n unpriv-cont-user
lxc-destroy -n unpriv-cont-user
sudo -i
lxc-create -t download -n priv-cont # ubuntu, xenial, amd64
lxc-start -n priv-cont -d
lxc-ls -f
lxc-info -n priv-cont
lxc-attach -n priv-cont
hostname
ps
cat /etc/os-release
exit
lxc-stop -n priv-cont
lxc-destroy -n priv-cont
```

### Lab 3.3 - Systemd-nspawn

```sh
apt install -y systemd-container
debootstrap --arch=amd64 stable ~/DebianContainer
systemd-nspawn -bD ~/DebianContainer
```

```sh
sudo -i
machinectl list
machinectl status DebianContainer
machinectl show DebianContainer
machinectl terminate DebianContainer
machinectl list
```

## Container Standards and Runtimes

### Containers

A container is the product of several OS-level virtualization features of the Linux kernel used in conjunction to build a lightweight isolated environment. A container image ships the application code together with its dependencies all packaged in a box, with a standard method of unboxing the application and the ability to run it on different isolated and secure environments. The isolated and secure execution environment created from it at runtime is a container.

### Standards

#### App Container (appc)

Specification was introduced in 2014 by CoreOS in collaboration with Google and RedHat. One of the container runtimes implementing the appc specification is rkt. The appc specification defines a container image format, how an application is packaged into a container image, a deployment mechanism and a runtime.

- App Container Image (ACI)
- App Container Image Discovery
- App Container Pod
- App Container Executor (ACE)

#### Open Container Initiative (OCI)

Introduced in 2015 by Docker together with other leaders in the container industry. One of the container runtimes implementing the OCI specification is runC.

- Runtime Specification (runtime-spec)
- Image Format Specification (image-spec)

### Container Runtimes

A container runtime is designed to perform some default operations under the hood as a response to user commands. The container runtime extracts the container image content and stores it on an overlay filesystem, that utilizes the Copy-on-Write mechanism for virtual file integrity. When the runtime executes a container, it interacts with the kernel to set resource limits, build isolation layers through virtualization mechanisms like control groups and namespaces in order to run a containerized application as specified by the container image.

#### runc

Basic CLI tool that leverages the libcontainer runtime (initially developed by Docker, then later open sourced), together providing a low level container runtime focused primarily on container execution. runc implements the OCI specification, and it handles the creation and running of OCI containers. Although runc does not include a centralized daemon, it may be integrated with the Linux service manager - systemd.

#### containerd

Adds robustness and portability by supporting several container operations, such as the storage and transfer of container images, executing containers, attaching storage and network to containers.

#### Docker

One of the most robust and complex container development and management platforms in the container industry. Although we list it here as a container runtime, Docker is a lot more than just a simple, or not so simple, container platform. The Docker platform supports a wide range of operations, from container image management to container lifecycle and runtime management.

#### rkt

Developed by CoreOS, rkt is a simpler application container runtime that implements the modern App Container (appc) specification and relies on the Application Container Image (ACI) format. Although the initial focus was on the ACI image format, in recent years rkt adopted support for container images created in different formats, such as the Docker container images.

#### CRI-O

Minimal implementation of the Container Runtime Interface (CRI) to enable the usage of any Open Container Initiative (OCI) compatible runtime with Kubernetes, a popular container orchestrator. As a lightweight alternative to using Docker or rkt as the runtimes for Kubernetes, it supports both GPG signed and unsigned container images. CRI-O supports runc and Kata Containers as the container runtimes but any OCI-conformant runtime can be plugged in instead.

### Lab 4.1 - Install runc

```sh
sudo apt update
sudo apt upgrade -y
sudo apt install -y runc
runc --version
```

### Lab 4.2 - Install containerd

```sh
sudo apt update
sudo apt upgrade -y
sudo apt install -y containerd
containerd --version
sudo systemctl status containerd.service
```

### Lab 4.3 - Install Docker Engine

```sh
sudo apt install -y docker.io
sudo apt remove docker docker-engine docker.io containerd runc
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
sudo docker version
sudo docker run hello-world
sudo systemctl status docker.service
```

### Lab 4.4 - Install CRI-O

```sh
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository ppa:projectatomic/ppa
sudo apt update
sudo apt install -y cri-o-1.15
crio --version
```

### Lab 4.5 - Install rkt

```sh
gpg --keyserver pool.sks-keyservers.net --recv-key 18AD5014C99EF7E3BA5F6CE950BDD3E0FC8A365E
wget https://github.com/rkt/rkt/releases/download/v1.30.0/rkt_1.30.0-1_amd64.deb
wget https://github.com/rkt/rkt/releases/download/v1.30.0/rkt_1.30.0-1_amd64.deb.asc
gpg --verify rkt_1.30.0-1_amd64.deb.asc
sudo dpkg -i rkt_1.30.0-1_amd64.deb
rkt version
```

### Lab 4.6 - Install LXD

```sh
sudo apt update
sudo apt install -y acl autoconf dnsmasq-base git golang libacl1-dev libcap-dev liblxc1 liblxc-dev libtool libudev-dev libuv1-dev make pkg-config rsync
lxd version
```

### Lab 4.7 - Install Podman

```sh
. /etc/os-release
sudo sh -c "echo 'deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/ /' > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list"
wget -nv https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/xUbuntu_${VERSION_ID}/Release.key -O- | sudo apt-key add -
sudo apt update -qq
sudo apt -qq -y install podman
podman --version
```

## Image Operations

### Container Images

Template for a running container and it is created in the form of a tarball with configuration files. The image includes container configuration options and runtime settings to guide the container’s behavior at runtime. Container runtimes load images to run them as containers, therefore at runtime, a container becomes a running instance of an image, and there may be many containers started from the same image.

### Image Registries and Repositories

- Shared storage location, paired with a software solution that supports security features and version control via tags for the images it stores, is called an image registry. A registry offers image protection features through Role Based Access Control (RBAC) policies, digital signatures for authenticity, and possibly vulnerability scans.
- Image management tools and container runtimes support an image caching feature that allows for a downloaded container image to be reused for multiple deployments on a particular host, a feature called container image repository.

### Container Image Operations with runc

The runc container runtime runs containers bundled in the OCI format, based on a JSON configuration file. As a low-level runtime, runc is capable of reading a predefined default JSON file or allows us to manually generate the configuration ourselves. Runc offers a limited set of operations that allow users to build the OCI formatted bundle which includes all necessary files for the creation and running of a container.

### Container Image Operations with Docker

The key component required for the creation of a container image is the Dockerfile. 

### Container Image Operations with rkt

Set of basic rkt image commands allow us to interact with container images to list, remove unwanted images, verify, remove unused images through garbage collection, to debug and inspect images through extract and render commands.

### Container Image Operations with Buildah and Podman

Buildah supports a container image creation from a running container, but it can also create an image from scratch following instructions found in a Dockerfile. Images could be of the generic OCI format or specific Docker format. Also, a new image can be created from a filesystem layer that includes the root filesystem of an updated container. In addition to image creation support, buildah supports the deletion of a container image. Podman is the tool that supports image download and verification, together with image layer and filesystem management. Its command structure closely resembles the docker command.

### Lab 5.1 - Image Operations with Docker

```sh
sudo docker search nginx
sudo docker image pull nginx
sudo docker image pull myregistry.com:5000/alpine:latest # Error
sudo docker image ls --digests
sudo docker login # guillermotti
sudo docker image push lfstudent/alpine:training # Error
sudo docker image push myregistry.com:5000/alpine:training # Error
sudo docker image prune
sudo docker image rm -f alpine:latest nginx:latest
sudo docker image inspect hello-world
```

### Lab 5.2 - Image Operations with rkt

```sh
sudo rkt fetch https://github.com/coreos/etcd/releases/download/v3.1.7/etcd-v3.1.7-linux-amd64.aci --insecure-options=image
sudo rkt fetch --insecure-options=image docker://nginx
sudo rkt image list
sudo rkt image rm coreos.com/etcd:v3.1.7
sudo rkt image list
```

### Lab 5.3 - Image Operations with Podman

```sh
sudo podman search --filter=is-official nginx
sudo podman image pull docker.io/library/nginx
sudo podman image list
sudo podman image inspect nginx
sudo podman image history nginx
sudo podman rmi nginx
sudo podman image prune
sudo podman image prune -a -f
```

## Container Operations

A container is a process running on the host system. This process is started by the container runtime from a container image, and the runtime provides a set of tools to manage the container process. The container is an isolated environment that encapsulates a running application, and it runs based on the configuration defined in the container image.

### Container Operations - runc

We can easily run a container with the runc run command, which creates, starts, and then deletes a stopped container on our behalf - operations that otherwise can be individually invoked. At any time we can list containers and their states with the runc list command.

### Container Operations - Docker

Docker, the most complex container platform, supports a wide range of container operations. Traditionally, there is a set of commands such as docker create, docker start, docker stop, docker restart, docker kill, docker remove that allow a user to alter the state of a container, together with docker ps for a listing of containers, docker logs to display container logs, docker inspect displays detailed information about an object, and docker stats displays resource usage.

### Container Operations - rkt

The primary operation supported by rkt is to run a pod from a container image with the rkt run command, with various supported options to specify the pod’s networking configuration, mount storage volumes, override configuration options, and change default environmental variables.

### Container Operations - CRI-O

CRI-O’s approach is slightly different than the previously discussed runtimes. There is a single command crio, to start the CRI-O daemon. When the daemon is started, the crio command accepts a multitude of configuration options to describe how a container is expected to run. The configuration options describe mount points, runtime (default is runc), how are image volumes handled, logs and metrics, permissions, process count limits, networking namespace management, and possibly TLS encryption.

### Container Operations - Podman

In addition to container image operations, Podman supports operations to manage the entire lifecycle of a container. For someone familiar with Docker, transitioning into the Podman environment will be seamless. Podman’s command structure follows the same commands as Docker. In Podman we find a set of commands that support operations on images, through the podman image command. We also find a set of commands for containers, that is the podman container command. There is a set of commands for pod management, podman pod, and sets for volumes and network management.

### Lab 6.1 - Container Operations with runc

```sh
mkdir -p runc-container/rootfs
sudo docker container export $(sudo docker container create busybox) > busybox.tar
tar -C runc-container/rootfs/ -xf busybox.tar
cd runc-container/rootfs/
ls
cd ..
runc spec
ls
cat config.json
sudo runc run busybox
```

```sh
sudo runc list
sudo runc ps busybox
sudo runc events busybox
sudo runc list
sudo runc pause busybox
sudo runc list
sudo runc resume busybox
sudo runc list
sudo runc state busybox
sudo runc delete -f busybox
sudo runc list
```

### Lab 6.2 - Container Operations with Docker

```sh
sudo docker image pull alpine # DockerHub
sudo docker image pull <private_registry>:<port>/image # Private registry
sudo docker image ls
sudo docker container create -it alpine sh
sudo docker container start d61
sudo docker container ls
sudo docker container run -it --name myalpine alpine sh
Ctrl p + Ctrl q
sudo docker container ls
sudo docker container attach myalpine
sudo docker container run -d alpine /bin/sh -c 'while [ 1 ];do echo "hello world from container"; sleep 1; done'
sudo docker container logs 08e
sudo docker container stop 08e0cf2a685f
sudo docker container ls -a
sudo docker container start 08e0cf2a685f
sudo docker container ls -a
sudo docker container restart 08e0cf2a685f
sudo docker container pause 08e0cf2a685f
sudo docker container ls -a
sudo docker container unpause 08e0cf2a685f
sudo docker container ls -a
sudo docker container rename trusting_driscoll hello_world_loop
sudo docker container ls -a
sudo docker container stop hello_world_loop
sudo docker container rm hello_world_loop
sudo docker container ls -a
sudo docker container rm -f myalpine
sudo docker container ls -a
sudo docker container run --rm --name auto_rm alpine ping -c3 google.com
sudo docker container ls -a
sudo docker container run -h alpine-host -it --rm alpine sh
hostname
exit
sudo docker container run -it -w /tmp/mypath --rm alpine sh
pwd
exit
sudo docker container run -it --env "WEB_HOST=172.168.1.1" --rm alpine sh
env
exit
sudo docker container run -it --rm alpine sh
ulimit -a
exit
sudo docker container run -it --ulimit nproc=10 --rm alpine sh
ulimit -a
exit
sudo docker container ls
sudo docker container inspect d61bfbc3d1d4
sudo docker container run -d --name cpu-set --cpuset-cpus="0" alpine top
sudo docker container inspect cpu-set | grep -i cpuset
sudo docker container rm -f cpu-set
sudo docker container run -d --name memory --memory "200m" alpine top
sudo docker container inspect memory | grep -i mem
sudo docker container rm -f memory
sudo docker container exec frosty_burnell ip a
sudo docker container run -d --restart=always --name web-always nginx
sudo docker container run -d --restart=on-failure:3 --name web-on-failure nginx
echo Welcome to Container Fundamentals! > host-file
sudo docker container cp host-file web-on-failure:/usr/share/nginx/html/index.html
sudo docker container inspect --format=’{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}’ web-on-failure
curl 172.17.0.4
sudo docker container run -d --label env=dev nginx
sudo docker container ls
sudo docker container ls --filter label=env=dev
sudo docker container ls -q
sudo docker container rm -f `sudo docker container ls -q`
sudo docker container run -it nginx sh
exit
sudo docker container run -it --net=host alpine sh
ifconfig ens4:0 192.168.2.1 up
exit
sudo docker container run -it --net=host --privileged alpine sh
ifconfig ens4:0 192.168.2.1 up
ip a
exit
sudo docker container ls
sudo docker container ls -a
sudo docker container prune
sudo docker container ls -a
sudo docker images
sudo docker container prune -a
sudo docker images
```

### Lab 6.3 - Container Operations with rkt

```sh
sudo rkt fetch https://github.com/coreos/etcd/releases/download/v3.1.7/etcd-v3.1.7-linux-amd64.aci --insecure-options=image
sudo rkt fetch --insecure-options=image docker://nginx
sudo rkt image list
sudo rkt run coreos.com/etcd:v3.1.7
```

```sh
sudo rkt run registry-1.docker.io/library/nginx:latest
```

```sh
sudo rkt list
sudo rkt list --full
sudo rkt status 31733fb7
sudo rkt status d8a0a5a2
sudo rkt fetch --insecure-options=image docker://busybox:latest
sudo rkt run docker://nginx registry-1.docker.io/library/busybox:latest --name=bzbx
```

```sh
sudo rkt list
sudo rkt enter --app=nginx bf2b8daa sh
ls /usr/share/nginx/html
exit
sudo rkt stop bf2b8daa
sudo rkt rm bf2b8daa
sudo rkt gc --grace-period=5m0s
sudo rkt cat-manifest 31733fb7
```

### Lab 6.4 - Container Operations with Podman

```sh
sudo podman search --filter=is-official nginx
sudo podman image pull docker.io/library/nginx
sudo podman image list
sudo podman container create nginx
sudo podman container list -a
sudo podman container start magical_lewin
sudo podman container list
sudo podman container restart magical_lewin
sudo podman container list
sudo podman container stop magical_lewin
sudo podman container rm magical_lewin
sudo podman container list -a
sudo podman container prune -f
sudo podman container create nginx
sudo podman container start eager_easley
sudo podman container list -a
sudo podman container top eager_easley
sudo podman container exec -ti eager_easley /bin/sh
ls /usr/share/nginx/html
exit
echo Welcome to Container Fundamentals! > host-file
sudo podman cp host-file eager_easley:/usr/share/nginx/html/index.html
sudo podman container exec -ti eager_easley /bin/sh
cat /usr/share/nginx/html/index.html
exit
sudo podman container inspect eager_easley
```

## Building Container Images

### Create a Container Image from Scratch

Creating a container image from scratch implies that all components of the image are specified manually or through scripts. The approaches vary when it comes to building images from scratch. Some tools require certain steps to be performed by the user prior to specific instruction can be issued by the runtime while other tools support all aspects of the container image build. There are several tools that manage various aspects of the image build process from scratch, such as runc, Docker, Buildah, Podman, Distrobuilder, Goaci, and Acbuild.

### Create an Image from a Running Container

This container image creation method assumes there is a running container that can be used as a source to create a new image out of it. The method aims to build a new container image based on the root filesystem of an existing running container. The process requires an export of the running container’s root filesystem into an archive, typically a tarball. The following steps extract the root filesystem from the archive and through an import feature, the root filesystem is added into the build of the new container image.

### Convert a Container Image

The image conversion method is no longer such a popular topic with users. This method assumes the existence of Docker or OCI images, and through conversion tools allows users to build ACI formatted images through an image conversion process. In general, as of late, ACI is not seeing too much interest from users, and as a result, many of the tools supporting ACI compatible runtimes, images and containers are slowly being retired. It is no surprise then that although still available, the tools supporting the image conversion method have stopped receiving updates. Such image conversion tools are Docker2aci , and Oci2aci.

### Dockerfile Instruction Formats

Dockerfile accepts instructions in two distinct formats.

- Shell Form: **CMD echo "We are learning about Dockerfile"** -> **/bin/sh -c echo "We are learning about Dockerfile"**
- Exec Form: **CMD ["/bin/echo", "We are learning about Dockerfile"]**

### Dockerfile Instruction

Instructions: https://docs.docker.com/engine/reference/builder/

#### Build Time Instructions

- FROM: **FROM <image>** -> **FROM fedora**
- ARG: **ARG <name>[=<default value>]** -> `docker build --build-arg BUILD_NO=v2`
- RUN: **RUN ["executable", "param1", "param2"]** -> **RUN apt-get dist-upgrade -y**
- LABEL: **LABEL <key1>=<value1> <key2>=<value2> <key3>=<value3> ...** -> **LABEL version="1.0" ENV="dev"**
- EXPOSE: **EXPOSE <port> [<port>...]** -> **EXPOSE 80 443**
- COPY: **COPY <src>... <dest>** -> **COPY ["<src>",... "<dest>"]** -> **COPY run.sh /code**
- ADD: **ADD <src>... <dest>** -> **ADD ["<src>",... "<dest>"]** -> **ADD https://raw.githubusercontent.com/lfstudent/rsvpapp/master/static/lfstudent.png /logo.png**
- WORKDIR: **WORKDIR /path/to/workdir** -> **RUN mkdir /code COPY run.sh /code WORKDIR /code CMD run.sh**
- ENV: **ENV <key> <value>** -> **ENV <key>=<value>** -> **ENV WEB 192.168.2.3**
- VOLUME: **VOLUME /data**
- USER: **USER <name>/<uid>** -> **USER jenkins**
- ONBUILD: **ONBUILD [INSTRUCTION]** -> **ONBUILD RUN pip install docker --upgrade**
- STOPSIGNAL: **STOPSIGNAL <signal>**
- HEALTHCHECK: **HEALTHCHECK [OPTIONS] CMD command** -> **HEALTHCHECK NONE**
- SHELL: **SHELL ["/bin/bash", "-c" ]**

#### Run Time Instructions

- CMD: **CMD ["nginx", "-g", "daemon off;"]**
- ENTRYPOINT: **ENTRYPOINT ["executable", "param1", "param2"]**

### Exclude Files and Directories from Build with .dockerignore

During an image build, the Docker client zips the referenced context folder and sends it to the Docker Host. If we want to exclude some files and directories during the ​ docker image build​ process, then we should list them in the .dockerignore​ file in the referenced folder.

### Lab 7.1 - Building Docker Images

```sh
sudo docker container run -ti --name myalpine alpine sh
date > /data
cat /data
<Ctrl p + Ctrl q>
sudo docker container ls
sudo docker container diff myalpine
sudo docker container commit myalpine lfstudent/alpine:training
sudo docker image ls
sudo docker container run -ti lfstudent/alpine:training
cat /data
<Ctrl p + Ctrl q>
sudo docker container ls
sudo docker container export b4dcee33d877 > lfstudent_alpine.tar
ls lfstudent_alpine.tar
sudo docker image import lfstudent_alpine.tar lfstudent/alpine:latest
sudo docker image ls
sudo docker container run -ti lfstudent/alpine:latest sh
<Ctrl p + Ctrl q>
sudo docker container ls
sudo docker login
sudo docker image import lfstudent_alpine.tar guillermotti/lfstudent:latest
sudo docker image push guillermotti/lfstudent:latest
sudo docker image push myregistry.com:5000/alpine:training # Error
sudo docker image prune
sudo docker image rm -f alpine:latest nginx:latest
```

### Lab 7.2 - Building a Docker Image with Dockerfile

Dockerfile example:

```Dockerfile
FROM debian:buster-slim

LABEL maintainer="NGINX Docker Maintainers <docker-maint@nginx.com>"

ENV NGINX_VERSION   1.17.9
ENV NJS_VERSION     0.3.9
ENV PKG_RELEASE     1~buster

RUN set -x \
# create nginx user/group first, to be consistent throughout docker variants    
    && addgroup --system --gid 101 nginx \    
    && adduser --system --disabled-login --ingroup nginx --no-create-home --home /nonexistent --gecos "nginx user" --shell /bin/false --uid 101 nginx \    
    && apt-get update \    
    && apt-get install --no-install-recommends --no-install-suggests -y gnupg1

# forward request and error logs to docker log collector
RUN ln -sf /dev/stdout /var/log/nginx/access.log \    
    && ln -sf /dev/stderr /var/log/nginx/error.log
    
EXPOSE 80

STOPSIGNAL SIGTERM

CMD ["nginx", "-g", "daemon off;"]
```

```sh
mkdir myapp
cd myapp/
vim Dockerfile # FROM alpine RUN date > data
cat Dockerfile
sudo docker image build -t lfstudent/alpine:dockerfile .
sudo docker image ls
cd
pwd
sudo docker image build -t lfstudent/alpine:dockerfile myapp
cd myapp
time sudo docker image build -t lfstudent/nginx:dockerfile .
time sudo docker image build -t lfstudent/nginx:cached .
echo "EXPOSE 80 443" >> Dockerfile
time sudo docker image build -t lfstudent/nginx:export .
time sudo docker image build --no-cache -t lfstudent/nginx:export .
sudo docker image ls
sudo docker image ls --quiet --filter=dangling=true
sudo docker image ls --quiet --filter=dangling=true | xargs --no-run-if-empty sudo docker image rm
sudo docker image ls
```

## Container Networking

### Standards and Specifications

A typical computing machine, whether a physical host system or a Virtual Machine, is connected to at least one network in order for the system and any running applications to become part of the enterprise’s software ecosystem. The network allows systems to connect with each other, allows applications to communicate with each other, allows for data to be transmitted not only between neighboring systems but also to systems and applications running halfway across the world. In addition, a connected system can be managed remotely, and its resources and applications performance may be monitored. Networking, however, is not at random. Instead, network connectivity and network traffic are governed by networking standards and principles, topologies, protocols, policies, rules, etc.

Container networking is guided by sets of standards that specify how a container may join one or multiple networks simultaneously. There are two container networking standards we need to be concerned about: the Container Network Model (CNM) and the Container Network Interface (CNI). Both networking models present pros and cons in terms of adoption and level of support.

### The Container Network Model (CNM)

The Container Network Model (CNM) is the container network specification introduced by Docker and implemented by the libnetwork project. Companies and other projects such as Cisco, Calico, Weave, Kuryr, VMware, and Open Virtual Networking (OVN) adopted the CNM.

### The Container Network Interface (CNI)

The Container Network Interface (CNI), a Cloud Native Computing Foundation Incubator project, is the container network specification introduced by CoreOS, adopted by projects such as Mesos, Amazon ECS, Kubernetes, Cloud Foundry, OpenShift, rkt, and supported by projects such as Calico, Cilium, Romana, CNI-Genie, VMware NSX, and Weave. It links container platforms such as Kubernetes and Cloud Foundry to multiple distinct container network implementations.

### CNI Plugins

CNI supports several plugin groups, such as the main group focused on creating new interfaces, IPAM focused on IP address allocation, and finally, the meta group for 3rd party plugin support.

- Main: bridge, ipvlan, loopback, macvlan, ptp, vlan, host-device
- IPAM: dhcp, host-local, static
- Meta: flannel, sysctl tuning, portmap, bandwidth, sbr, firewall

### Docker Networking Drivers

Although many network drivers are available and in use today, typically, a few drivers are used in conjunction in order to implement a solution to the networking requirements of containerized applications.

- Bridge: The default network type that Docker containers attach to is the bridge network, implemented by the bridge driver.
- Host: The host network driver option, as opposed to the bridge, eliminates the network isolation between the container and the host system by allowing the container to directly access the host network.
- Overlay: It is the network type that spans multiple hosts, typically part of a cluster, allowing the containers' traffic to be routed between hosts as containers or services from one host attempt to talk to others running on another host in the cluster.
- Macvlan: Allows a user to change the appearance of a container on the physical network. A container may appear as a physical device with its own MAC address on the network
- None: Disables the networking of a container while allowing the very same container to use a custom third-party network driver, if needed, to implement its networking requirements.
- Network Plugins: Expand the capabilities of Docker through third-party network drivers that integrate Docker with specialized network stacks

### Container Networking with rkt

Rkt networking supports the Container Network Interface (CNI), which enables containers to connect to a network when created and then remove resources upon containers’ deletion.

- Default contained mode: The contained network mode implies that a rkt pod runs in a separate network namespace with the network aided by the Container Network Interface (CNI) and CNI plugins.
    - Built-in networks
    - No networking (loopback only)
    - Additional networks
    - Other plugins
- Host mode: The microservice running in the rkt pod inherits the network namespace of the process that invoked rkt, which could be directly from the host system. When rkt is invoked by the host system, the microservice from the rkt pod shares the host system network stack. While in host mode, no isolation is available between the host and the pod networks, the pod having access to the host’s IP address, routes, and iptables.

### Container Networking with CRI-O

CRI-O, the Kubernetes container runtime, has its networking designed to work with Kubernetes pods. Pod networking is set up through the Container Network Interface (CNI). The CNI enables containers to connect to a network when created and then remove resources upon containers’ deletion. It also makes it possible for any network plugin (also called CNI plugin) to work with CRI-O. CNI plugins such as Flannel, Weave, OpenShift-SDN have been tested and performed successfully with CRI-O.

When the CRI-O daemon is started by the crio command, there are options to specify the host network IP and to manage the network namespace.

### Lab 8.1 - Docker Networking

```sh
sudo -i
docker network ls
docker network create -d bridge --attachable mynet
docker network ls
docker network inspect mynet
docker network inspect mynet |grep Name|tail -n +2|cut -d':' -f2|tr -d ' ,"'
docker network create testnet
docker network ls
docker network rm testnet
docker network ls
docker network prune
docker network create -d bridge --attachable mynet
docker container run -d --name web nginx
docker container ls
docker container inspect web | more
ip a
docker container run -d --name web1 -p 80:80 nginx
docker container ls
curl 10.154.0.2
docker container run -d --name web2 -p 80:80 nginx # Error, port is already alocated
docker container run -d --name web3 -p 8080:80 nginx
docker container ls
curl 10.154.0.2:8080
docker container run -d --name web4 -P nginx
docker container ls
curl 10.154.0.2:49153
docker container run -it --network=none alpine sh
ip a
exit
docker container run -it --network=host alpine sh
ip a
exit
ip a
docker container exec -it web sh
which ip
apt update
apt install -y iproute2
which ip
ip a
^p^q # Ctrl p + Ctrl q
docker container run -it --network=container:web alpine sh
^p^q
docker container run -it --network=mynet alpine sh
ip a
exit
docker container run -it --name alpine alpine sh
^p^q
docker network connect mynet alpine
docker container exec -it alpine sh
ip a
exit
docker container inspect alpine
docker network disconnect mynet alpine
docker container inspect alpine
```

### Lab 8.2 - Rkt Networking

```sh
rkt run --interactive --net=none quay.io/coreos/alpine-sh:latest
ip a
exit
rkt run --interactive --net=host quay.io/coreos/alpine-sh:latest
ip a
exit
systemd-run --slice=machine rkt run --insecure-options=image docker://nginx
rkt list
rkt status f20ade66
systemd-run --slice=machine rkt run --port=80-tcp:8080 --insecure-options=image docker://nginx
rkt list
curl 10.154.0.2:8080
mkdir -p /etc/rkt/net.d
vim /etc/rkt/net.d/10-containers.conf # {"name": "containers","type": "bridge","bridge": "rktbr1","ipam":{"type": "host-local","subnet": "172.19.0.0/16"}}
rkt run --interactive --net=containers quay.io/coreos/alpine-sh
ip a
exit
```

## Container Storage

### Manage Storage with Docker

The complexity of Docker as a container management framework is also reflected in its methods of handling storage for containerized applications. We will explore various storage concepts, access modes, storage drivers and volume types as we learn how Docker manages storage and volumes for its containers.

### UnionFS with Copy-on-Write

UnionFS is used by Docker to overlay a base container image with storage layers, such as ephemeral storage layer, custom storage layer, and config layer at the time a new container is created. The ephemeral storage is reserved for the container’s Input/Output (I/O) operations and it is not recommended to be used for persistent data; instead, a volume should be mounted on the container to provide persistent storage not managed by UnionFS.

### Docker Storage Drivers

Typically, not a lot of data needs to be written to the container’s writable layer. However, when such a requirement presents itself, then a storage driver needs to be used to control how the container images and the running containers are stored and managed on the host system.

While Docker supports a variety of storage drivers, each driver may be limited by its supported backing filesystems. The overlay, overlay2 and aufs drivers are supported by xfs and ext4 filesystems, devicemapper driver is backed by direct-lvm, but vfs is supported by any filesystem. Despite the implementation and feature differences between storage drivers, they all use the CoW strategy for stacked storage layers.

### Storage and Docker Volumes

Applications and containers make use of two different types of data - ephemeral and persistent; therefore Docker takes separate approaches when managing ephemeral vs. persistent data.

- Ephemeral data is data typically needed for a very short period of time, and there is no damage to the enterprise when that data is lost once the container is terminated and deleted. Ephemeral data is stored on ephemeral storage, which is specified and defined within the scope of the container, and; the ephemeral storage also gets deleted together with the container. For ephemeral data management, Docker provides a bind mount mechanism tied in with the host’s directory structure. The bind mount references a directory of the host system created on-demand if not existing when the container is created. One of the downsides of bind mounts is that they cannot be directly managed from the Docker CLI.
- Persistent data, however, is critical data that has to be stored in a location that provides data resilience, to then later be managed by an aggregator or other applications designed to process and manage critical data. Persistent data is recommended to be stored on external persistent volume mounts managed from the Docker CLI for easy backup and migration, flexibility, portability, sharing, and extended functionality. Persistent volumes are not managed by UnionFS, are not specified and defined only within the scope of a container, and do not get deleted together with the container.

### Manage Storage with rkt

Rkt supports the mounting of host volumes and empty volumes to each application running inside a Pod, with or without mount points, as specified in the ACI Pod manifest.

The host volume type allows for a host directory to be mounted by all applications in the Pod, where an optional parameter may specify whether the volume is read-only or not. The host volume type is specific for the storage of persistent data, which is considered critical and should outlive the Pod and the applications running in the Pod.

For non-persistent data that only needs to be shared among the applications of a Pod, the empty volume may be used instead. Since the volume is created on the fly, the user may specify the access mode, UID and GID of the empty volume.

### Manage Storage with CRI-O

CRI-O also uses the Copy-on-Write (CoW) strategy in conjunction with its containers/storage library when unpacking a downloaded container image into the root filesystem of a container. CRI-O uses the containers/storage library to manage storage layers and to create the root filesystems for OCI-compliant containers running in a Kubernetes pod. It implements a variety of storage drivers, such as overlayfs, devicemapper, AUFS and btrfs, with overlayfs being the default storage driver.

### Lab 9.1 - Manage Storage with Docker

```sh
sudo -i
docker container run -i -t --mount target=/data --name cvol alpine sh
cd /data/
ls
touch file1
ls
exit
docker container run -i -t -v /data --name cvol2 alpine sh # The same than previous command
exit
docker container run -i -t --mount source=mountvol,target=/data --name cmountvol alpine sh
cd /data/
ls
touch mount-file1
ls
exit
docker container run -i -t -v mountvol:/data --name cmountvol2 alpine sh
exit
docker ps -a
docker container inspect cvol | more
ls /var/lib/docker/volumes/b999dd64b0b46c0892d9cbd9241f6fe849d16d0dec2268ef87e2e53eae19ca7a/_data
docker volume ls
docker volume create --name myvol
docker volume ls
docker container run -i -t --mount source=myvol,target=/data --name cmyvol alpine sh
cd /data/
echo "Docker volumes" > learning.txt
ls
cat learning.txt
^p^q
cat /var/lib/docker/volumes/myvol/_data/learning.txt
docker container rm -f -v cvol
docker volume ls
docker volume rm myvol
docker container ls
docker container rm -f cmyvol
docker volume rm myvol
docker container ls
docker volume ls
mkdir /mnt/shared
docker container run -i -t --mount type=bind,source=/mnt/shared,target=/data --name csharedvol alpine sh
cd /data/
ls
touch bind-file
^p^q
cd /mnt/shared/
ls
echo "text from host" > bind-file
docker attach csharedvol
ls
cat bind-file
exit
mkdir /tmp/ro
echo "host file" > /tmp/ro/host-ro-file
docker container run -i -t --mount type=bind,source=/tmp/ro,target=/data,readonly --name crovol alpine sh
cd /data/
ls
touch container-file
ls
cat host-ro-file
echo "hello from container" >> host-ro-file
ls
cat host-ro-file
exit
docker system df
docker volume prune
docker system prune -a
```

### Lab 9.2 - Manage Storage with rkt

```sh
mkdir /tmp/rktvol
touch /tmp/rktvol/host-file
rkt run --interactive --insecure-options=image --volume=rkthostvol,kind=host,source=/tmp/rktvol --mount volume=rkthostvol,target=/data/app1 docker://nginx --exec /bin/sh
ls
cd data
ls
cd app1
ls
echo pod data > host-file
cat host-file
touch pod-file
exit
cat /tmp/rktvol/host-file
ls /tmp/rktvol/
mkdir /tmp/rktrovol
echo host read-only data > /tmp/rktrovol/host-file-ro
rkt run --interactive --insecure-options=image --volume=rkthostrovol,kind=host,source=/tmp/rktrovol,readOnly=true --mount volume=rkthostrovol,target=/data/app2 docker://nginx --exec /bin/sh
ls
cd data
ls
cd app2
ls
echo pod data >> host-file-ro
cat host-file-ro
touch pod-file
ls
exit
cat /tmp/rktrovol/host-file-ro
ls /tmp/rktrovol/
```

View my verified achievement from The Linux Foundation. What a Hackerman! FSociety is trying to hire me.
