# thmi-atrium-ljungberg

# THMI On-Prem Deployment Guide

This document describes how to provision an Ubuntu server for this repository and prepare it to run the THMI stack with Docker.

## Target Host

### 
###'
Hardware requirements 
- CPU
- RAM
- HDD

### Software
- Ubuntu Server 24.04 LTS or 26.04 (LTS)
- A user with `sudo` access
- Internet access for `apt`, Docker packages, and container image pulls
- Access to the THMI configuration files mounted under `/home/thmi/share`
- Enough RAM and disk for the THMI Java services, logs, and Docker images
- Docker Engine
- Docker Compose plugin

## 1. Base Ubuntu Provisioning

Update the host and install baseline packages:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y ca-certificates	curl gnupg lsb-release git	jq unzip net-tools
```


## 2. Create the Deployment User and Directories

Create the service account or use already existing on used by the deployment volumes:
later in this example 'ubuntu' is used

```bash
sudo useradd -m -s /bin/bash ubuntu
sudo usermod -aG sudo ubuntu
```

## 2. Clone This Repository

```bash
git clone https://github.com/cybermental/thmi-atrium-ljungberg thmi-cloud
cd thmi-cloud
```

## 3. Install Docker Engine and Compose Plugin

Install Docker from Docker's official Ubuntu repository:

```bash
chmod a+x ./provisioning/install_docker.sh
./provisioning/install_docker.sh
```

the script will install docker. it executes following steps

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Check that docker is running 

```bash
sudo systemctl status docker
```

If docker is not running start the service 

```bash
sudo systemctl start docker
```

Enable Docker and allow the thmi user to run it:

```bash
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
```


## 7. Deploy the Stacks


# THMI Application Stack

make sure that the edge can communicate with the cloud component 

## THMI Cloud

### log into the github registry with provided credentials

```bash 
export CR_PAT=ghp_XXXYYYYZZZ
echo "$CR_PAT" | docker login ghcr.io/cybermental -u username --password-stdin
```

Run MariaDB, Clickhouse and RabbitMQ containers:

```bash
docker-compose -f env.yml up -d
```

MariaDB and clickhouse should have persistent volumes to prevent data loss on stack restart.
Compose file do not have any redundancy or backup mechanism

Populate database

```bash
docker run --network=dev -it --rm ghcr.io/cybermental/thmi/engine-utils
```

Start the THMI stack 
```bash
docker compose -f cloud.yml up -d
```
## THMI Edge

```bash
git clone <repo-url>
cd <repo-url>
docker compose -f edge.yml up -d
``` 
