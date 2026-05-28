# THMI On-Prem Deployment Guide

This document describes how to provision an Ubuntu server for this repository and prepare it to run the THMI stack with Docker.

# Provisioning and host requerements

### Hardware requirements 
- CPU 8 core
- RAM 12 Gb or more
- HDD 250 Gb

### Software
- Ubuntu Server 24.04 LTS or 26.04 (LTS) or Debian
- A user with `sudo` access
- Internet access for `apt`, Docker packages, and container image pulls
- Access to the THMI configuration files mounted under `/home/thmi/share`
- Enough RAM and disk for the THMI Java services, logs, and Docker images
- Docker Engine
- Docker Compose plugin

### Access 

You need to provide us with an email, so we cant grant you access to the docker registry and the repository with docker compose file and configurations.

## Base Ubuntu Provisioning

Update the host and install baseline packages:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y ca-certificates	curl gnupg lsb-release git	jq unzip net-tools
```

### Create the Deployment User and Directories

Create the service account or use already existing. Later in this example 'ubuntu' is used

```bash
sudo useradd -m -s /bin/bash ubuntu
sudo usermod -aG sudo ubuntu
```

### Clone This Repository

```bash
git clone https://github.com/cybermental/thmi-onprem thmi-cloud
cd thmi-cloud
```

### Install Docker Engine and Compose Plugin

Install Docker from Docker's official Ubuntu repository:

```bash
chmod a+x ./provisioning/install_docker.sh
./provisioning/install_docker.sh
```

Check that docker is running 

```bash
sudo systemctl status docker
```

If docker is not running start the service 

```bash
sudo systemctl start docker
```

Enable Docker and allow the ubuntu user to run it:

```bash
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
```

#### Down bellow is the breakdown of steps executed by the script

Sripts executes steps described in the official docker docs
https://docs.docker.com/engine/install/

# Deploying THMI cloud stack

thmi-onprem repository contains files and configuration for both, edge and cloud. 
When deploying the edge component cloud configuration is unused.

## THMI Cloud

### log into the github registry with provided credentials
```bash 
export CR_PAT=ghp_XXXYYYYZZZ
echo "$CR_PAT" | docker login ghcr.io/cybermental -u username --password-stdin
```

Add google api, maptiler keys and host IP address or the domain name for the fronend

`cloud.yml`, `nginx` service, `environment` section

```yaml
  EXTERNAL_ADDRESS: 192.168.1.10
  GOOGLE_MAPS_API_KEY: "XXXYYYZZZ"
  MAPTILER_API_KEY: "XXXYYYZZZ"
```

`EXTERNAL_ADDRESS` is the ip where you will access the thmi cloud. Also used as an endpoing for the edge mqtt connection.

Run MariaDB, Clickhouse and RabbitMQ containers:

```bash
docker compose -f env.yml up -d
```

MariaDB and Clickhouse should have persistent volumes to prevent data loss on stack restart.
Compose file do not have any redundancy or backup mechanism

Populate database

```bash
docker run --network=thmi-com -it --rm ghcr.io/cybermental/emc/engine-utils:eval
```

Start the THMI stack 
```bash
docker compose -f cloud.yml up -d
```
## THMI Edge

THMI Edge is running as a Docker container. The compose file `edge.yml` includes MariaDB.

Edge should be able to connect to the same RabbitMQ as a Cloud stack.
The config file for Edge can be found there: ./conf/edge/app_config.xml

To run the edge you need only `conf/edge` and `edge.yml` files. We do not recomment keeping cloud confgiration files on the edge server

```bash
git clone --no-checkout https://github.com/cybermental/thmi-onprem thmi-edge
cd thmi-edge
git sparse-checkout init --cone
git sparse-checkout set conf/edge edge.yml
git checkout
docker compose -f edge.yml up -d
``` 

## Edge server configuration

configuration is located at `conf/edge/app_config.xml`

in the `mqtt` and `engine` section you should replace `HOST` with the address of your cloud server.

```xml
<Mqtt
		url="tcp://HOST:1883"
		userName="admin"
		password="admin"
/>
```

```xml
<Engine
		clientId="2ftJLnis7QOqczIczKsBboAieie"
		secretKey="QIfOPPis3o6e2nUVii4rFp97hron03fbPTurCyRThav5cie"
		url="http://HOST:88"
/>
```

## NOTE:
this is only for the evaluation. this is why we are not creating a new server with proper id and credentials. this is made for siplicity of the setup process
 to get thing up  and running as easy as possible.


## Adding project to the edge

https://docs.energymachines.cloud/docs/quick-start/how-to/add-project/

## Networking 

make sure that the edge can communicate with the cloud component 

list of ports

| Service | Port |
|---------|---------|
| THMI web UI |    80     |
| MQTT |    1883     |
| engine| 88 | 

you can check that 

### SSL security

We are not provivng you with the SSL sertificates or any type of encryption.
So the connection security is up to you.
If you want to expose the THMI cloud or the edge componet to the internet you shhould take care of encrypting the traffic 

### monitoring

We are not providing any means of montoring the stack for the evaluation time. you can use any tool that are compatible with docker to do so. 

## Access 

Add external collaborator to the github

generate fine access token

https://docs.github.com/en/repositories/creating-and-managing-repositories/access-to-repositories


