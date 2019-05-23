layout: top-left
# Introduction to: vSphere Integrated Containers

---
# Author
- Eli Ben-Shoshan
- ebs@ufl.edu
- @linkages
- last updated: 2019/05/22

---
# Contents


---
# What is it?

vSphere integrated containers provides a simple to consume docker environment and docker image registry for vSphere deployments

---
# Why?

- quick to deploy
- easy for developers to use
- leverages native vSphere functionality

---
# Architecture

There are 3 parts to the system:
- VIC Appliance
- VIC Container Host - VCH
- Container VMs

---
# VIC appliance

- deploys VCH hosts
- provides a private docker image registry with RBAC based on the concept of Projects
- can be used by developers to deploy containers graphically but this is not the prefered way to deploy containers

---
# VCH

- provides a secure docker REST endpoint
	- uses certificate based authentication
	- all docker REST calls are done over TLS v1.2
- deploys the container VMs
- manages volume creation and mapping to container VMs
- main point of interaction with developers
- Each developer/group/tenant gets a VCH

---
# Container VMs

When you deploy a container like this:
```
docker run -d -p 80:80 --name hi-there nginx
```
you get a stateless VM that starts up your container

Every container get another VM.

---
# Networking

There are 2 ways that a VM can communicate with the outside world in VIC

1) NAT/Bridge
2) Direct

---
# NAT/Bridge

Each container VM gets an interface on a bridge network so that it can privately communicate with other containers managed by the same VCH and with the VCH itself.

In NAT mode, the container ports are forwarded through the VCH like this:

```
Client <--> VCH:80 <--> Container:8080
```
When the container makes outbound requests, they are NAT'ed through the VCH

This is the default method that is used when deploying a container

This is _NOT_ the preffered method for container networking

---
# Direct

Each VCH is assigned a range of IP addresses to use for its container VMs.

When a container is deployed, the next available IP in that range is statically assigned to that container.

The network must be explicitly requested when deploying a container like this:

```
docker run -d -p 80:80 --network public --name hi-there nginx
```

To get a list of networks available to use do this:

```
docker network ls 
```

When a VCH is deployed, the name for this network will be called ```public```. The NAT network will be called ```bridge```.

---
# Demo time!

---
# Three demos

1) Deploy stateless containers
2) Deploy stateful containers
3) Deploy containers using docker-compose

---
# But first...

You will be given the CA and certificates for your VCH in a secure fashion ( TBD )

You then need to setup environment varialbes that docker cli will use:

```
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=/home/eli/src/vic/1.5.2/vic/vch-1.infr.ufl.edu
export DOCKER_HOST=vch-1.infr.ufl.edu:2376
export COMPOSE_TLS_VERSION=TLSv1_2
```

---
# Stateless containers

Deploy containers:

```
for i in 1 2 3; do
	docker run -d -p 80:80 --network public --name hello${i} nginx
done;
```

Get IP addresses:
```
for i in 1 2 3; do
	docker container inspect hello${i} | grep IPAddress | tail -n 1;
done;
```

Stop and remove:
```
for i in 1 2 3; do
    docker container stop hello${i};
    docker container rm hello${i};
done;
```

---
# Stateful containers

Create the volumes:

```
for i in 1 2 3; do
	docker volume create --opt VolumeStore=ds --name hello${i};
done;
```

Putting stuff in a volume is "hard".
First deploy a busybox container with an attached volume:

```
docker run -d -v hello1:/stuff --name truck busybox
```

Then copy over files to that container:

```
docker cp ./local-file truck:/stuff/dest-file
```

---
# Stateful containers

Then delete the busybox container:
```
docker rm truck
```

To test that the files are there run this:
```
docker run -it -v hello1:/stuff --rm busybox /bin/bash
```

To remove the volume:
```
docker volume rm hello1
```

---
# docker-compose

