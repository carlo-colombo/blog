---
title: Set up Mosquitto on Google Cloud Platform
tags: [mqtt, gcp, docker]
---

### Prerequisites

* google cloud account, `gcloud` cli
* `docker`, `docker-compose`, `docker-machine`

### Provision docker on a google compute instance

```bash
  docker-machine create \
    --driver google \
    --google-project YOUR_PROJECT_ID \
    --google-machine-type f1-micro \
    --google-tags mosquitto \
    machine-name
```

    


### Setting up mosquitto

#### Generate password

Mosquitto came with a tool to manage users of the broker, to avoid install it on the workstation is possible to run it from a docker container.

This command create a file and then write to it a user `foo` with password `bar`, the file will be later referenced in `mosquitto.conf`

```bash
  touch pwfile
  docker run -ti \
    -v `pwd`:/data \
    -w `pwd` \
    toke/mosquitto \
    mosquitto_passwd \
      -b /data/pwfile foo bar
```

#### mosquitto.conf
