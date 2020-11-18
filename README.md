# How to run FoundryVTT on a Raspberry Pi 4 using nginx and foundryvtt-docker

## Prerequisites:
* External SSD with USB 3.0 cable
* Raspberry Pi 4

## Format SSD so you can boot from it
1. Install Pi OS on the external SSD. I downloaded [Raspberry Pi Imager](https://www.raspberrypi.org/software/) on my adjacent Mac and used that to install the latest Raspbery Pi OS, but you can use another method. 
<img width="674" alt="Screen Shot 2020-11-17 at 10 05 03 PM" src="https://user-images.githubusercontent.com/33645693/99486155-f588af00-2920-11eb-805d-facb578fdabe.png">

2. Once the install finishes, unplug the SSD, remove the SD card from your raspberry pi (if it has one), plug in the SSD, and boot it up from the SSD.
  
## Setup Steps
Basically just follow the steps as indicated on the FoundryVTT [installation](https://foundryvtt.com/article/installation/) instructions for Node.js, [Nginx](https://foundryvtt.com/article/nginx/), and [foundryvtt-docker](https://github.com/felddy/foundryvtt-docker). You can get 5 free subdomains from [DuckDNS](http://www.duckdns.org/).

1. Go to www.duckdns.org, sign in, and create a subdomain of your choice. 

![Screen Shot 2020-11-17 at 7 41 23 PM](https://user-images.githubusercontent.com/33645693/99475967-e13ab700-290c-11eb-8cd6-caa0bd11e64a.png)

Then go [here](https://www.duckdns.org/install.jsp?tab=pi) and follow the installation instructions.

2. Install Node.js
```
sudo apt install -y libssl-dev
curl -sL https://deb.nodesource.com/setup_12.x | sudo bash -
sudo apt install -y nodejs
```
3. Create a directory for the application data
```
cd $HOME
mkdir foundrydata
```
4. Install nginx: 
```
sudo apt-get update
sudo apt-get install nginx
```
5. Create an Nginx configuration file for your domain. Make sure to update the references to `your.hostname.com` in the configuration file: `sudo nano /etc/nginx/sites-available/your.hostname.com`
```
# This goes in a file within /etc/nginx/sites-available/. By convention,
# the filename would be either "your.domain.com" or "foundryvtt", but it
# really does not matter as long as it's unique and descriptive for you.

# Define Server
server {

    # Enter your fully qualified domain name or leave blank
    server_name             your.hostname.com;

    # Listen on port 80 without SSL certificates
    listen                  80;

    # Sets the Max Upload size to 300 MB
    client_max_body_size 300M;

    # Proxy Requests to Foundry VTT
    location / {

        # Set proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # These are important to support WebSockets
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";

        # Make sure to set your Foundry VTT port number
        proxy_pass http://localhost:30000;
    }
}
```
Save and exit nano with ctrl + x, Press Y, then Enter

6. Use the `service` utility to manage your Nginx server. Enable the site, test your configuration, fix any errors, and start the service.
```
# Enable new site
sudo ln -s /etc/nginx/sites-available/your.hostname.com /etc/nginx/sites-enabled/

# Test your configuration file
sudo service nginx configtest

# View configuration errors (if the configtest was not OK)
sudo nginx -t

# Start Nginx
sudo service nginx start

# Stop Nginx
sudo service nginx stop

# Restart Nginx
sudo service nginx restart
```
7. Next, create a docker-compose file to run foundryvtt-docker: `nano $HOME/docker-compose.yml`. See the [foundryvtt-docker](https://github.com/felddy/foundryvtt-docker) for instructions on using secrets, a temporary foundryvtt download url, and other options. Make sure to replace `YOUR_USERNAME`, `YOUR_PASSWORD`, and `YOUR_ADMIN_KEY` before saving the file.
```
version: "3.3"

services:
  foundry_1:
    container_name: foundry
    image: felddy/foundryvtt:release
    hostname: my_foundry_host_1
    restart: "unless-stopped"
    network_mode: webproxy
    volumes:
      - type: bind
        source: ./foundrydata
        target: /data
    environment:
      - CONTAINER_CACHE=/data/container_cache
      - FOUNDRY_USERNAME=YOUR_USERNAME
      - FOUNDRY_PASSWORD=YOUR_PASSWORD
      - FOUNDRY_ADMIN_KEY=YOUR_ADMIN_KEY
    ports:
      - target: "30000"
        published: "30000"
        protocol: tcp
```
*Note: if you need to run FoundryVTT on a different port, change the published port to the desired port. No need to change the target port. The `target` is the port inside the container, the `published` is the publicly exposed port.*

8. Now you should be able to start the contianer and see FoundryVTT running at http://localhost:30000 and at http://your.hostname.com:
```
docker-compose up -d
```
Check that your container is running using `docker container ls`, view the container logs using `docker logs foundry`. If needed you can stop the container `docker stop foundry`, remove it `docker rm foundry`, and run it again after making any necessary changes to your docker compose file `docker-compose up -d`. Alternately, `docker-compose down` will stop all containers and removes containers, networks, volumes, and images created by the previous `docker-compose up` in a single command.

9. Now it's time to setup HTTPS for your domain. Create SSL certificates using Certbot. Follow the instructions [here](https://certbot.eff.org/lets-encrypt/debianbuster-nginx).
10. Update the nginx config file to use port 443 and the SSL certificates you created. Again, make sure to replace `your.hostname.com`: `sudo nano /etc/nginx/sites-available/your.hostname.com`
```
# the filename needs to be "your.hostname.com"

# Define Server
server {

    # Enter your fully qualified domain name or leave blank
    server_name             your.hostname.com;

    # Listen on port 443 using SSL certificates
    listen                  443 ssl;
    ssl_certificate         "/etc/letsencrypt/live/your.hostname.com/fullchain.pem";
    ssl_certificate_key     "/etc/letsencrypt/live/your.hostname.com/privkey.pem";

    # Sets the Max Upload size to 300 MB
    client_max_body_size 300M;

    # Proxy Requests to Foundry VTT
    location / {

        # Set proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # These are important to support WebSockets
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";

        # Make sure to set your Foundry VTT port number
        proxy_pass http://localhost:30000;
    }
}

# Optional, but recommend. Redirects all HTTP requests to HTTPS for you
server {
    if ($host = your.hostname.com) {
        return 301 https://$host$request_uri;
    }

    listen 80;
	listen [::]:80;

    server_name your.hostname.com;
    return 404;
}
```
Save and exit nano with ctrl + x, Press Y, then Enter

11. Don't forget to setup Port Forwarding on your router. You'll need to forward ports 80 and 443 to your raspberry pi. Every router is different, but you can find specific instructions [here](https://portforward.com/).

Now your site should be accessible at `https://your.hostname.com`!

