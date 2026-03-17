# Experiment 5: Docker - Volumes, Environment Variables, Monitoring & Networks

## Part 1: Docker Volumes - Persistent Data Storage

### Lab 1: Understanding Data Persistence

**The Problem: Container Data is Ephemeral**
```bash 
# Create a container that writes data
docker run -it --name test-container ubuntu /bin/bash
# Inside container:
echo "Hello World" > /data/message.txt
cat /data/message.txt # Shows "Hello World"
exit

# Restart container
docker start test-container
docker exec test-container cat /data/message.txt
# ERROR: File doesn't exist! Data was lost.
Solution: Docker Volumes
```

<img width="931" height="336" alt="image" src="https://github.com/user-attachments/assets/537925cb-f19c-4709-9406-2960dd12662f" />


## Lab 2: Volume Types

```bash
# Create anonymous volume (auto-generated name)
docker run -d -v /app/data --name web1 nginx
# Check volume
docker volume ls
# Inspect container to see volume mount
docker inspect web1 | grep -A 5 Mounts

2. Named Volumes
Bash
# Create named volume
docker volume create mydata

# Use named volume
docker run -d -v mydata:/app/data --name web2 nginx

# List volumes
docker volume ls
# Shows: mydata

# Inspect volume
docker volume inspect mydata
```
<img width="881" height="453" alt="image" src="https://github.com/user-attachments/assets/dddb77f0-91c3-4578-92a7-4bf76e0a0683" />

3. Bind Mounts (Host Directory)
```Bash
# Create directory on host
mkdir ~/myapp-data

# Mount host directory to container
docker run -d -v ~/myapp-data:/app/data --name web3 nginx

# Add file on host
echo "From Host" > ~/myapp-data/host-file.txt

# Check in container
docker exec web3 cat /app/data/host-file.txt
# Shows: From Host
```
<img width="933" height="817" alt="image" src="https://github.com/user-attachments/assets/3bd57864-9eba-437f-b57a-c2f4e07b2b80" />



### Lab 3: Practical Volume Examples
Example 1: Database with Persistent Storage

```Bash
# MySQL with named volume
docker run -d \
  --name mysql-db \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8.0

# Check data persists
docker stop mysql-db
docker rm mysql-db

# New container with same volume
docker run -d \
  --name new-mysql \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8.0
# Data is preserved!
```

Example 2: Web App with Configuration Files
```Bash

# Create config directory
mkdir ~/nginx-config

# Create nginx config file
echo 'server {
    listen 80;
    server_name localhost;
    location / {
        return 200 "Hello from mounted config!";
    }
}' > ~/nginx-config/nginx.conf

# Run nginx with config bind mount
docker run -d \
  --name nginx-custom \
  -p 8080:80 \
  -v ~/nginx-config/nginx.conf:/etc/nginx/conf.d/default.conf \
  nginx

# Test
curl http://localhost:8080
```
<img width="933" height="817" alt="image" src="https://github.com/user-attachments/assets/b0783e2d-06b5-4cf0-bcf1-b0fb6d506cf8" />


Lab 4: Volume Management Commands
```Bash
# List all volumes
docker volume ls

# Create a volume
docker volume create app-volume

# Inspect volume details
docker volume inspect app-volume

# Remove unused volumes
docker volume prune

# Remove specific volume
docker volume rm volume-name

# Copy files to/from volume
docker cp local-file.txt container-name:/path/in/volume

```
<img width="927" height="520" alt="image" src="https://github.com/user-attachments/assets/2c0999e6-1e26-4d50-90ea-f5441633ea3e" />

<img width="341" height="362" alt="image" src="https://github.com/user-attachments/assets/f82122fa-a076-42db-9a3b-5aaf0ebab90d" />


## Part 2: Environment Variables
Lab 1: Setting Environment Variables
## Method 1: Using -e flag

```Bash
# Single variable
docker run -d \
  --name app1 \
  -e DATABASE_URL="postgres://user:pass@db:5432/mydb" \
  -e DEBUG="true" \
  -p 3000:3000 \
  my-node-app

# Multiple variables
docker run -d \
  -e VAR1=value1 \
  -e VAR2=value2 \
  -e VAR3=value3 \
  my-app
```

### Method 2: Using --env-file
``` Bash
# Create .env file
echo "DATABASE_HOST=localhost" > .env
echo "DATABASE_PORT=5432" >> .env
echo "API_KEY=secret123" >> .env

# Use env file
docker run -d \
  --env-file .env \
  --name app2 \
  my-app

# Use multiple env files
docker run -d \
  --env-file .env \
  --env-file .env.secrets \
  my-app
```
<img width="950" height="1018" alt="image" src="https://github.com/user-attachments/assets/54681f7d-2915-47b5-ba0c-6543b7f30632" />

### Method 3: In Dockerfile
```Bash
# Set default environment variables
ENV NODE_ENV=production
ENV PORT=3000
ENV APP_VERSION=1.0.0
# Can be overridden at runtime
```
## Lab 2: Environment Variables in Applications
```Bash
Python Flask Example

app.py
import os
from flask import Flask

app = Flask(__name__)
Read environment variables
db_host = os.environ.get('DATABASE_HOST', 'localhost')
debug_mode = os.environ.get('DEBUG', 'false').lower() == 'true'
api_key = os.environ.get('API_KEY')

@app.route('/config')
def config():
    return {
        'db_host': db_host,
        'debug': debug_mode,
        'has_api_key': bool(api_key)
    }

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 5000))
    app.run(host='0.0.0.0', port=port, debug=debug_mode)
```
<img width="766" height="603" alt="image" src="https://github.com/user-attachments/assets/89a643fe-4c6c-418a-97ca-769fbbc1f65d" />


### Dockerfile with Environment Variables
```Bash

Dockerfile
FROM python:3.9-slim
Set environment variables at build time
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .

# Default runtime environment variables
ENV PORT=5000
ENV DEBUG=false

EXPOSE 5000
CMD ["python", "app.py"]
```
<img width="465" height="168" alt="image" src="https://github.com/user-attachments/assets/97a403a0-1a5c-4475-bfaf-839c5e6a7af6" />

### Lab 3: Test Environment Variables
```Bash
# Run with custom env vars

docker run -d \
  --name flask-app \
  -p 5000:5000 \
  -e DATABASE_HOST="prod-db.example.com" \
  -e DEBUG="true" \
  -e PORT="8080" \
  flask-app

# Check environment in running container
docker exec flask-app env
docker exec flask-app printenv DATABASE_HOST

# Test the endpoint
curl http://localhost:5000/config
```
<img width="953" height="1000" alt="image" src="https://github.com/user-attachments/assets/976583a1-8b51-481d-b3bb-7841f688e395" />

### Part 3: Docker Monitoring 
### Lab 1: Basic Monitoring Commands
### docker stats - Real-time Container Metrics

```bash
# Live stats for all containers
docker stats

# Live stats for specific containers
docker stats container1 container2

# Specific format output
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"

# No-stream (single snapshot)
docker stats --no-stream

# All containers (including stopped)
docker stats --all

# Useful Format Options:
# Custom format
docker stats --format "Container: {{.Name}} | CPU: {{.CPUPerc}} | Memory: {{.MemPerc}}"
# JSON output
docker stats --format json --no-stream
# Wide output
docker stats --no-stream --no-trunc

```
<img width="951" height="1017" alt="image" src="https://github.com/user-attachments/assets/82b707ae-dd58-4e1f-b669-cea38dd60ac0" />

### Lab 2: docker top - Process Monitoring
``` Bash
# View processes in container
docker top container-name

# View with full command line
docker top container-name -ef

# Compare with host processes
ps aux | grep docker
```

### Lab 3: docker logs - Application Logs
```Bash
# View logs
docker logs container-name

# Follow logs (like tail -f)
docker logs -f container-name

# Last N lines
docker logs --tail 100 container-name

# Logs with timestamps
docker logs -t container-name

# Logs since specific time
docker logs --since 2024-01-15 container-name

# Combine options
docker logs -f --tail 50 -t container-name
```
<img width="932" height="876" alt="image" src="https://github.com/user-attachments/assets/6288a82c-5050-43de-9677-5b410e98ad97" />

### Lab 4: Container Inspection
```Bash
# Detailed container info
docker inspect container-name

# Specific information
docker inspect --format='{{.State.Status}}' container-name
docker inspect --format='{{.NetworkSettings.IPAddress}}' container-name
docker inspect --format='{{.Config.Env}}' container-name

# Resource limits
docker inspect --format='{{.HostConfig.Memory}}' container-name
docker inspect --format='{{.HostConfig.NanoCpus}}' container-name
Lab 5: Events Monitoring
Bash
# Monitor Docker events in real-time
docker events

# Filter events
docker events --filter 'type=container'
docker events --filter 'event=start'
docker events --filter 'event=die'

# Since specific time
docker events --since '2024-01-15'

# Format output
docker events --format '{{.Type}} {{.Action}} {{.Actor.Attributes.name}}'
```
### Lab 6: Practical Monitoring Script
```Bash

#!/bin/bash
# monitor.sh
# Simple Docker monitoring

echo "=== Docker Monitoring Dashboard ==="
echo "Time: $(date)"
echo

echo "1. Running Containers:"
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
echo

echo "2. Resource Usage:"
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"
echo

echo "3. Recent Events:"
docker events --since '5m' --until '0s' --format '{{.Time}} {{.Type}} {{.Action}}' | tail -5
echo

echo "4. System Info:"
docker system df
```
### Part 4: Docker Networks
### Lab 1: Understanding Docker Network Types
```Bash
# Default networks
docker network ls

# Output:
# NETWORK ID          NAME                DRIVER              SCOPE
# abc123              bridge              bridge              local
# def456              host                host                local
# ghi789              none                null                local
Lab 2: Network Types Explained
1. Bridge Network (Default)
Bash
# Containers on bridge network can communicate
# Each container gets own IP, isolated from host

# Create custom bridge network
docker network create my-network

# Inspect network
docker network inspect my-network

# Run containers on custom network
docker run -d --name web1 --network my-network nginx
docker run -d --name web2 --network my-network nginx

# Containers can communicate using container names
docker exec web1 curl http://web2
```
<img width="937" height="902" alt="image" src="https://github.com/user-attachments/assets/50eb952c-0741-42b7-aecd-aa3283db0b60" />

### 2. Host Network
```Bash
# Container uses host's network directly
# No network isolation, shares host's IP

docker run -d --name host-app --network host nginx

# Access directly on host port 80
curl http://localhost
```
<img width="933" height="595" alt="image" src="https://github.com/user-attachments/assets/17e96af9-9505-4af9-abd9-f35942a5ff6b" />


### 3. None Network
```Bash
# No network access
docker run -d --name isolated-app --network none alpine sleep 3600

# Test no network interfaces
docker exec isolated-app ifconfig

```
<img width="882" height="322" alt="image" src="https://github.com/user-attachments/assets/ff0ccde7-e9f1-4498-8bd2-21232803d13d" />

### 4. Overlay Network (Swarm)
```Bash
# For Docker Swarm multi-host networking
docker network create --driver overlay my-overlay
```
<img width="603" height="220" alt="image" src="https://github.com/user-attachments/assets/8e7a87f8-f32f-4716-8f5e-2d152cb423a6" />

### Lab 3: Network Management Commands
```Bash
# Create network
docker network create app-network
docker network create --driver bridge --subnet 172.20.0.1/16 my-subnet

# Connect container to network
docker network connect app-network existing-container

# Disconnect container from network
docker network disconnect app-network container-name

# Remove network
docker network rm network-name

# Prune unused networks
docker network prune
```
<img width="618" height="45" alt="image" src="https://github.com/user-attachments/assets/a8f0a548-a958-4904-a1aa-d27823ad5bf2" />

<img width="952" height="682" alt="image" src="https://github.com/user-attachments/assets/8c5e60e9-1b7f-40ae-ba10-829dc2d4614d" />

### Lab 4: Multi-Container Application Example
Web App + Database Communication

```Bash
# Create network
docker network create app-network

# Start database
docker run -d \
  --name postgres-db \
  --network app-network \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:15

# Start web application
docker run -d \
  --name web-app \
  --network app-network \
  -p 8080:3000 \
  -e DATABASE_URL="postgres://postgres:secret@postgres-db:5432/mydb" \
  -e DATABASE_HOST="postgres-db" \
  node-app

# Web app can connect to database using "postgres-db" hostname
```
<img width="618" height="45" alt="image" src="https://github.com/user-attachments/assets/db770d13-26f0-4c7d-8b26-4b4cb2bc4a32" />

<img width="932" height="397" alt="image" src="https://github.com/user-attachments/assets/228f991b-7e49-4027-9ca2-ea70db0f8995" />



### Lab 5: Network Inspection & Debugging
```Bash
# Inspect network
docker network inspect bridge

# Check container IP
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container-name

# DNS resolution test
docker exec container-name nslookup another-container

# Network connectivity test
docker exec container-name ping -c 4 google.com
docker exec container-name curl -I http://another-container

# View network ports
docker port container-name
```
<img width="1430" height="128" alt="image" src="https://github.com/user-attachments/assets/8673c7a0-d0b9-46b2-83ab-7283d43b47e5" />


### Lab 6: Port Publishing vs Exposing
``` Bash
# PORT PUBLISHING (host:container)
docker run -d -p 80:8080 --name app1 nginx
# Host port 80 → Container port 8080

# Dynamic port publishing
docker run -d -p 8080 --name app2 nginx
# Docker assigns random host port

# Multiple ports
docker run -d -p 80:80 -p 443:443 --name app3 nginx

# Specific host IP
docker run -d -p 127.0.0.1:8080:80 --name app4 nginx

# EXPOSE in Dockerfile (metadata only)
# Dockerfile: EXPOSE 80
# Still need -p to publish
```
<img width="1447" height="322" alt="image" src="https://github.com/user-attachments/assets/9867ea4c-4e9d-44b5-83f8-d4e6e5d57cf0" />


### Part 5: Complete Real-World Example
Application Architecture:

Flask Web App (port 5000)

PostgreSQL Database (port 5432)

Redis Cache (port 6379)

All connected via custom network

Implementation:

```Bash
# 1. Create network
docker network create myapp-network

# 2. Start database with volume
docker run -d \
  --name postgres \
  --network myapp-network \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -e POSTGRES_DB=mydatabase \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:15

# 3. Start Redis
docker run -d \
  --name redis \
  --network myapp-network \
  -v redis-data:/data \
  redis:7-alpine

# 4. Start Flask app with all configurations
docker run -d \
  --name flask-app \
  --network myapp-network \
  -p 5000:5000 \
  -v $(pwd)/app:/app \
  -v app-logs:/var/log/app \
  -e DATABASE_URL="postgresql://postgres:mysecretpassword@postgres:5432/mydatabase" \
  -e REDIS_URL="redis://redis:6379" \
  -e DEBUG="false" \
  -e LOG_LEVEL="INFO" \
  --env-file .env.production \
  flask-app:latest
```
<img width="1458" height="428" alt="image" src="https://github.com/user-attachments/assets/135b4c43-fef3-4610-81d2-67e535c6c824" />

```bash
Monitoring commands 
# Check all components
docker ps

# Monitor resources
docker stats postgres redis flask-app

# Check logs
docker logs -f flask-app

# Network connectivity test
docker exec flask-app ping -c 2 postgres
docker exec flask-app ping -c 2 redis

# View network details
docker network inspect myapp-network
```
<img width="1341" height="235" alt="image" src="https://github.com/user-attachments/assets/5af457a1-a678-424a-bfdd-e41df9d2b530" />

<img width="772" height="650" alt="image" src="https://github.com/user-attachments/assets/3df003d4-818a-45e0-903f-84aa4af497be" />

<img width="1446" height="652" alt="image" src="https://github.com/user-attachments/assets/b434e77e-c914-4dab-bd77-8c8b90596a32" />

