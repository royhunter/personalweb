title: How to setup a private docker registry?
date: 2016-01-13 22:53:47
tags: Virtualization
categories: Docker
banner: http://7xoxkz.com1.z0.glb.clouddn.com/docker-registry.PNG
---
A registry is a storage and content delivery system, holding named Docker images, available in different tagged version.
Docker official service supports several types of registry:
1. Docker Hub
2. Docker Trusted Registry
3. Docker Registry

<!--more--> 

Docker Hub is for public pull/push, and it is for free.
Docker Trusted Registry is a nonfree version of registry which similar to docker hub, provide some authentication and security function. But it is a charge version.
Docker registry is provided by a registry image which can be deployed by youself as a container, provide registry function, can be used in a small team or an organization, even a company.

This note will focus on how to setup a private registry by registry:v2 which is supported by docker authority. 

## Start a simple registry server
Previously, docker provide registry v1 for using, but it is deprecated now, so we should download registry v2 from docker hub.

```bash
$docker run -d -p 5000:5000 --restart=always -v /reg:/var/lib/registry --name registry registry:2
```

-p means mapping 5000 port with localhost:5000
When you push a image into registry, it will store the image into /var/lib/registry internal. So, "-v /reg:/var/lib/registry" means mount local storage directory to keep images persistently.


Now a registry is running on your localhost.
Edit /etc/systemd/system/docker.service and add "--insecure-registry ip_address:5000" at the end of the ExecStart line. Then restart the docker service:

```bash
$service docker restart
```

you can do some check, ip_address is ip of your service running machine.

```bash
$curl -v ip_address:5000/v2/
```

Also, you can try push/pull to verify registry's basic function.

```bash
$docker push ip_address:5000/image_name:tag
$docker pull ip_address:5000/image_name:tag
```

## Upgrade registry with more security
Awesome, private registry is running, .
But problem is: anyone with access to the registry can use it. It's all HTTP.And you will have to configure each docker daemon to use the insecure registry. It is very boring.

So we should add some authentication to enhence the security of registry service.

### Setup a self signed certification
First step: we can secure it with a self signed certificate

```bash
$mkdir -p 
$openssl req  -newkey rsa:4096 -nodes -sha256 -keyout certs/domain.key -x509 -days 365 -out certs/domain.crt
```
It will generate two files, one is domain.key, the other is domain.crt.

In server and client, you should both install the certification:

```bash
$mkdir -p /etc/docker/certs.d/ip_address:5000
$cp certs/domain.crt /etc/docker/certs.d/ip_address:5000/ca.crt

$cp certs/domain.crt /usr/local/share/ca-certificates/ca.crt
$update-ca-certificates
```

### Add Username and Password for registry access

```bash
$mkdir auth
$docker run -it --entrypoint htpasswd -v $PWD/auth:/auth -w /auth registry:2 -Bbc /auth/htpasswd username password
```
This command will use htpasswd to create a username and associated password.

### Restart docker service

```bash
$service docker restart
```

### Start registry service with new config
```bash
$docker run -d -p 5000:5000 --restart=always --name registry -v $PWD/certs:/certs -v $PWD/auth:/auth -v /reg:/var/lib/registry -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd -e REGISTRY_AUTH=htpasswd registry:2 
```

### Try login/pull/push
Now everything is ok, you can use follow commands to try the new registry:
```bash
$docker login -uusername -ppassword -emailaddress ip_address:5000
$docker push ip_address:5000/image:tag
$docker pull ip_address:5000/image:tag
$docker logout
```

PS. docker registry does not provide web UI. So you should write script to get images info from registry by youself. 
Below is a example commands in order to get images info and tags list info, data return with json format. 
```bash
$ curl -X GET http://localhost:5000/v2/_catalog
{"repositories":["ubuntu"]}
$ curl -X GET http://localhost:5000/v2/ubuntu/tags/list
{"name":"ubuntu","tags":["latest"]}
```









 