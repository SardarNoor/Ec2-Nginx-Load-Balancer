
# EC2 NGINX Load Balancer on AWS

This project demonstrates the deployment of an NGINX Load Balancer on AWS EC2 to distribute traffic between two backend web servers running in private subnets.  
The architecture is fully secure and follows real-world cloud deployment patterns.

---

## Project Overview

The load balancer runs in a public subnet and receives traffic from the internet.  
It forwards HTTP requests to backend servers hosted in private subnets. Both backend servers run Apache with custom HTML pages to identify responses.

Traffic Flow:

Users → NGINX Load Balancer (Public Subnet) → Backend Server 1 & Backend Server 2 (Private Subnets)

---

## Architecture Diagram

The diagram below represents the full infrastructure:

![Architecture Diagram](./Nginx%20Load%20Balancer%20EC2.png)

---

## Architecture Components

- VPC CIDR: 10.0.0.0/16
- Subnets:
  - Public Subnet (NGINX Load Balancer)
  - Private Subnet 1 (Backend Server 1)
  - Private Subnet 2 (Backend Server 2)
- Internet Gateway: Provides external access to LB
- NAT Gateway: Allows backend servers to install updates
- Route Tables:
  - Public Route Table → Internet Gateway
  - Private Route Table → NAT Gateway
- Security Groups:
  - LB-SG → Allows HTTP from internet, SSH from My IP
  - Backend-SG → Allows HTTP only from LB-SG

---

## Backend Servers Setup (Private Subnets)

Both backend EC2 instances were deployed using Ubuntu in private subnets without public IP addresses.

### Install Apache and Configure the Web Page

Run these commands inside each backend server:

```bash
sudo apt update -y
sudo apt install apache2 -y
sudo systemctl enable apache2
sudo systemctl start apache2
echo "<h1>Hello from Backend Server X (SardarNoor)</h1>" | sudo tee /var/www/html/index.html
````

Replace **X** with Server1 or Server2 accordingly.

### Verification from the Load Balancer Instance

```bash
curl http://10.0.2.161
curl http://10.0.3.204
```

Each command should return its respective backend HTML response.

---

## NGINX Load Balancer Setup (Public Subnet)

### Install NGINX

```bash
sudo apt update -y
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

### Configure Load Balancing

Edit the NGINX configuration:

```bash
sudo nano /etc/nginx/nginx.conf
```

Add the following inside the `http` block:

```nginx
http {
    upstream backend_servers {
        server 10.0.2.161;
        server 10.0.3.204;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend_servers;
        }
    }
}
```

Test and restart:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

---

## Testing

Open a browser and access the public IP of the Load Balancer:

```
http://<LoadBalancer_Public_IP>
```

Refreshing the page multiple times displays alternating responses:

* Hello from Backend Server 1
* Hello from Backend Server 2

This confirms round-robin load balancing.

Using curl:

```bash
curl http://<LoadBalancer_Public_IP>
```

---

## Troubleshooting (Issues Faced and Solutions)

### 1. 502 Bad Gateway

**Cause:** Apache2 was not installed or running on Backend Server 1
**Solution:** Installed Apache manually and created index.html

---

### 2. Unable to SSH into Backend Servers

**Cause:** Backend servers are in private subnets; Load Balancer instance did not have the key pair
**Solution:** Copied key to LB using SCP and allowed SSH from LB security group

```bash
scp -i noor-key.pem noor-key.pem ubuntu@<LB_Public_IP>:/home/ubuntu/
chmod 400 noor-key.pem
```

---

### 3. User Data Did Not Execute

**Cause:** Missing `#!/bin/bash` line and reboot was used instead of Stop/Start
**Solution:** Corrected script and performed Stop → Start

---

### 4. Load Balancer Only Returned Backend Server 2

**Cause:** Apache was not running on Backend Server 1
**Solution:** Installed Apache; load balancing worked immediately

---

### 5. Route Table Confusion

Routes were correct; backend failure was misdiagnosed.
Real issue was missing Apache installation.

---

## Security Groups Summary

### Load Balancer SG (LB-SG)

* HTTP (80) → 0.0.0.0/0
* SSH (22) → My IP

### Backend SG (backend-servers-sg)

* HTTP (80) → Allowed only from LB-SG
* SSH (22) → Optional (for debugging from LB)

---

## Author

**Sardar Noor Ul Hassan**
Cloud Intern at Cloudelligent

