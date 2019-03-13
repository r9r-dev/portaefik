# Portaefik
portainer + traefik + docker in a nutshell

# Description
This repository contains everything needed to prepare a scalable linux server ready to manage many web applications in a secured environnement.

## Why not Kubernetes ?
Because it's hard to master and not really needed for such small projects and for personal use. It's subject to discuss anyway :)

# Features
 * Reverse proxy exposing only 443 (https)
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
You need a linux machine preferably with Ubuntu 18.04 LTS (because that's what I'm using). Of course, you must be root or have sudo access to it. I am using a [Scaleway Pro X64-15GB for 24,99€/month](https://www.scaleway.com/pricing/#anchor_pro) because it's a very powerfull VPS (6x2Ghz Epyc CPU, 15GB RAM, 200GB SSD) that can run many services at once. Next commands in this readme suppose you are root. If not, you can use `sudo su`.

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

Next, download this repository and change acme.json authorisations to 600 (for traefik to work properly).
```sh
apt install -y git
git clone https://github.com/rlcx/portaefik.git
cd portaefik
chmod 600 acme.json
```

### Traefik configuration
Edit `traefik.toml` and change lines `domain = "altf4.dev"` and `email = "xxx@xx.com"`. Please use a domain you own and where you can use a wildcard `*.yourdomain.tld` **A** entry or a **A** entry for every subdomain you want to redirect to your edge router. For long term usage, it's preferable to create an entry for every subdomain because traefik will receive the request for anything.yourdomain.tld and will return a HTTPS/404 without any certificate (of course, it will not create a certificate for anything.yourdomain.tld).
```sh
nano traefik.toml
```

So ! That configuration tells traefik to:
 * Accept HTTPS/443
 * Watch docker api on /var/run/docker.sock for containers to route to frontend
 * Use network called "traefik" for containers (so it will not route containers you don't want to route)
 * Create a certificate on host rules using Let's Encrypt

### Traefik and Portainer deployment
Edit `docker-compose.yml` and modify the line `"traefik.frontend.auth.basic.users=User:Hash"` to secure your traefik dashboard. To do it, you will need htpasswd.
```sh
apt install -y apache2-utils
htpasswd -nb username password
```
Alternatively, you can generate a password/hash on [htaccesstools.com](http://www.htaccesstools.com/htpasswd-generator/).

In the resulting output, replace any `$` with `$$` and paste it over the `User:Hash` part. On the previous example, the output could be `username:$apr1$W3ovhvF4$juH88W/ijxNVSHpN2S5K./` so the resulting line must be `traefik.frontend.auth.basic.users=username:$$apr1$$W3ovhvF4$$juH88W/ijxNVSHpN2S5K./`

 Start the containers using the following command:
```sh
docker-compose up -d
```

# Access your server
On your nameserver hosting service, add a wildcard **A** entry pointing to your server's public ip. Like `A *.altf4.dev 1.2.3.4 14400`. Preferably, you can create a **A** entry for `portainer.yourdomain.tld` and `traefik.yourdomain.tld`. Now, when you try to access to `https://portainer.yourdomain.tld`, your browser will probably insult you telling you that your connexion is not private. Wait some second (traefik received your request and is asking for a valid certificate to let's encrypt). Refresh your browser until everything is ok.

Take no time to create an admin password for portainer. Use a very strong password. You can also create a new admin user and delete the one called "admin" for increased security.

You are also able to access to https://traefik.yourdomain.tld. Again, please use a strong password and avoid users like `admin`, `root` or any other common usernames.

# How to use ?
So, the only opened port traefik is handling is 443. It's recommended to deny access to any other ports. When you want to create a new webapp, you have to first write a docker-compose file and send it to portainer (in Stacks menu). Alternatively, you can use the App Templates menu or start new containers manually. The choice is yours :)

The important things to remember are:
 * For traefik to route your server, you must connect it to the network called `traefik`.
 * Even if on the traefik network, you can disable the route by adding the label `traefik.enable=false` to the container
 * Do not expose the port on your host ! If your docker-compose use the `port` section, remove it and route it to traefik instead.
 * Use the label traefik.frontend.rule to route a subdomain to your container
 * Use the label traefik.port to route the container port to the frontend rule

## Example
Here is a docker-compose.yml to deploy a simple whoami:
```yaml
version: '2'

services:
  whoami:
    image: emilevauge/whoami
    ports:
      - "80:80"
    restart: unless-stopped
```

This simple compose file expose your server port 80 to the port 80 of the container `emilevauge/whoami`. Your objective is to remove it and just let traefik expose it itself. Bonus, traefik will handle https instead of http and will create a certificate for you flawlessly.

So you have to modify it like this :
```yaml
version: '2'

services:
  whoami:
    image: emilevauge/whoami
    networks:
      - traefik
    labels:
      - traefik.frontend.rule=Host:ping.yourdomain.tld
      - traefik.port=80
      - traefik.backend=whoami
    restart: unless-stopped

networks:
  traefik:
    external:
      name: traefik    
```

The container is not exposing is port to host anymore. Instead, you use labels to handle it. The frontend rule tell traefik on which url to route that container's port. Easy =)

ping.yourdomain.tld => your_server:443 => traefik => container:80

Don't forget to route your subdomain to your server ip on your nameserver hosting service. When you are ready, click on *Deploy the stack*. This is where the magic starts. Go to https://ping.yourdomain.tld : it should work using https with a valid certificate.

With portainer, you can then go back to your stacks and edit the `docker-compose.yml` file. Portainer let you also manage your stack, stopping, killing, restarting any containers with a single click.
