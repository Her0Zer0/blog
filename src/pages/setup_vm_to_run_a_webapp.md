---
layout: '../layouts/BlogPost.astro'
title: 'Setup VM to run a Service'
description: 'This post will cover setting up your virtual machine (VM) host name, Ubuntu Server 20.04 in this example. Then creating a self signed certificate and NGINX to port our endpoint to Ollama API that is running locally.'
poster: 
    url: '/blog/assets/setup_vm_to_run_a_service_locally.png'
    alt: 'brain covered with circuits sitting next to a locked lock'
tags: ['nginx', 'linux']
published: '02/20/2025'
author: 'Robert Smith'
imageCredit: ''
---

## Overview
This post will cover setting up your virtual machine (VM) host name, Ubuntu Server 20.04 in this example. Then creating a self signed certificate and NGINX to port our endpoint to Ollama API that is running locally. Ollama is already installed on the VM but to get an understanding of that process please see the docs [here](https://github.com/ollama/ollama). 

## Setting Up The Host
To begin, we'll setup the host name of the VM for our use case. Lets use ```ollama-service.local```. 

To change the hostname of your server. 

```sh
sudo hostnamectl set-hostname ollama-service.local
```

Edit the _/etc/hosts_ file to reflect the change. 

From:
```txt
127.0.0.1 localhost
127.0.1.1 linux-server
```

To: 
```txt
127.0.0.1 localhost
127.0.1.1 ollama-service.local
```

Run the following command in the terminal to confirm the changes. The VM may need to be restarted for other services to take effect.
```sh
hostnamectl

# output
Static hostname: ollama-service.local
         Icon name: computer-vm
           Chassis: vm
        Machine ID: e94c741849c848f8baca2ec406f88c41
           Boot ID: 55f7533a0cb6448ba677ef30b2104481
    Virtualization: oracle
  Operating System: Ubuntu 20.04.1 LTS
            Kernel: Linux 5.4.0-42-generic
      Architecture: x86-64
```

## Setting Up NGINX
To begin, we will need to install NGINX. 
```sh
sudo apt-get update && sudo apt-get install nginx
```

Now let's make a directory to store the certificate and key. 
```sh
sudo mkdir -p /etc/nginx/ssl
```

Next, we'll generate a self signed certificate using openssl.
```sh
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/self-signed.key -out /etc/nginx/ssl/self-signed.crt
```

You will be prompted to enter information such as country, state, and domain name. Provide this information or leave them as default. 

Edit your NGINX configuration file. 
```sh
sudo nano /etc/nginx/sites-available/default
```
Add the following configuration. 
```sh
server {
    listen 443 ssl; # Enable SSL for this server block
    server_name ollama-service.local; # Your domain name

    ssl_certificate /etc/nginx/ssl/self-signed.crt; # Path to your SSL certificate
    ssl_certificate_key /etc/nginx/ssl/self-signed.key; # Path to your SSL key

    location / {
        proxy_pass http://localhost:11434; # Forward traffic to the Ollama service
        proxy_set_header Host $host; # Preserve the original Host header
        proxy_set_header X-Real-IP $remote_addr; # Preserve the client's IP address
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; # Forward the original client's IP address
        proxy_set_header X-Forwarded-Proto $scheme; # Preserve the original protocol (HTTP or HTTPS)
    }
}

```

Once you've saved the file, you can check the configuration by using: 
```sh
nginx -t
```
You should see something like the below:
```txt
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

If you encounter errors, the NGINX output will provide details on what needs to be corrected.

Finally we can start the service. 
```sh
systemctl start nginx
```

You can check the status of the service by running: 
```sh
systemctl status nginx
```
The output should be ```active (running)```.

If you are not seeing this in the output, check the error logs in ```/var/log/nginx/error.log```. You can follow the file in another terminal window by using the below command.

```sh
tail -f -n 1 /var/log/nginx/error.log
```

If you have your service up and running we can now check the results of the effort. 

## Posting to the Ollama Service
Ollama, by default, runs a service on Linux. This can be reached by posting to:
```http
GET /api/version
```

We can run ```curl``` with the ```-k``` option to see if we get the desired result. Using ```-k``` will allow bypass of the certificate authority check because it is a self-signed certificate.
```bat
curl -k https://ollama-server.local/api/version
# output
{"version": "0.5.7"}
```

## Conclusion
You should now have a working endpoint that accepts requests on 443 securely. There is more to secure but that will be in another article. I hope you found this article helpful in your journey. 


