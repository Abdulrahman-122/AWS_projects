# Project -> Running our Todo on Ec2 instance (Rocky(i used it here) or whatever you want to go with)

 - note : this project is good for any one who want to be good linux admin + Devops Engineer later
# these are the missions that you need to prove while going through this project :
 - [ ] AWS EC2 provisioning
	    
- [ ] Rocky Linux administration
	
- [ ] SSH hardening (keys only)
	
- [ ] User and privilege management
	
- [ ] Docker and Docker Compose
	
- [ ] Container networking
	
- [ ] firewalld management
	
- [ ] AWS Security Groups

- [ ] SELinux policy configuration

- [ ] Nginx reverse proxy
	
- [ ] HTTP request flow
	
- [ ] Docker troubleshooting
	
- [ ] Nginx troubleshooting
	
- [ ] Linux networking (`ss`, `curl`)

---



## Final Architecture

```text
                    Internet
                        │
                AWS Security Group
                        │
                  firewalld (Rocky)
                        │
                     Nginx :80
                        │
              Reverse Proxy
                        │
          Docker Compose (FastAPI)
                        │
             PostgreSQL Container
```

---

# Phase 1 — Launch EC2

Launch a Rocky Linux EC2 instance.

Security Group:

```
22/TCP
80/TCP
```

(8000 was opened temporarily during testing.)

Connect

```bash
ssh -i Todo_rocky.pem ec2-user@PUBLIC_IP
```

---

# Phase 2 — Server Hardening

## Create a normal user

```bash
sudo adduser Tom
sudo passwd Tom
```

Give sudo privileges

```bash
sudo usermod -aG wheel Tom
```

Verify

```bash
groups Tom
```

---

## Copy SSH key

Create ssh directory

```bash
sudo mkdir /home/Tom/.ssh
sudo cp ~/.ssh/authorized_keys /home/Tom/.ssh/
sudo chown -R Tom:Tom /home/Tom/.ssh
sudo chmod 700 /home/Tom/.ssh
sudo chmod 600 /home/Tom/.ssh/authorized_keys
```

Login

```bash
ssh -i Todo_rocky.pem Tom@PUBLIC_IP
```

---

## Disable root login

```bash
sudo nano /etc/ssh/sshd_config
```

```
PermitRootLogin no
```

Restart

```bash
sudo systemctl restart sshd
```

---

## Disable password login

```
PasswordAuthentication no
```

Restart

```bash
sudo systemctl restart sshd
```

Now only SSH keys can authenticate.

---

# Phase 3 — Install Docker


- go to digital ocean and copy the commands for that ->https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-rocky-linux-8

```

Enable

```bash
sudo systemctl enable --now docker
```

Add yourself

```bash
sudo usermod -aG docker Tom
```

Reconnect

```bash
exit
ssh Tom@PUBLIC_IP
```

Verify

```bash
docker --version
docker compose version
```

---

# Problem #1

Wrong Docker repository

```
https://docker.com
```

Solution

Use

```
download.docker.com/linux/centos/docker-ce.repo
```

---

# Phase 4 — Clone GitHub Project

```bash
git clone YOUR_REPOSITORY
```

Go inside

```bash
cd Todo_with_CI-CD
```

Start

```bash
docker compose up -d
```

Check

```bash
docker ps
```

---

Expected

```
FastAPI

PostgreSQL
```

---

Logs

```bash
docker compose logs
```

or

```bash
docker logs CONTAINER_NAME
```

---

# Phase 5 — firewalld

Check

```bash
sudo firewall-cmd --list-all
```

Open HTTP

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

Temporary testing

```bash
sudo firewall-cmd --permanent --add-port=8000/tcp
sudo firewall-cmd --reload
```

---

# Phase 6 — AWS Security Group

Allow

```
22
80
```

Temporarily

```
8000
```

---

# Phase 7 — Verify FastAPI

Check Docker

```bash
docker ps
```

Verify

```
0.0.0.0:8000->8000
```

Check listening

```bash
ss -tulnp | grep 8000
```

Should show

```
docker-proxy
```

---

Test locally

```bash
curl localhost:8000
```

Should return HTML.

---

Test externally

```
http://PUBLIC_IP:8000
```

---

# Problem #2

Browser timed out.

Things checked

Docker

```
docker ps
```

Listening ports

```
ss -tulnp
```

AWS Security Group

firewalld

FastAPI logs

Eventually everything worked.

---

# Problem #3

Firefox cache

Browser kept timing out.

Solution

Hard refresh

```
Ctrl + Shift + R
```

or

```
curl
```

confirmed application worked.

---

# Phase 8 — Install Nginx

Install

```bash
sudo dnf install nginx
```

Enable

```bash
sudo systemctl enable --now nginx
```

---

Test

```bash
sudo nginx -t
```

---

# Create Reverse Proxy

```bash
sudo nano /etc/nginx/conf.d/fastapi.conf
```

Configuration

```nginx
server {

    listen 80;

    server_name _;

    location / {

        proxy_pass http://127.0.0.1:8000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

    }

}
```

Reload

```bash
sudo systemctl reload nginx
```

---

# Problem #4

Nginx still showed the Rocky default page.

Reason

Default server block was taking precedence.

Found using

```bash
sudo nginx -T
```

Warning

```
conflicting server name "_"
```

Solution

Comment out or remove the default `server` block in `/etc/nginx/nginx.conf`, leaving only your `fastapi.conf` server block, then reload Nginx.
or you can add the location lines inside nginx.conf and remove it's old ones

---

# Problem #5

502 Bad Gateway

Reason

Nginx couldn't connect to FastAPI.

Docker worked.

FastAPI worked.

The real issue was SELinux.

---

# SELinux

Check

```bash
getenforce
```

Output

```
Enforcing
```

Allow Nginx to connect to network services

```bash
sudo setsebool -P httpd_can_network_connect 1
```

Reload

```bash
sudo systemctl reload nginx
```

Problem solved.

---

# Verify Everything

Docker

```bash
docker ps
```

---

Nginx

```bash
sudo nginx -t
```

---

Service

```bash
sudo systemctl status nginx
```

---

Ports

```bash
ss -tulnp
```

---

HTTP

```bash
curl localhost
```

Should return your FastAPI HTML.

---

Public

```
http://PUBLIC_IP
```

Application loads through Nginx.

---

# Final Security

Remove

```
8000
```

from AWS Security Group.

Remove

```bash
sudo firewall-cmd --permanent --remove-port=8000/tcp
sudo firewall-cmd --reload
```

Now only

```
22

80
```

remain open.

FastAPI is no longer directly accessible.

Only Nginx communicates with it.

---

# uses this commands while you go through the project in order to understand more

```bash
docker ps
```

```bash
docker compose logs
```

```bash
sudo systemctl status nginx
```

```bash
sudo nginx -t
```

```bash
curl localhost
```

```bash
ss -tulnp
```

```bash
getenforce
```

```bash
sudo firewall-cmd --list-all
```

```bash
docker images
```

---
