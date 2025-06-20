# About

This project contains guides you in:

* Creating AWS EC2 instance

* (Docker-) Installation of Keycloak on an AWS EC2 instance

* Setup Nginx with TLS

* Configuration of Users within Keycloak

* SSO Integration of Keycloak within a Cumulocity Tenant

# Instructions

**VM Setup**

* get yourself an AWS EC2 VM. I chose t3.micro instance (2 vCPU, 1 GiB memory) with 8 GiB storage runnung Ubuntu 24.04 LTS. 

* Toggle "Allow SSH traffic from 0.0.0.0" and "Allow HTTPS traffic from 0.0.0.0"

* In order to have a static IP for your VM, create an elastic IP and assign it to your VM instance.

**Keycloak installation**

* Login to your VM and do `sudo apt update && sudo apt upgrade` 

* Now install docker: https://docs.docker.com/engine/install/ubuntu/

* Configure docker to run without sudo: https://docs.docker.com/engine/install/linux-postinstall/

* Create new folder `~/keycloak` and copy the `docker-compose.yml` file in the directory. **Adapt usernames and passwords**.

* Go into directory and start it via `docker compose up -d`

**Nginx and TLS setup** 

First, create an SSL certificate for your domain. I'm having the domain `*.kbutz.org` and followed this guide to create certificate via Let's Encrypt: https://www.digitalocean.com/community/tutorials/how-to-create-let-s-encrypt-wildcard-certificates-with-certbot

I've placed the certificate in `/etc/letsencrypt/live/keycloak.kbutz.org/fullchain.pem` and `/etc/letsencrypt/live/keycloak.kbutz.org/privkey.pem`. 

Now do below steps one-by-one:

```
sudo apt install nginx

# configure ufw/firewall
sudo ufw allow "Nginx Full"
sudo ufw allow ssh
sudo ufw enable
sudo ufw reload
# should show ufw status = active and Nginx + SSH being allowed from anywhere
sudo ufw status 

sudo unlink /etc/nginx/sites-enabled/default

# Copy file from this repository in /etc/nginx...
# Adapt the 'ssl_certificate' and 'ssl_certificate_key' to proper path before!
sudo cp nginx.conf /etc/nginx/sites-available/keycloak.kbutz.conf

sudo ln -s /etc/nginx/sites-available/keycloak.kbutz.conf /etc/nginx/sites-enabled/keycloak.kbutz.conf
```

You can use `sudo nginx -t` to let nginx check if your configuration is valid. If so, restart the service via `sudo systemctl restart nginx`. Make sure it's running with checking `sudo systemctl status nginx`. 


**Domain**

Login at your DNS provider an A-Record for your domain to forward traffic to the elastic IP of your EC2 instance. Keep in mind, in some cases it can take up to few hours until your record is active, so stay patient. You can use `nslookup your.domain.here` to verify if DNS is working. 


**Configure Keycloak**

Open browser and log in to `https://keycloak.kbutz.org`. Username and password is what you're configured in docker-compose.yml in previous step. 
Within Keycloak:

* Create a new realm: `c8y-realm`

* Create new client within this realm:  `c8y-client-01`. Set:

    * Root URL: `https://kb-keycloak.latest.stage.c8y.io` (name of my Cumulocity tenant)

    * Home URL: `https://kb-keycloak.latest.stage.c8y.io`

    * Valid redirect URIs:  `https://kb-keycloak.latest.stage.c8y.io/tenant/oauth/*` 

    * Web origins: `https://kb-keycloak.latest.stage.c8y.io` 

    * Admin URL: `https://kb-keycloak.latest.stage.c8y.io` 

    * Authentication flow: `standard flow`, `Direct access grants`, `OAuth 2.0 Device Authorization Grant` 

* Create realm role: `c8y-admin` 

* Create user. Add credentials and assign `c8y-admin` role to the user. 

**Configure Cumulocity**

You can find the UI for SSO configuration in Administration > Settings > Authentication > Single Sign On. Additionally - and that's what we're doing here - you can also configure it via HTTP/Rest API endpoint: https://cumulocity.com/api/core/#operation/postLoginOptionCollectionResource

Please find my `cumulocity-oauth2.json` configuration in the repository, adapt domain- and client-id names and use the linked endpoint for SSO configuration. Once done, navigate to your Cumulocity domain, you can now login via the "Sign On via Keycloak` button.
