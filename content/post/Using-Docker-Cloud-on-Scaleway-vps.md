---
title: Using Docker Cloud on Scaleway vps
tags:
    - docker
    - vps
date: 2016-05-14 11:54:31
---

### Docker cloud

[Docker Cloud](https://cloud.docker.com) (formerly Tutum) help to deploy containers image on node clusters. Nodes can be provisioned directly from the service (Digital Ocean, Azure, AWS, Packet.net, and IBM SoftLayer). Additionally is possible to use the function _Bring your own node_ (BYON) to add any linux server connected to the internet as node and deploy on it.

I'm using this service to manage a stack (a set of images described by a file similar to docker-compose.yml) composed of a static webiste served by nginx, two api server built with elixir, [nginx-proxy](https://github.com/jwilder/nginx-proxy) (for reverse proxing) and [jrcs/letsencrypt-nginx-proxy-companion](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion) (create/renewal of Let's Encrypt certificates automatically). Dcoker cloud provide with an interface to start/stop containers and scale up the same image to multiple nodes.

BYON requires some manual intervention, installing an agent and usually open a port (2375) in to the server firewall to let docker cloud communicate with the agent - additional ports are required to allow network overlay.

{{< figure src="/images/docker-cloud-byon.png" title="Install the agent" >}}

### Scaleway

[Scaleway](https://www.scaleway.com/) is a cloud provider still in beta that offer the smaller server (VC1S - 2 x86 64bit Cores, 2GB memory - 50GB SSD Disk - 200Mbit/s badnwidth) for the price of 2.99 €/month. You can request an invite to the beta at https://www.scaleway.com/invite/

To open the port requested by the agent to communicate with docker cloud you need to go to security, pick one of the security group and open the necessary port as shown below. A security group is a set of firewall rules that could be applied to multiple servers.

{{< figure src="/images/scaleway-ports.png" title="Open the port on Scaleway " >}}

### Installing the agent

I set up an ubuntu image (14.04) on the server run the command shown in the BYON pop-up on the server, the agent download and install docker and a few service images. After the installation complete it should connect to the Docker Cloud server and update the pop-up with a success message. Taking some times to connect I checked the log of the agent `/var/log/dockercloud/agent.log` and saw the following error.

```text
time="2016-05-13T21:51:46.196955038Z" level=error msg="There are no more loopback devices available."
time="2016-05-13T21:51:46.197040334Z" level=error msg="[graphdriver] prior storage driver \"devicemapper\" failed: loopback mounting failed"
time="2016-05-13T21:51:46.197121360Z" level=fatal msg="Error starting daemon: error initializing graphdriver: loopback mounting failed"
```

To solve this issue is possible is necessary to create some loopback devices, once done the agent start docker and notify Docker Cloud that is ready. Once done is possible to start containers on the newly provided node.

```shell
for i in $(seq 0 255); do
  mknod -m0660 /dev/loop$i b 7 $i
  chown root.disk /dev/loop$i
done
```
