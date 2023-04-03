# Portaefik

portainer + traefik + docker in a nutshell

My blog (french) : (down for now)

# Description

This repository contains everything needed to prepare a scalable linux server ready to manage many web applications in a secured environnement. It uses Portainer to manage services and traefik to route traffic to the right container. It's a very simple and easy to use solution for small projects and personal use. It's also a good way to learn how to use docker and docker-compose.

Please note that this solution is for single server like a VPS or a personal server at home. It relies only on docker and not orchestrators.

## Prerequisites

- A linux server (I am using a VPS at Scaleway with Ubuntu 22.04 LTS)
- A domain name

# Features

With traefik and portainer, you will be able to:

- Reverse proxy exposing only 443 (https)
- Simple configuration, clean, lightweight, forgetable
- Able to manage many domain names and/or subdomains
- Automagically creating validated certificates (with Let's Encrypt)
- Docker administration from a friendly web interface

# Services

## Traefik

Traefik is the front wired to the backs. It can automagically discover your docker containers and expose them to the frontend opened port. It can manage your certificates for every domains flawlessly and without any manual intervention. And if it was not enough, it can load balance your backends if you have many for the same service. So it's future proof if you need a little scaling.

Yup, Traefik is just magic and it uses almost no ressources. Fantastic !

## Portainer

You don't want to manage your containers using command-line (no you don't !). Portainer can do everything dirty for you by letting you manage everything docker related on your server using a web UI. The last time you will use a command line is for installing this server.

# Let's go !

## Run your server

You need a linux machine with any distribution. For examples here, I am using `apt` so you'll need a debian-based distro (I'm using Ubuntu). Of course, you must be root or have sudo access to it. I am using a Scaleway DEV1-L but you can get a testing server for [0,07€/month](https://www.scaleway.com/fr/tarifs/?tags=compute).

Scaleway are using root user by default. So, next commands in this readme suppose you are root. If not, you can use `sudo su -`.

### Optional (but recommended) : Update your system

Update packages to their lastest versions

```sh
apt update && apt dist-upgrade -y
```

Even if it's not mandatory, there is no service running on your server right now so it's a good time to reboot it. After a potentially big update, anything could happen and it's preferable to see now if your server reboot properly.

```sh
shutdown -r now
```

## Install docker

It's that easy!

```sh
curl -sSL https://get.docker.com/ | sh
```

## Install docker compose

You should install docker compose as it's what I will use to configure portainer and traefik containers. Just use apt because we don't mind using the lastest version available.

```sh
apt install docker-compose
```

## Deploy

### Prepare your environnement

We are going to use a single network called `traefik` where all containers that are handling HTTP traffic (using Traefik) will reside in. This will let you run containers on the same virtual network and let traefik now which of your containers have to be publicly available.

```sh
docker network create traefik
```

Next, download this repository and create a folder named `letsencrypt` where traefik will store your certificates.

```sh
apt install -y git
git clone https://github.com/rlcx/portaefik.git
cd portaefik
mkdir letsencrypt
```

### Traefik configuration

Edit `docker-compose.yml` and change lines `domain = "domain.tld"` and `youremail@domain.tld`. Please use a domain you own and where you can use a wildcard `*.yourdomain.tld` **A** entry or a **A** entry for every subdomain you want to redirect to your edge router. For long term usage, it's preferable to create an entry for every subdomain and not using a wildcard because traefik will receive the request for anything.yourdomain.tld and will return a HTTPS/404 without any certificate (of course, it will not create a certificate for anything.yourdomain.tld).

So ! That configuration tells traefik to:

- Accept HTTPS/443
- Watch docker api on /var/run/docker.sock for containers to route to frontend
- Use network called "traefik" for containers (so it will not route containers you don't want to route)
- Create a certificate on host rules using Let's Encrypt

### Traefik and Portainer deployment

Create a folder `/data/portaefik/portainer-data` where portainer will store its data.

Create a file `users.u` to secure your traefik dashboard. Put user and password in it. To do it, you will need htpasswd.

```sh
apt install -y apache2-utils
htpasswd -nb username password
```

Alternatively, you can generate a password/hash on [htaccesstools.com](http://www.htaccesstools.com/htpasswd-generator/).

Paste the resulting ouput `user:hash` into the file.

Start the containers using the following command:

```sh
docker-compose up -d
```

# Access your server

On your nameserver hosting service, add a wildcard **A** entry pointing to your server's public ip. Like `A *.domain.tld 1.2.3.4 14400`. Preferably, you can create a **A** entry for `portainer.domain.tld` and `traefik.domain.tld`. Now, when you try to access to `https://portainer.domain.tld`, everything should be ok. You can check that the certificate is issued by Let's Encrypt by clicking on the lock icon in your browser.

If you are not sure about what you are doing, you can make some tests with Let's Encrypt staging environment. Just uncomment the lines `- "--certificatesresolvers.letsencryptresolver.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory"` and `- "--log.level=DEBUG"` in `docker-compose.yml` and start the containers using `docker-compse up` (without -d).

After checking that the certificates are properly generated, don't forget to delete `acme.json` in the `letsencrypt` folder before restarting in production environment.

Take no time to create an admin password for portainer. Use a very strong password. You can also create a new admin user and delete the one called "admin" for increased security.

You are also able to access to https://traefik.domain.tld. Again, please use a strong password and avoid users like `admin`, `root` or any other common usernames.

# How to use ?

So, the only opened port traefik is handling are 80 and 443. It's recommended to deny access to any other ports. When you want to create a new webapp, you have to first write a docker-compose file and send it to portainer (in Stacks menu). Alternatively, you can use the App Templates menu or start new containers manually. The choice is yours :)

The important things to remember are:

- For traefik to route your server, you must connect it to the network called `traefik`.
- Even if on the traefik network, you can enable or disable the route by adding the label `traefik.enable=false` (or true) to the container
- Do not expose the port on your host ! If your docker-compose use the `port` section, remove it and route it to traefik instead.
- Use the label `traefik.http.routers.[servicename].rule=Host` to route a subdomain to your container
- Use the label `traefik.http.services.[servicename].loadbalancer.server.port` to route the container port to the frontend rule

## Example

Here is a docker-compose.yml to deploy a simple whoami:

```yaml
version: "3"

services:
  whoami:
    image: traefik/whoami
    ports:
      - "80:80"
    restart: unless-stopped
```

This simple compose file expose your server port 80 to the port 80 of the container `traefik/whoami`. Your objective is to remove it and just let traefik expose it itself. Bonus, traefik will handle https instead of http and will create a certificate for you flawlessly.

So you have to modify it like this :

```yaml
version: "3"

services:
  whoami:
    image: traefik/whoami
    networks:
      - traefik
    labels:
      - "traefik.enable=true"
      - "traefik.http.services.whoami.loadbalancer.server.port=80" # facultative (traefik will use the first exposed port by default)
      - "traefik.http.routers.whoami.rule=Host(`whoami.domain.tld`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls.certresolver=letsencryptresolver"
    restart: unless-stopped

networks:
  traefik:
    external:
      name: traefik
```

The container is not exposing is port to host anymore. Instead, you use labels to handle it. The frontend rule tell traefik on which url to route that container's port. Easy =)

ping.yourdomain.tld => your_server:443 => traefik => container:80

Don't forget to route your subdomain to your server ip on your nameserver hosting service. When you are ready, click on _Deploy the stack_. This is where the magic starts. Go to https://ping.yourdomain.tld : it should work using https with a valid certificate.

With portainer, you can then go back to your stacks and edit the `docker-compose.yml` file. Portainer let you also manage your stack, stopping, killing, restarting any containers with a single click.
