# portaefocker
portainer + traefik + docker in a nutshell

# Description
This repository contains everything needed to prepare a scalable linux server ready to manage many web applications in a secured environnement.

## Why not Kubernetes ?
Because it's hard to master and not really needed for such small projects and for personal use. It's subject to discuss anyway :)

# Features
 * Reverse proxy exposing only 443 and redirect 80 to 443
 * Simple configuration, clean, lightweight, forgetable
 * Able to manage many domain names and/or subdomains
 * Automagically creating validated certificates (with Let's Encrypt)
 * Ready to load-balance frontend with backends if needed (in the future)
 * Docker administration from a friendly web interface

# Services
## Traefik
Traefik is the front wired to the backs. It can automagically discover your docker containers and expose them to the frontend opened port. It can manage your certificates for every domains flawlessly and without any manual intervention. And if it was not enough, it can load balance your backends if you have many for the same service.

Yup, Traefik is just magic and it uses almost no ressources. Fantastic !

## Portainer
You don't want to manage your containers using command-line (no you don't !). Portainer can do everything dirty for you by letting you manage everything docker related on your server using a web UI. The last time you will use a command line is for installing this server.

# Let's go !
## Run your server
You need a linux machine with Ubuntu 18.04 LTS. Of course, you must be root or have sudo access to it. I am using a [Scaleway Pro X64-15GB for 24,99€/month](https://www.scaleway.com/pricing/#anchor_pro) because it's a very powerfull VPS (6x2Ghz Epyc CPU, 15GB RAM, 200GB SSD) that can run many services at once.

### Optional (but recommended) : Update your system
Update packages to their lastest versions
```sh
apt update
apt -y upgrade
```
You should restart your server now using `shutdown -r now` :)

## Install docker
It's that easy !
```sh
curl -sSL https://get.docker.com/ | sh
```

## Install docker compose
You should install docker compose as it's what I will use to configure portainer and traefik containers. First, download the docker compose binary into the `/usr/local/bin` directory with the following curl command:
```sh
apt install docker-compose
```

## Deploy
### Prepare your environnement
We are going to use a single network called `traefik` where all containers that are handling HTTP traffic (using Traefik) will reside in.
```sh
docker network create traefik
```

Next, we download this repository.
```sh
apt install -y git
git clone https://github.com/rlcx/portaefocker.git
cd portaefocker
```

### Traefik configuration
Edit `traefik.toml` and change lines `domain = "altf4.dev"` and `email = "xxx@xx.com"`.
```sh
nano traefik.toml
```

That configuration tells traefik to:
 * Accept HTTP/80 and HTTPS/443 but redirect HTTP to HTTPS
 * Watch docker api on /var/run/docker.sock
 * Use network called "web" for containers
 * Create a certificate on host rules using Let's Encrypt

### Traefik and Portainer deployment
You can review `docker-compose.yml` to suit your needs but you probably have nothing to change. Start the required containers using the following command:
```sh
docker-compose up -d
```
