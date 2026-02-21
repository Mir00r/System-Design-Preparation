# **Docker Commands - Complete Cheatsheet** 🐳📋

**Quick Reference for All Essential Docker Commands**

---

## **Table of Contents** 📑
1. [Container Management](#1-container-management)
2. [Image Management](#2-image-management)
3. [Dockerfile Commands](#3-dockerfile-commands)
4. [Network Management](#4-network-management)
5. [Volume Management](#5-volume-management)
6. [Docker Compose](#6-docker-compose)
7. [System & Info](#7-system--info)
8. [Registry & Hub](#8-registry--hub)
9. [Logs & Monitoring](#9-logs--monitoring)
10. [Troubleshooting](#10-troubleshooting)
11. [Advanced Commands](#11-advanced-commands)
12. [One-Liners & Tricks](#12-one-liners--tricks)

---

## **1. Container Management** 📦

### **Running Containers**

```bash
# Run container (create + start)
docker run nginx
docker run -d nginx                    # Detached (background)
docker run -it ubuntu /bin/bash        # Interactive terminal
docker run --name mynginx nginx        # Custom name
docker run -p 8080:80 nginx            # Port mapping
docker run -v /host:/container nginx   # Volume mount
docker run -e VAR=value nginx          # Environment variable
docker run --rm nginx                  # Auto-remove when stopped

# Run with multiple options
docker run -d \
  --name web \
  -p 8080:80 \
  -v $(pwd)/html:/usr/share/nginx/html \
  -e NGINX_HOST=example.com \
  --restart=unless-stopped \
  --memory=512m \
  --cpus=1.0 \
  nginx:alpine

# Run and execute command
docker run ubuntu echo "Hello Docker"
docker run ubuntu ls -la /
docker run -it ubuntu /bin/bash
```

### **Container Lifecycle**

```bash
# List containers
docker ps                        # Running containers
docker ps -a                     # All containers (including stopped)
docker ps -q                     # Only container IDs
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}"

# Start/Stop
docker start container_name      # Start stopped container
docker stop container_name       # Stop running container (SIGTERM)
docker restart container_name    # Restart container
docker kill container_name       # Force stop (SIGKILL)
docker pause container_name      # Pause container
docker unpause container_name    # Unpause container

# Create (without starting)
docker create --name mynginx nginx

# Remove
docker rm container_name         # Remove stopped container
docker rm -f container_name      # Force remove running container
docker rm $(docker ps -aq)       # Remove all containers
docker container prune           # Remove all stopped containers
```

### **Container Interaction**

```bash
# Execute command in running container
docker exec container_name ls -la
docker exec -it container_name /bin/bash    # Interactive shell
docker exec -it container_name sh           # For Alpine
docker exec -u root container_name whoami   # As specific user

# Attach to running container
docker attach container_name

# Copy files
docker cp file.txt container_name:/path/
docker cp container_name:/path/file.txt ./

# View container details
docker inspect container_name
docker inspect container_name --format='{{.State.Status}}'
docker top container_name                    # Running processes
docker stats container_name                  # Resource usage
docker port container_name                   # Port mappings
```

---

## **2. Image Management** 🖼️

### **Image Operations**

```bash
# List images
docker images
docker images -a                 # All images (including intermediates)
docker images -q                 # Only image IDs
docker images --filter "dangling=true"  # Untagged images
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Pull image
docker pull nginx
docker pull nginx:1.25.3         # Specific version
docker pull nginx:alpine         # Specific variant

# Build image
docker build -t myapp .
docker build -t myapp:v1.0 .
docker build -t myapp:latest --no-cache .
docker build -f Dockerfile.prod -t myapp:prod .
docker build --build-arg VERSION=1.0 -t myapp .

# Tag image
docker tag myapp myapp:v1.0
docker tag myapp:v1.0 username/myapp:v1.0
docker tag myapp registry.example.com/myapp:latest

# Remove image
docker rmi image_name
docker rmi -f image_name         # Force remove
docker rmi $(docker images -q)   # Remove all images
docker image prune               # Remove dangling images
docker image prune -a            # Remove all unused images

# Image details
docker inspect image_name
docker history image_name        # Layer history
docker image ls --digests        # Show image digests
```

### **Image Import/Export**

```bash
# Save image to tar file
docker save -o myapp.tar myapp:v1.0
docker save myapp:v1.0 | gzip > myapp.tar.gz

# Load image from tar file
docker load -i myapp.tar
gunzip -c myapp.tar.gz | docker load

# Export container filesystem
docker export container_name -o container.tar

# Import container as image
docker import container.tar myapp:imported
```

---

## **3. Dockerfile Commands** 📝

### **Common Instructions**

```dockerfile
# Base image
FROM ubuntu:22.04
FROM node:18-alpine AS builder

# Metadata
LABEL maintainer="email@example.com"
LABEL version="1.0"
LABEL description="My application"

# Working directory
WORKDIR /app

# Copy files
COPY package.json .
COPY . /app
COPY --from=builder /app/dist /app

# Add files (can extract tar, download URLs)
ADD file.tar.gz /app
ADD https://example.com/file.txt /app

# Run commands
RUN apt-get update && apt-get install -y python3
RUN npm install
RUN pip install -r requirements.txt

# Environment variables
ENV NODE_ENV=production
ENV PORT=3000
ENV PATH="/app/bin:${PATH}"

# Arguments (build-time variables)
ARG VERSION=1.0
ARG BUILD_DATE

# Expose ports (documentation only)
EXPOSE 3000
EXPOSE 8080 8443

# Volume mount points
VOLUME ["/data"]
VOLUME /var/log

# User
USER appuser
USER 1000:1000

# Default command
CMD ["npm", "start"]
CMD ["/bin/bash"]

# Entrypoint (always runs)
ENTRYPOINT ["python", "app.py"]
ENTRYPOINT ["/docker-entrypoint.sh"]

# Healthcheck
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost/ || exit 1

# Stop signal
STOPSIGNAL SIGTERM

# Shell form vs Exec form
RUN echo "Shell form"              # Shell: /bin/sh -c
RUN ["echo", "Exec form"]          # Exec: Direct execution
```

### **Multi-Stage Build**

```dockerfile
# Build stage
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm install --production
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

---

## **4. Network Management** 🌐

### **Network Commands**

```bash
# List networks
docker network ls
docker network ls --filter driver=bridge

# Create network
docker network create mynetwork
docker network create --driver bridge mynetwork
docker network create --subnet=172.20.0.0/16 mynetwork
docker network create --driver overlay swarm-network

# Inspect network
docker network inspect mynetwork
docker network inspect bridge

# Connect container to network
docker network connect mynetwork container_name
docker network connect --ip 172.20.0.10 mynetwork container_name

# Disconnect container
docker network disconnect mynetwork container_name

# Remove network
docker network rm mynetwork
docker network prune            # Remove unused networks

# Run container on specific network
docker run -d --name web --network mynetwork nginx
docker run -d --name web --network=host nginx  # Host network mode
```

### **Network Drivers**

```bash
# Bridge (default)
docker network create --driver bridge my-bridge

# Host network (no isolation)
docker run --network host nginx

# None (no networking)
docker run --network none nginx

# Overlay (multi-host)
docker network create --driver overlay --attachable my-overlay

# Macvlan (assign MAC address)
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 macvlan-net
```

---

## **5. Volume Management** 💾

### **Volume Commands**

```bash
# List volumes
docker volume ls
docker volume ls --filter dangling=true

# Create volume
docker volume create myvolume
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.1,rw \
  --opt device=:/path/to/dir \
  nfs-volume

# Inspect volume
docker volume inspect myvolume

# Remove volume
docker volume rm myvolume
docker volume prune             # Remove unused volumes

# Use volume with container
docker run -v myvolume:/data nginx
docker run --mount source=myvolume,target=/data nginx
```

### **Volume Types**

```bash
# Named volume
docker run -v mydata:/app/data nginx

# Bind mount (host path)
docker run -v /host/path:/container/path nginx
docker run -v $(pwd):/app nginx

# Anonymous volume
docker run -v /app/data nginx

# Read-only mount
docker run -v mydata:/app/data:ro nginx
docker run -v $(pwd):/app:ro nginx

# Tmpfs mount (RAM)
docker run --tmpfs /app/cache nginx
docker run --mount type=tmpfs,destination=/app/cache nginx
```

---

## **6. Docker Compose** 🎼

### **Compose Commands**

```bash
# Start services
docker-compose up
docker-compose up -d                # Detached
docker-compose up --build           # Rebuild images
docker-compose up --scale web=3     # Scale service

# Stop services
docker-compose stop
docker-compose down                 # Stop and remove
docker-compose down -v              # Also remove volumes
docker-compose down --rmi all       # Also remove images

# View services
docker-compose ps
docker-compose ps -a
docker-compose top                  # Running processes

# Logs
docker-compose logs
docker-compose logs -f              # Follow
docker-compose logs service_name
docker-compose logs --tail=100 service_name

# Execute commands
docker-compose exec service_name sh
docker-compose exec web npm test
docker-compose run web npm install

# Build
docker-compose build
docker-compose build --no-cache
docker-compose build service_name

# Other
docker-compose config              # Validate and view config
docker-compose pull                # Pull service images
docker-compose restart             # Restart services
docker-compose pause               # Pause services
docker-compose unpause             # Unpause services
```

### **Compose File Example**

```yaml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "8080:80"
    environment:
      - NODE_ENV=production
    volumes:
      - ./app:/app
    depends_on:
      - db
    restart: unless-stopped
    
  db:
    image: postgres:14
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db-data:/var/lib/postgresql/data
      
volumes:
  db-data:
```

---

## **7. System & Info** 🖥️

### **System Commands**

```bash
# Docker version
docker --version
docker version               # Client and server info

# Docker info
docker info
docker info --format '{{.ServerVersion}}'

# Disk usage
docker system df             # Overview
docker system df -v          # Verbose

# Clean up
docker system prune          # Remove unused data
docker system prune -a       # Also remove unused images
docker system prune -a --volumes  # Also remove volumes
docker system prune --filter "until=24h"  # Older than 24h

# Events
docker events                # Real-time events
docker events --since '2024-01-01'
docker events --filter 'type=container'

# Builder cache
docker builder prune         # Remove build cache
```

---

## **8. Registry & Hub** 🌐

### **Registry Commands**

```bash
# Login
docker login
docker login registry.example.com
docker login -u username
docker login ghcr.io         # GitHub Container Registry

# Logout
docker logout
docker logout registry.example.com

# Search
docker search nginx
docker search --filter is-official=true python
docker search --filter stars=100 redis

# Pull
docker pull nginx
docker pull registry.example.com/myapp:v1.0
docker pull ghcr.io/user/repo:latest

# Push
docker tag myapp username/myapp:v1.0
docker push username/myapp:v1.0
docker push registry.example.com/myapp:v1.0

# Private registry
docker run -d -p 5000:5000 --name registry registry:2
docker tag myapp localhost:5000/myapp
docker push localhost:5000/myapp
docker pull localhost:5000/myapp
```

---

##**9. Logs & Monitoring** 📊

### **Logging**

```bash
# View logs
docker logs container_name
docker logs -f container_name        # Follow (tail -f)
docker logs --tail 100 container_name  # Last 100 lines
docker logs --since 2024-01-01 container_name
docker logs --since 1h container_name
docker logs --until 2024-01-01T12:00:00 container_name
docker logs -t container_name        # With timestamps

# Log drivers
docker run --log-driver json-file nginx
docker run --log-driver syslog nginx
docker run --log-driver journald nginx
docker run --log-driver none nginx   # No logging

# Log options
docker run --log-opt max-size=10m --log-opt max-file=3 nginx
```

### **Monitoring**

```bash
# Resource usage
docker stats                         # All containers
docker stats container_name          # Specific container
docker stats --no-stream             # No live update
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# Top processes
docker top container_name
docker top container_name aux

# Events
docker events
docker events --filter container=mynginx
docker events --filter event=start
```

---

## **10. Troubleshooting** 🔧

### **Debugging Commands**

```bash
# Inspect everything
docker inspect container_name
docker inspect image_name
docker inspect network_name
docker inspect volume_name

# Check container logs
docker logs -f --tail 100 container_name

# Get shell access
docker exec -it container_name /bin/bash
docker exec -it container_name sh

# Check container processes
docker top container_name

# View container changes
docker diff container_name

# Port mappings
docker port container_name

# Container resource usage
docker stats container_name

# Network connectivity
docker exec container_name ping google.com
docker exec container_name curl -I localhost

# DNS resolution
docker exec container_name nslookup google.com
docker exec container_name cat /etc/resolv.conf
```

### **Common Issues**

```bash
# Container won't start
docker logs container_name
docker inspect container_name
docker events --filter container=container_name

# Port already in use
docker ps | grep 8080
lsof -i :8080
netstat -tulpn | grep 8080

# Clean up space
docker system df
docker system prune -a --volumes

# Remove stuck containers
docker ps -a | grep Exited
docker rm -f $(docker ps -aq --filter status=exited)

# Network issues
docker network inspect bridge
docker exec container_name ip addr
docker exec container_name route -n
```

---

## **11. Advanced Commands** 🚀

### **Advanced Docker Run**

```bash
# Resource limits
docker run --memory=512m --memory-swap=1g nginx
docker run --cpus=1.5 nginx
docker run --cpus=2 --cpu-shares=512 nginx
docker run --pids-limit=100 nginx
docker run --blkio-weight=500 nginx

# Security
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx
docker run --security-opt=no-new-privileges nginx
docker run --read-only nginx
docker run --user 1000:1000 nginx
docker run --privileged nginx          # Full access (avoid!)

# Healthcheck
docker run --health-cmd='curl -f http://localhost/' \
  --health-interval=30s \
  --health-timeout=3s \
  --health-retries=3 \
  nginx

# Restart policies
docker run --restart=no nginx          # Never restart
docker run --restart=on-failure nginx  # Only on error
docker run --restart=on-failure:3 nginx # Max 3 retries
docker run --restart=unless-stopped nginx
docker run --restart=always nginx

# Networking
docker run --network=host nginx
docker run --network=none nginx
docker run --add-host=host.docker.internal:host-gateway nginx
docker run --dns=8.8.8.8 nginx
docker run --dns-search=example.com nginx
```

### **Docker Context**

```bash
# List contexts
docker context ls

# Create context (remote Docker)
docker context create mycontext --docker "host=ssh://user@remote"

# Use context
docker context use mycontext

# Switch back to default
docker context use default

# Remove context
docker context rm mycontext
```

### **BuildKit (Advanced Builds)**

```bash
# Enable BuildKit
export DOCKER_BUILDKIT=1

# Build with BuildKit
docker buildx build -t myapp .

# Multi-platform builds
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 -t myapp .

# Build and push
docker buildx build --platform linux/amd64,linux/arm64 \
  -t myapp:latest \
  --push .
```

---

## **12. One-Liners & Tricks** 💡

### **Cleanup One-Liners**

```bash
# Stop all running containers
docker stop $(docker ps -q)

# Remove all containers
docker rm $(docker ps -aq)

# Remove all images
docker rmi $(docker images -q)

# Remove dangling images
docker rmi $(docker images -f "dangling=true" -q)

# Remove exited containers
docker rm $(docker ps -aq --filter status=exited)

# Clean everything
docker system prune -a --volumes -f

# Remove containers older than 24 hours
docker ps -a --filter "until=24h" -q | xargs docker rm
```

### **Useful One-Liners**

```bash
# Get container IP address
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name

# Get container volumes
docker inspect -f '{{json .Mounts}}' container_name | jq .

# Container count
docker ps -q | wc -l

# Total disk usage
docker system df --format "table {{.Type}}\t{{.TotalCount}}\t{{.Size}}"

# Follow logs of multiple containers
docker-compose logs -f service1 service2

# Execute in all running containers
docker ps -q | xargs -I {} docker exec {} command

# Copy file to all containers
for container in $(docker ps -q); do
  docker cp file.txt $container:/path/
done

# Get logs from all containers
docker ps -q | xargs -L 1 docker logs
```

### **Quick Docker Aliases**

```bash
# Add to ~/.bashrc or ~/.zshrc

alias dps='docker ps'
alias dpsa='docker ps -a'
alias di='docker images'
alias dex='docker exec -it'
alias dlog='docker logs -f'
alias drm='docker rm'
alias drmi='docker rmi'
alias dstop='docker stop $(docker ps -q)'
alias dclean='docker system prune -a --volumes -f'
alias dcp='docker-compose'
alias dcup='docker-compose up -d'
alias dcdown='docker-compose down'
alias dclogs='docker-compose logs -f'
```

### **Check Docker Status**

```bash
# Quick status check
docker version && docker info && docker ps

# Health check script
#!/bin/bash
echo "Docker Version:"
docker --version

echo -e "\nRunning Containers:"
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

echo -e "\nDisk Usage:"
docker system df

echo -e "\nResource Usage:"
docker stats --no-stream
```

---

## **Quick Reference Table** 📊

| Category | Command | Description |
|----------|---------|-------------|
| **Run** | `docker run nginx` | Run container |
| | `docker run -d -p 80:80 nginx` | Detached with port |
| | `docker run -it ubuntu bash` | Interactive |
| **Life** | `docker start/stop/restart` | Control container |
| | `docker rm container_name` | Remove container |
| | `docker ps -a` | List all |
| **Images** | `docker build -t name .` | Build image |
| | `docker pull/push name` | Registry ops |
| | `docker images` | List images |
| **Exec** | `docker exec -it container bash` | Enter  container |
| | `docker logs -f container` | View logs |
| | `docker cp file container:/path` | Copy files |
| **System** | `docker system prune -a` | Clean up |
| | `docker stats` | Monitor |
| | `docker info` | System info |
| **Network** | `docker network create net` | Create network |
| | `docker network ls` | List networks |
| **Volume** | `docker volume create vol` | Create volume |
| | `docker volume ls` | List volumes |
| **Compose** | `docker-compose up -d` | Start services |
| | `docker-compose down` | Stop services |
| | `docker-compose logs -f` | View logs |

---

## **Emergency Commands** 🚨

```bash
# Docker not responding
sudo systemctl restart docker

# Kill all containers immediately
docker kill $(docker ps -q)

# Nuclear option - remove everything
docker stop $(docker ps -aq)
docker rm $(docker ps -aq)
docker rmi $(docker images -q)
docker volume rm $(docker volume ls -q)
docker network rm $(docker network ls -q)
docker system prune -a --volumes -f

# Reset Docker Desktop (macOS/Windows)
# Settings → Troubleshoot → Reset to factory defaults

# Repair Docker (Linux)
sudo apt-get purge docker-ce docker-ce-cli containerd.io
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

---

## **Next Steps** 📚

- **[Docker Fundamentals](Docker_Fundamentals.md)** - Core concepts and architecture
- **[Docker Images](Docker_Images.md)** - Building and optimizing images
- **[Docker Networking](Docker_Networking.md)** - Network concepts
- **[Docker Volumes](Docker_Volumes.md)** - Data persistence
- **[Docker Compose](Docker_Compose.md)** - Multi-container apps

---

**🐳 Keep This Cheatsheet Handy for Quick Reference!**

*Bookmark this page for instant access to all Docker commands you need.*
