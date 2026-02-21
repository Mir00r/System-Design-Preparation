# **Docker Compose - Complete Guide** 🎼

**Master Multi-Container Application Management with Docker Compose**

---

## **Table of Contents** 📑
1. [What is Docker Compose?](#1-what-is-docker-compose)
2. [Installation & Setup](#2-installation--setup)
3. [Compose File Structure](#3-compose-file-structure)
4. [Services Configuration](#4-services-configuration)
5. [Networks in Compose](#5-networks-in-compose)
6. [Vol

umes in Compose](#6-volumes-in-compose)
7. [Environment Variables](#7-environment-variables)
8. [Docker Compose Commands](#8-docker-compose-commands)
9. [Real-World Examples](#9-real-world-examples)
10. [Best Practices](#10-best-practices)
11. [Troubleshooting](#11-troubleshooting)
12. [Interview Cheat Sheet](#12-interview-cheat-sheet)

---

## **1. What is Docker Compose?** 🎯

### **Definition**

**Docker Compose** is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application's services, networks, and volumes. Then, with a single command, you create and start all the services.

### **The Problem It Solves**

```
Without Compose:
docker network create mynet
docker run -d --name db --network mynet postgres
docker run -d --name cache --network mynet redis
docker run -d --name web --network mynet -p 80:80 myapp
# Multiple commands, hard to manage, error-prone

With Compose:
docker-compose up
# One command, all services start correctly
```

### **Key Benefits**

```
✅ Single file configuration (docker-compose.yml)
✅ Multi-container orchestration
✅ One command to start/stop all services
✅ Environment isolation (multiple projects)
✅ Service dependencies management
✅ Volume and network management
✅ Easy scaling
✅ Perfect for development environments
```

### **Use Cases**

```
Development Environments:
  - Full-stack app (frontend + backend + database)
  - Microservices development
  - Testing environments
  
Demo & POC:
  - Quick setup of complex applications
  - Reproducible environments
  - Easy teardown and rebuild
  
Small Production (with caution):
  - Single-server deployments
  - Internal tools
  - Low-traffic applications
  
Note: For production at scale, use Kubernetes or Docker Swarm
```

---

## **2. Installation & Setup** 📦

### **Installation**

```bash
# Linux
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Or using pip
pip install docker-compose

# macOS/Windows (Docker Desktop)
# Compose included with Docker Desktop

# Verify installation
docker-compose --version
# Output: docker-compose version 2.23.0
```

### **Compose V1 vs V2**

```bash
# V1 (standalone binary)
docker-compose up

# V2 (Docker CLI plugin) - Recommended
docker compose up

# Both work, V2 is newer and integrated
```

---

## **3. Compose File Structure** 📝

### **Basic Structure**

```yaml
# docker-compose.yml

version: '3.8'                # Compose file version

services:                     # Service definitions
  web:                        # Service name
    image: nginx:alpine       # Docker image to use
    ports:
      - "80:80"               # Port mapping
      
  db:
    image: postgres:14
    environment:
      POSTGRES_PASSWORD: secret

networks:                     # Network definitions
  default:
    driver: bridge

volumes:                      # Volume definitions
  db-data:
    driver: local
```

### **Complete Example**

```yaml
version: '3.8'

services:
  # Frontend Service
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - API_URL=http://backend:5000
    volumes:
      - ./frontend:/app
      - /app/node_modules
    depends_on:
      - backend
    networks:
      - app-network
    restart: unless-stopped
    
  # Backend Service
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    ports:
      - "5000:5000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
      - REDIS_URL=redis://cache:6379
    volumes:
      - ./backend:/app
    depends_on:
      - db
      - cache
    networks:
      - app-network
    restart: unless-stopped
    
  # Database Service
  db:
    image: postgres:14-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - db-data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 5
      
  # Cache Service
  cache:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - cache-data:/data
    networks:
      - app-network
    restart: unless-stopped
    command: redis-server --appendonly yes
    
networks:
  app-network:
    driver: bridge
    
volumes:
  db-data:
  cache-data:
```

---

## **4. Services Configuration** ⚙️

### **Service Definition Options**

```yaml
services:
  myservice:
    # Image to use
    image: nginx:alpine             # From registry
    
    # OR build from Dockerfile
    build: .                        # Build context
    build:                          # Advanced build
      context: ./app
      dockerfile: Dockerfile.prod
      args:
        VERSION: "1.0"
    
    # Container name
    container_name: my-nginx        # Custom name (optional)
    
    # Ports
    ports:
      - "8080:80"                   # host:container
      - "443:443"
      - "127.0.0.1:9000:9000"      # bind to localhost only
    
    # Expose (documentation only, no host binding)
    expose:
      - "3000"
    
    # Environment variables
    environment:
      NODE_ENV: production
      DEBUG: "false"
    # OR from file
    env_file:
      - .env
      - .env.production
    
    # Volumes
    volumes:
      - ./app:/app                  # Bind mount
      - data:/var/lib/data          # Named volume
      - /app/node_modules           # Anonymous volume
    
    # Networks
    networks:
      - frontend
      - backend
    
    # Dependencies
    depends_on:
      - db
      - cache
    # With conditions (requires healthcheck)
    depends_on:
      db:
        condition: service_healthy
    
    # Command
    command: npm start
    command: ["python", "app.py"]
    
    # Entrypoint
    entrypoint: /app/entrypoint.sh
    entrypoint: ["sh", "-c"]
    
    # Working directory
    working_dir: /app
    
    # User
    user: "1000:1000"
    
    # Restart policy
    restart: "no"                   # Never restart
    restart: always                 # Always restart
    restart: on-failure             # Only on error
    restart: unless-stopped         # Always unless manually stopped
    
    # Resource limits
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    
    # Healthcheck
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    
    # Logging
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    
    # Labels
    labels:
      com.example.description: "My service"
      com.example.version: "1.0"
```

---

## **5. Networks in Compose** 🌐

### **Network Configuration**

```yaml
version: '3.8'

services:
  web:
    image: nginx
    networks:
      - frontend
      
  api:
    image: node:18
    networks:
      - frontend
      - backend
      
  db:
    image: postgres
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true              # No external access
```

### **Network Types**

```yaml
networks:
  # Default bridge
  default:
    driver: bridge
    
  # Custom bridge
  custom-bridge:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
          gateway: 172.20.0.1
    
  # Host network
  host-network:
    driver: host
    
  # Overlay (Swarm mode)
  overlay-network:
    driver: overlay
    attachable: true
    
  # External network (created outside Compose)
  existing-network:
    external: true
    name: my-pre-existing-network
```

### **Service Network Aliases**

```yaml
services:
  db:
    image: postgres
    networks:
      backend:
        aliases:
          - database
          - postgres-server
    
  api:
    image: node:18
    networks:
      - backend
    # Can connect to db using: database, postgres-server, or db

networks:
  backend:
```

---

## **6. Volumes in Compose** 💾

### **Volume Types**

```yaml
version: '3.8'

services:
  web:
    image: nginx
    volumes:
      # Named volume
      - web-data:/var/www/html
      
      # Bind mount (host path)
      - ./html:/usr/share/nginx/html
      
      # Anonymous volume
      - /var/cache/nginx
      
      # Read-only
      - ./config:/etc/nginx:ro
      
      # With volume options
      - type: bind
        source: ./app
        target: /app
        read_only: false
      
      - type: volume
        source: db-data
        target: /var/lib/postgresql/data
        volume:
          nocopy: true

volumes:
  web-data:                     # Named volume
  db-data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.1,rw
      device: ":/path/to/dir"
```

### **Volume Configuration**

```yaml
volumes:
  # Simple named volume
  db-data:
  
  # With driver and options
  nfs-data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=10.0.0.1,rw
      device: ":/exports/data"
  
  # External volume (created outside Compose)
  existing-volume:
    external: true
    name: my-existing-volume
  
  # With labels
  app-data:
    driver: local
    labels:
      com.example.description: "Application data"
```

---

## **7. Environment Variables** 🔐

### **Methods to Set Environment Variables**

```yaml
services:
  web:
    image: node:18
    
    # Method 1: Inline
    environment:
      NODE_ENV: production
      PORT: 3000
      DB_HOST: db
    
    # Method 2: Array format
    environment:
      - NODE_ENV=production
      - PORT=3000
    
    # Method 3: From .env file
    env_file:
      - .env
      - .env.production
    
    # Method 4: From host environment
    environment:
      HOST_USER: ${USER}
      HOST_HOME: ${HOME}
```

### **Using .env File**

```.env
# .env file
NODE_ENV=production
PORT=3000
DB_PASSWORD=secret
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    image: node:18
    ports:
      - "${PORT}:3000"          # Use variable from .env
    environment:
      - NODE_ENV=${NODE_ENV}
      - DB_PASSWORD=${DB_PASSWORD}
```

### **Environment Variable Substitution**

```yaml
services:
  web:
    image: nginx:${NGINX_VERSION:-latest}    # Default to 'latest'
    ports:
      - "${WEB_PORT:-8080}:80"                # Default to 8080
```

```bash
# Run with custom values
NGINX_VERSION=1.25.3 WEB_PORT=9000 docker-compose up
```

---

## **8. Docker Compose Commands** 🛠️

### **Basic Commands**

```bash
# Start services
docker-compose up                    # Foreground
docker-compose up -d                 # Detached (background)
docker-compose up --build            # Rebuild images before starting
docker-compose up --force-recreate   # Recreate containers
docker-compose up --no-deps web      # Start web without dependencies

# Stop services
docker-compose stop                  # Stop containers
docker-compose down                  # Stop and remove containers
docker-compose down -v               # Also remove volumes
docker-compose down --rmi all        # Also remove images
docker-compose down --remove-orphans # Remove containers for unlisted services

# View status
docker-compose ps                    # List containers
docker-compose ps -a                 # All containers
docker-compose top                   # Display running processes

# Logs
docker-compose logs                  # All logs
docker-compose logs -f               # Follow logs
docker-compose logs web              # Specific service
docker-compose logs --tail=100 web   # Last 100 lines
docker-compose logs -f --since 5m    # Last 5 minutes

# Execute commands
docker-compose exec web sh           # Shell in running container
docker-compose exec web ls -la       # Run command
docker-compose exec -u root web sh   # As specific user

# Run one-off commands
docker-compose run web npm install   # Creates new container
docker-compose run --rm web npm test # Remove after execution

# Build
docker-compose build                 # Build all images
docker-compose build --no-cache      # Without cache
docker-compose build web             # Specific service

# Pull images
docker-compose pull                  # Pull all service images
docker-compose pull web              # Specific service

# Restart
docker-compose restart               # Restart all
docker-compose restart web           # Specific service

# Pause/Unpause
docker-compose pause                 # Pause all
docker-compose unpause               # Unpause all

# Scale
docker-compose up -d --scale web=3   # Run 3 instances of web

# Config
docker-compose config                # Validate and view configuration
docker-compose config --services     # List services
docker-compose config --volumes      # List volumes

# Events
docker-compose events                # Real-time events
docker-compose events web            # Specific service
```

---

## **9. Real-World Examples** 🌍

### **Example 1: MERN Stack Application**

```yaml
# docker-compose.yml
version: '3.8'

services:
  # MongoDB
  mongodb:
    image: mongo:6
    container_name: mongodb
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: secret
    volumes:
      - mongodb_data:/data/db
    ports:
      - "27017:27017"
    networks:
      - mern-network
    restart: unless-stopped
    
  # Express Backend
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: express-backend
    ports:
      - "5000:5000"
    environment:
      - NODE_ENV=development
      - MONGODB_URI=mongodb://admin:secret@mongodb:27017/mydb?authSource=admin
      - JWT_SECRET=your-secret-key
    depends_on:
      - mongodb
    volumes:
      - ./backend:/app
      - /app/node_modules
    networks:
      - mern-network
    restart: unless-stopped
    command: npm run dev
    
  # React Frontend
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: react-frontend
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://localhost:5000
    depends_on:
      - backend
    volumes:
      - ./frontend:/app
      - /app/node_modules
    networks:
      - mern-network
    restart: unless-stopped
    stdin_open: true
    tty: true
    
  # Nginx Reverse Proxy
  nginx:
    image: nginx:alpine
    container_name: nginx-proxy
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - frontend
      - backend
    networks:
      - mern-network
    restart: unless-stopped

networks:
  mern-network:
    driver: bridge

volumes:
  mongodb_data:
```

### **Example 2: Microservices with Service Discovery**

```yaml
version: '3.8'

services:
  # Service Registry (Consul)
  consul:
    image: consul:latest
    ports:
      - "8500:8500"
    networks:
      - microservices
    command: agent -server -ui -bootstrap-expect=1 -client=0.0.0.0
    
  # Service 1: User Service
  user-service:
    build: ./services/user-service
    ports:
      - "3001:3000"
    environment:
      - SERVICE_NAME=user-service
      - CONSUL_HOST=consul
      - DB_HOST=postgres
    depends_on:
      - consul
      - postgres
    networks:
      - microservices
    restart: unless-stopped
    
  # Service 2: Order Service
  order-service:
    build: ./services/order-service
    ports:
      - "3002:3000"
    environment:
      - SERVICE_NAME=order-service
      - CONSUL_HOST=consul
      - DB_HOST=postgres
    depends_on:
      - consul
      - postgres
    networks:
      - microservices
    restart: unless-stopped
    
  # API Gateway
  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    environment:
      - CONSUL_HOST=consul
    depends_on:
      - consul
      - user-service
      - order-service
    networks:
      - microservices
    restart: unless-stopped
    
  # Database
  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: microservices
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - microservices
    restart: unless-stopped

networks:
  microservices:
    driver: bridge

volumes:
  postgres_data:
```

### **Example 3: WordPress with MySQL**

```yaml
version: '3.8'

services:
  db:
    image: mysql:8.0
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
    networks:
      - wordpress-network
    
  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    ports:
      - "8000:80"
    restart: always
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wordpress_data:/var/www/html
    networks:
      - wordpress-network

  # Optional: phpMyAdmin
  phpmyadmin:
    depends_on:
      - db
    image: phpmyadmin:latest
    restart: always
    ports:
      - "8080:80"
    environment:
      PMA_HOST: db
      MYSQL_ROOT_PASSWORD: rootpassword
    networks:
      - wordpress-network

networks:
  wordpress-network:
    driver: bridge

volumes:
  db_data:
  wordpress_data:
```

---

## **10. Best Practices** ⭐

### **Development Best Practices**

```yaml
✅ Use docker-compose.yml for core configuration
✅ Override with docker-compose.override.yml (auto-loaded)
✅ Use .env for environment-specific values
✅ Version control docker-compose.yml (not .env)
✅ Use bind mounts for development (live reload)
✅ Use health checks for dependencies
✅ Set resource limits
✅ Use meaningful service names
✅ Document exposed ports
✅ Use networks for service isolation
```

### **Multiple Environment Setup**

```bash
# Development
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up

# Production
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up

# Testing
docker-compose -f docker-compose.yml -f docker-compose.test.yml up
```

```yaml
# docker-compose.yml (base)
version: '3.8'
services:
  web:
    image: myapp
    
# docker-compose.dev.yml (development overrides)
version: '3.8'
services:
  web:
    build: .
    volumes:
      - ./:/app
    command: npm run dev
    
# docker-compose.prod.yml (production overrides)
version: '3.8'
services:
  web:
    restart: always
    environment:
      NODE_ENV: production
```

### **Don'ts**

```yaml
❌ Don't use :latest in production
❌ Don't hardcode secrets in compose file
❌ Don't expose database ports in production
❌ Don't use Compose for production (use Kubernetes/Swarm)
❌ Don't commit .env files
❌ Don't mix dependencies and deployment
❌ Don't run all services as root
```

---

## **11. Troubleshooting** 🔧

### **Common Issues**

```bash
# Issue: Port already in use
Error: Bind for 0.0.0.0:8080 failed: port is already allocated

# Solution 1: Change port in docker-compose.yml
ports:
  - "8081:80"

# Solution 2: Find and kill process
lsof -i :8080
kill -9 <PID>

# Issue: Service won't start
# Check logs
docker-compose logs service_name

# Issue: Cannot connect between services
# Check network
docker-compose exec service_name ping other_service
docker network inspect project_default

# Issue: Volume permission denied
# Fix permissions
docker-compose exec -u root service_name chown -R 1000:1000 /app

# Issue: Build cache issues
# Rebuild without cache
docker-compose build --no-cache

# Issue: Services not updating
# Force recreate
docker-compose up -d --force-recreate
```

---

## **12. Interview Cheat Sheet** 🎯

### **Q1: What is Docker Compose?**
```
Answer:
Docker Compose is a tool for defining and running multi-container
Docker applications using a YAML file (docker-compose.yml).

Benefits:
- Single command to start all services
- Service dependencies management
- Network and volume configuration
- Environment-specific configurations
- Easy scaling

Example:
docker-compose.yml defines web, database, cache services.
Command: docker-compose up
Result: All services start with proper networking
```

### **Q2: Compose vs Kubernetes?**
```
Docker Compose:
- Single host (one machine)
- Development/testing
- Simple orchestration
- Easy to learn
- YAML configuration

Kubernetes:
- Multi-host (cluster)
- Production at scale
- Advanced orchestration
- Steeper learning curve
- YAML configuration

Use Compose for: Dev, small deployments
Use Kubernetes for: Production, scaling
```

### **Q3: Common Compose commands?**
```
Start:     docker-compose up -d
Stop:      docker-compose down
Logs:      docker-compose logs -f
Execute:   docker-compose exec service sh
Build:     docker-compose build
Scale:     docker-compose up -d --scale web=3
Config:    docker-compose config
```

---

## **Next Steps** 📚

- **[Docker Fundamentals](Docker_Fundamentals.md)** - Core concepts
- **[Docker Networking](Docker_Networking.md)** - Network deep dive
- **[Docker Volumes](Docker_Volumes.md)** - Data persistence
- **[Docker Commands Cheatsheet](Docker_Commands_Cheatsheet.md)** - Quick reference

---

**🎼 Master Docker Compose for Efficient Multi-Container Management!**

*Docker Compose simplifies complex multi-container setups - essential for modern development.*
