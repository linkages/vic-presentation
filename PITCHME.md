# Getting started with
## vSphere Integrated Containers
 
---
# Author

@ul[](false)
- Eli Ben-Shoshan
- ebs@ufl.edu
- @linkages
- last updated: 2019/06/06
@ulend

---
## What is it?

vSphere integrated containers provides a simple to consume docker environment and docker image registry for vSphere deployments

---
## Why?

@ul
- quick to deploy
- easy for developers to use
- leverages native vSphere functionality
@ulend

---
## Architecture

There are 3 parts to the system:

@ol
- VIC Appliance
- VIC Container Host - VCH
- Container VMs
@olend

---
@snap[north-west]
## VIC appliance
@snapend

@snap[west span-100]
@ul
- deploys VCH hosts
- provides a private docker image registry with RBAC based on the concept of Projects
- can be used by developers to deploy containers graphically but this is not the prefered way to deploy containers
@ulend
@snapend

---
@snap[north-west]
## VCH
@snapend

@snap[west span-100]
@ul
- provides a secure docker REST endpoint
	- uses certificate based authentication
	- all docker REST calls are done over TLS v1.2
- deploys the container VMs
- manages volume creation and mapping to container VMs
- main point of interaction with developers
- Each developer/group/tenant gets a VCH
@ulend
@snapend

---
@snap[north-west]
## Container VMs
@snapend

When you deploy a container like this:

```bash
docker run -d -p 80:80 --name hi-there nginx
```

you get a stateless VM that starts up your container

Every container is another VM.

---
## Networking

There are 2 ways that a VM can communicate with the outside world in VIC

@ol[](false)
- NAT/Bridge
- Direct
@olend

---
@snap[north-west]
## NAT/Bridge
@snapend

Each container VM gets an interface on a bridge network so that it can privately communicate with other containers managed by the same VCH and with the VCH itself.

In NAT mode, the container ports are forwarded through the VCH like this:

```bash
Client <--> VCH:80 <--> Container:8080
```

---
@snap[north-west]
## Nat/Bridge
@snapend

When the container makes outbound requests, they are NAT'ed through the VCH.

This is the default method that is used when deploying a container.

This is **NOT** the preffered method for container networking.

---
@snap[north-west]
## Direct
@snapend

Each VCH is assigned a range of IP addresses to use for its container VMs.

When a container is deployed, the next available IP in that range is statically assigned to that container.

The network must be explicitly requested when deploying a container like this:

```bash
docker run -d -p 80:80 --network public --name hi-there nginx
```

---
@snap[north-west]
## Direct
@snapend

To get a list of networks available to use do this:

```bash
docker network ls 
```

When a VCH is deployed, the name for this network will be called **public**

The NAT network will be called **bridge**

---
@snap[north-west]
## Limitations
@snapend

@ul
- Can not be used to build containers
- Container startup time is slower than compared to local machine
@ulend

---

# Demo time!

---
## Demos

@ol[](false)
- Deploy stateless containers
- Deploy stateful containers
- Deploy containers using docker-compose
- Build a container and push to VIC registry
@olend

---
## But first...

You will be given the CA and certificates for your VCH in a secure fashion ( TBD )

You then need to setup environment varialbes that docker cli will use:

```bash
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=<path to certs>
export DOCKER_HOST=<FQDN>:2376
export COMPOSE_TLS_VERSION=TLSv1_2
```

---
@snap[north-west]
## Stateless containers
@snapend

Containers by default save any changes to the their filesystems to a layer that is owned by that container.

When that container is removed, all the layers are removed as well.

---
@snap[north-west]
## Stateless containers
@snapend

Deploy containers:

```bash
for i in 1 2 3; do
	docker run -d -p 80:80 --network public --name hello${i} nginx
done;
```

---
@snap[north-west]
## Stateless containers
@snapend

Get IP addresses:
```bash
for i in 1 2 3; do
	docker container inspect hello${i} | grep IPAddress | tail -n 1;
done;
```

---
@snap[north-west]
## Stateless containers
@snapend

Stop and remove:
```bash
for i in 1 2 3; do
    docker container stop hello${i};
    docker container rm hello${i};
done;
```

---
@snap[north-west]
## Stateful containers
@snapend

If you need to keep the work that is created in a container, then you attach a volume to a filesystem path
within the container and have it write to it. This can be used for things like database containers.

That volume is independant of the lifecycle of the container.

---
@snap[north-west]
## Stateful containers
@snapend

Create the volumes:

```bash
	docker volume create --opt VolumeStore=ds --opt Capacity=1G --name hello1;
```

---
@snap[north-west]
## Stateful containers
@snapend

Putting stuff in a volume is *hard*

First deploy a busybox container with an attached volume:

```bash
docker run -d -v hello1:/stuff --name truck busybox
```

Then copy over files to that container:

```bash
docker cp ./local-file truck:/stuff/dest-file
```

---
@snap[north-west]
## Stateful containers
@snapend

Then delete the busybox container:

```bash
docker rm truck
```

---
@snap[north-west]
## Stateful containers
@snapend

To test that the files are there run this:

```bash
docker run -it -v hello1:/stuff --rm busybox /bin/bash
```

---
@snap[north-west]
## Stateful containers
@snapend

To remove the volume:

```bash
docker volume rm hello1
```

---
@snap[north-west]
## docker-compose
@snapend

docker-compose is a tool that:

@ul
- gives you a concise way of defining all the components for a container in a single file
- lifecycle state
- scaling a container
@ulend

---
@snap[north-west]
## docker-compose
@snapend

The first step is to create a docker-compose.yml file

---?include=assets/code/docker-compose.md

---
@snap[north-west]
## docker-compose
@snapend

Then you use docker-compose to start the service:

```
docker-compose up -d
```

---
@snap[north-west]
## docker-compose
@snapend

You can check the state of the containers with:

```
docker-compose ps
```

---
@snap[north-west]
## docker-compose
@snapend

You can stop your containers with:

```
docker-compose stop
```

---
@snap[north-west]
## docker-compose
@snapend

You can destroy your containers with:

```
docker-compose down
```

---
@snap[north-west]
## docker-compose
@snapend

You can also update and restart your containers with:

```
docker-compose pull
docker-compose down
docker-compose up -d
```

**Be careful with this**

If your container has state this will destroy it with the down command

---
@snap[north-west]
## docker-compose
@snapend

You can also scale your container if the docker-compose file is written correctly with this:

```
docker-compose up --scale <service>=<number> -d
```

---
@snap[north-west]
## Build a container
@snapend

VIC does not support building a container. You will need to do that localy and then push the image to VIC.

---
@snap[north-west]
## Build a container
@snapend

To build a container you need a machine with docker installed and a Dockerfile.
We are going to use https://github.com/linkages/docker-powercli as an example.

---
@snap[north-west]
## Build a container
@snapend

From our local machine we will use this command to build the image:

```
docker build -t vic.infr.ufl.edu/es-sandbox/powerclicore
```

---
@snap[north-west]
## Build a container
@snapend

We then login to the registry and push the image:

```
docker login vic.infr.ufl.edu
docker push vic.infr.ufl.edu/es-sandbox/powerclicore
```

---
@snap[north-west]
## Build a container
@snapend

Using a shell with the environment setup for VCH you use this command to pull the image:

```
docker pull vic.infr.ufl.edu/es-sandbox/powerclicore
```

---

