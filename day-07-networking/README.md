# Day 7: Docker Networking - Complete Deep Dive

## 🎯 Why Do We Need Docker Networking?

### The Problem

**Imagine building a web application:**
```
Frontend (React) → needs to talk to → Backend (API)
Backend (API) → needs to talk to → Database (PostgreSQL)
```

**Questions:**
- How does Frontend find Backend?
- How does Backend find Database?
- Can containers talk to each other?
- What about security?

**Docker Networking solves all of this!**

---

## 🌐 Understanding Container Networking

### Without a Network

```powershell
# Run two containers
docker run -d --name container1 nginx
docker run -d --name container2 nginx

# Can they talk to each other?
docker exec container1 ping container2
# ❌ ERROR: Unable to resolve host 'container2'
```

**Why?** They're isolated, like people in different rooms with no way to communicate.

### With a Network

```powershell
# Create a network
docker network create mynetwork

# Run containers on the network
docker run -d --name container1 --network mynetwork nginx
docker run -d --name container2 --network mynetwork nginx

# Now they can talk!
docker exec container1 ping container2
# ✅ SUCCESS! Ping replies!
```

**Why?** They're now in the same "room" and can see each other by name.

---

## 📚 Four Network Types

### 1️⃣ Bridge Network (Most Common)

**What:** Default network, containers get their own subnet  
**When:** Most applications, isolated communication  
**How:** Containers can talk to each other by name  

```
Your Computer (Host)
┌────────────────────────────────────────┐
│                                        │
│  Docker Bridge Network (172.17.0.0/16) │
│  ┌──────────┐      ┌──────────┐      │
│  │Container1│◄────►│Container2│      │
│  │172.17.0.2│      │172.17.0.3│      │
│  └──────────┘      └──────────┘      │
│         ↕               ↕             │
└─────────┼───────────────┼─────────────┘
          └───────┬───────┘
              Internet
```

**Example:**
```powershell
# Create bridge network (explicit)
docker network create my-app-network

# Run containers
docker run -d --name web --network my-app-network nginx
docker run -d --name api --network my-app-network node:18

# Web can talk to API using name "api"
docker exec web ping api
```

---

### 2️⃣ Host Network

**What:** Container uses host's network directly  
**When:** Need maximum performance, no isolation needed  
**How:** Container shares host's IP address  

```
Your Computer (192.168.1.100)
┌────────────────────────────────────────┐
│                                        │
│  Container uses HOST network           │
│  ┌──────────┐                         │
│  │Container │  (No isolation)          │
│  │   Uses   │  IP: 192.168.1.100      │
│  │  Host IP │                          │
│  └──────────┘                         │
│                                        │
└────────────────────────────────────────┘
```

**Example:**
```powershell
# Container listens on host's ports directly
docker run -d --name web --network host nginx
# nginx now accessible on host's IP without -p mapping!

# Warning: Port conflicts possible!
```

**⚠️ Note:** Host networking doesn't work the same way on Windows/Mac as on Linux due to Docker Desktop architecture.

---

### 3️⃣ None Network

**What:** No network at all  
**When:** Maximum isolation, processing containers  
**How:** Container has loopback only  

```
Container (Isolated)
┌────────────────────┐
│  ┌──────────┐     │
│  │Container │     │
│  │          │     │
│  │ No Net   │     │
│  └──────────┘     │
│     ↕  localhost  │
│  (127.0.0.1 only) │
└────────────────────┘
      No Internet
```

**Example:**
```powershell
# Container with no network
docker run -d --name isolated --network none alpine sleep 3600

# Can't access anything
docker exec isolated ping google.com
# ❌ ERROR: No network
```

---

### 4️⃣ Default Bridge (Automatic)

**What:** Docker's built-in default network  
**When:** When you don't specify --network  
**How:** All containers without network go here  

```powershell
# These go on default bridge
docker run -d --name container1 nginx
docker run -d --name container2 nginx

# But they CAN'T communicate by name!
docker exec container1 ping container2
# ❌ ERROR: Can't resolve 'container2'

# They CAN communicate by IP address
docker inspect container2 | Select-String IPAddress
# Get IP, then:
docker exec container1 ping 172.17.0.3
# ✅ Works
```

**⚠️ Important:** Default bridge doesn't support DNS. Always create custom networks!

---

## 🔧 Container Communication - Step by Step

### Example: Web App + Database

Let me show you a complete real-world example:

```powershell
# Step 1: Create a custom network
docker network create webapp-network

# Step 2: Start database
docker run -d `
  --name mydb `
  --network webapp-network `
  -e POSTGRES_USER=admin `
  -e POSTGRES_PASSWORD=secret `
  -e POSTGRES_DB=myapp `
  postgres:15

# Wait for database to start
Start-Sleep -Seconds 10

# Step 3: Start backend that connects to database
# Create a simple Node.js server first
mkdir C:\networking-demo
cd C:\networking-demo

@"
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200, {'Content-Type': 'text/html'});
  res.end(\`
    <html>
    <body>
      <h1>Backend API</h1>
      <p>Connected to database at: mydb:5432</p>
      <p>Database host resolves by NAME!</p>
    </body>
    </html>
  \`);
});

server.listen(8000, '0.0.0.0', () => {
  console.log('Backend listening on 8000');
});
"@ | Out-File server.js -Encoding UTF8

# Run backend
docker run -d `
  --name backend `
  --network webapp-network `
  -v ${PWD}/server.js:/app/server.js `
  -w /app `
  node:18-alpine `
  node server.js

# Step 4: Test communication
# Backend can reach database by name!
docker exec backend ping mydb
# ✅ Works! DNS resolves 'mydb' to database container

# Step 5: Access backend from host
docker run -d `
  --name frontend `
  --network webapp-network `
  -p 3000:80 `
  nginx:alpine

# Only frontend is exposed to host (port 3000)
# Backend and database are isolated!
```

**What's happening:**

```
Internet
   ↓
Your Computer (Host)
   ↓ (port 3000)
┌─────────────────────────────────────────┐
│  webapp-network                         │
│                                         │
│  ┌──────────┐    ┌──────────┐         │
│  │ frontend │───►│ backend  │         │
│  │  :80     │    │  :8000   │         │
│  └──────────┘    └─────┬────┘         │
│                        │               │
│                        ↓               │
│                  ┌──────────┐         │
│                  │   mydb   │         │
│                  │  :5432   │         │
│                  └──────────┘         │
│                                         │
└─────────────────────────────────────────┘

Key Points:
• Frontend talks to backend using name "backend"
• Backend talks to database using name "mydb"
• Only frontend port 3000 is exposed to host
• Database and backend are hidden from outside
```

---

## 🎯 DNS and Service Discovery

### Magic of Container Names

When you put containers on the same network, **Docker provides DNS**:

```powershell
# Create network
docker network create mynet

# Start containers
docker run -d --name database --network mynet postgres:15
docker run -d --name api --network mynet node:18-alpine sleep 3600

# From 'api' container, you can use names!
docker exec api ping database
# Docker DNS resolves "database" → 172.18.0.2 (actual IP)

docker exec api nslookup database
# Shows: database resolves to internal IP
```

**Visual:**
```
Container: api
├─ Wants to connect to: "database"
├─ Asks Docker DNS: "Where is database?"
├─ Docker DNS replies: "172.18.0.2"
└─ api connects to 172.18.0.2
```

---

## 🏗️ Multi-Container Application - Complete Example

### 3-Tier Architecture (Frontend, Backend, Database)

Let me build a complete application:

```powershell
# Step 1: Create network
docker network create fullstack-network

# Step 2: Create project structure
mkdir C:\fullstack-demo
cd C:\fullstack-demo
mkdir frontend, backend
```

#### Create Backend

```powershell
# backend/server.js
@"
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200, {
    'Content-Type': 'application/json',
    'Access-Control-Allow-Origin': '*'
  });
  
  res.end(JSON.stringify({
    message: 'Hello from Backend!',
    database: 'Connected to: postgres:5432',
    timestamp: new Date().toISOString()
  }));
});

server.listen(8000, '0.0.0.0', () => {
  console.log('Backend API running on 8000');
});
"@ | Out-File backend\server.js -Encoding UTF8

@"
{
  "name": "backend",
  "version": "1.0.0",
  "main": "server.js"
}
"@ | Out-File backend\package.json -Encoding UTF8
```

#### Create Frontend

```powershell
# frontend/index.html
@"
<!DOCTYPE html>
<html>
<head>
    <title>Fullstack Docker App</title>
    <style>
        body {
            font-family: Arial;
            max-width: 800px;
            margin: 50px auto;
            padding: 20px;
        }
        .card {
            border: 2px solid #333;
            padding: 20px;
            margin: 20px 0;
            border-radius: 10px;
        }
        button {
            background: #007bff;
            color: white;
            border: none;
            padding: 10px 20px;
            border-radius: 5px;
            cursor: pointer;
        }
        button:hover { background: #0056b3; }
        #response { 
            background: #f0f0f0;
            padding: 15px;
            border-radius: 5px;
            margin-top: 10px;
        }
    </style>
</head>
<body>
    <h1>🐳 Docker Networking Demo</h1>
    
    <div class="card">
        <h2>Frontend Container</h2>
        <p>This is served by Nginx</p>
        <p>Port: 80 (exposed as 3000 on host)</p>
    </div>

    <div class="card">
        <h2>Test Backend Connection</h2>
        <button onclick="callBackend()">Call Backend API</button>
        <div id="response"></div>
    </div>

    <script>
        async function callBackend() {
            try {
                // Frontend talks to backend using container name!
                const response = await fetch('http://localhost:3001/');
                const data = await response.json();
                document.getElementById('response').innerHTML = 
                    '<pre>' + JSON.stringify(data, null, 2) + '</pre>';
            } catch (error) {
                document.getElementById('response').innerHTML = 
                    'Error: ' + error.message;
            }
        }
    </script>
</body>
</html>
"@ | Out-File frontend\index.html -Encoding UTF8
```

#### Start Everything

```powershell
# Database
docker run -d `
  --name postgres `
  --network fullstack-network `
  -e POSTGRES_USER=admin `
  -e POSTGRES_PASSWORD=secret `
  -e POSTGRES_DB=myapp `
  postgres:15

Write-Host "Database starting..." -ForegroundColor Cyan
Start-Sleep -Seconds 10

# Backend API
docker run -d `
  --name backend `
  --network fullstack-network `
  -v ${PWD}\backend:/app `
  -w /app `
  -p 3001:8000 `
  node:18-alpine `
  node server.js

Write-Host "Backend starting..." -ForegroundColor Cyan
Start-Sleep -Seconds 3

# Frontend
docker run -d `
  --name frontend `
  --network fullstack-network `
  -v ${PWD}\frontend:/usr/share/nginx/html `
  -p 3000:80 `
  nginx:alpine

Write-Host "Frontend starting..." -ForegroundColor Cyan
Start-Sleep -Seconds 2

# Test connectivity
Write-Host "`n=== Testing Container Communication ===" -ForegroundColor Green

Write-Host "`n1. Backend can reach Database:" -ForegroundColor Yellow
docker exec backend ping -c 2 postgres

Write-Host "`n2. Frontend can reach Backend:" -ForegroundColor Yellow
docker exec frontend ping -c 2 backend

Write-Host "`n✅ All containers are connected!" -ForegroundColor Green
Write-Host "`n🌐 Open http://localhost:3000 in your browser" -ForegroundColor Cyan
Write-Host "🔌 Backend API: http://localhost:3001" -ForegroundColor Cyan

# Open browser
Start-Process "http://localhost:3000"
```

**Architecture:**
```
┌─────────────────────────────────────────────┐
│     fullstack-network (172.18.0.0/16)       │
│                                             │
│  ┌─────────────┐                           │
│  │  frontend   │ :80 (exposed as 3000)     │
│  │   Nginx     │                           │
│  └──────┬──────┘                           │
│         │ Talks to "backend"               │
│         ↓                                   │
│  ┌─────────────┐                           │
│  │  backend    │ :8000 (exposed as 3001)   │
│  │  Node.js    │                           │
│  └──────┬──────┘                           │
│         │ Talks to "postgres"              │
│         ↓                                   │
│  ┌─────────────┐                           │
│  │  postgres   │ :5432 (NOT exposed)       │
│  │  Database   │                           │
│  └─────────────┘                           │
│                                             │
└─────────────────────────────────────────────┘

Host Access:
• Frontend: localhost:3000 ✅
• Backend: localhost:3001 ✅
• Database: NOT accessible from host ✅ (Security!)
```

---

## 🔍 Network Commands Reference

### List Networks

```powershell
# Show all networks
docker network ls

# Example output:
# NETWORK ID     NAME                DRIVER    SCOPE
# abc123def456   bridge             bridge    local
# def789ghi012   fullstack-network  bridge    local
# ghi345jkl678   host               host      local
# jkl901mno234   none               null      local
```

### Inspect Network

```powershell
# See detailed information
docker network inspect fullstack-network

# Output shows:
# - Subnet: 172.18.0.0/16
# - Gateway: 172.18.0.1
# - Connected containers and their IPs
```

**Example output:**
```json
[
    {
        "Name": "fullstack-network",
        "Driver": "bridge",
        "Containers": {
            "abc123": {
                "Name": "frontend",
                "IPv4Address": "172.18.0.2/16"
            },
            "def456": {
                "Name": "backend",
                "IPv4Address": "172.18.0.3/16"
            },
            "ghi789": {
                "Name": "postgres",
                "IPv4Address": "172.18.0.4/16"
            }
        }
    }
]
```

### Connect/Disconnect Containers

```powershell
# Add container to network
docker network connect fullstack-network my-container

# Remove container from network
docker network disconnect fullstack-network my-container

# Container can be on multiple networks!
docker network connect network1 my-container
docker network connect network2 my-container
```

### Create Networks with Options

```powershell
# Create with custom subnet
docker network create --subnet=192.168.100.0/24 my-custom-network

# Create with specific driver
docker network create --driver bridge my-bridge

# Create with gateway
docker network create --subnet=10.0.0.0/24 --gateway=10.0.0.1 my-net
```

### Remove Networks

```powershell
# Remove specific network
docker network rm fullstack-network

# Remove all unused networks
docker network prune

# Force remove (skip confirmation)
docker network prune -f
```

---

## 🎯 Practical Exercises

### Exercise 1: WordPress with MySQL

```powershell
# Create network
docker network create wordpress-net

# Run MySQL
docker run -d `
  --name mysql `
  --network wordpress-net `
  -e MYSQL_ROOT_PASSWORD=rootpass `
  -e MYSQL_DATABASE=wordpress `
  -e MYSQL_USER=wpuser `
  -e MYSQL_PASSWORD=wppass `
  mysql:8.0

Write-Host "Waiting for MySQL..." -ForegroundColor Yellow
Start-Sleep -Seconds 15

# Run WordPress
docker run -d `
  --name wordpress `
  --network wordpress-net `
  -p 8080:80 `
  -e WORDPRESS_DB_HOST=mysql `
  -e WORDPRESS_DB_USER=wpuser `
  -e WORDPRESS_DB_PASSWORD=wppass `
  -e WORDPRESS_DB_NAME=wordpress `
  wordpress:latest

Write-Host "WordPress starting..." -ForegroundColor Yellow
Start-Sleep -Seconds 10

Write-Host "✅ WordPress available at http://localhost:8080" -ForegroundColor Green
Start-Process "http://localhost:8080"

# Test connection
docker exec wordpress ping -c 2 mysql
# ✅ WordPress can reach MySQL by name!
```

### Exercise 2: Isolated Networks (Security)

```powershell
# Create two separate networks
docker network create frontend-net
docker network create backend-net

# Public-facing web server
docker run -d `
  --name public-web `
  --network frontend-net `
  -p 8080:80 `
  nginx:alpine

# Internal API (on backend network only)
docker run -d `
  --name private-api `
  --network backend-net `
  node:18-alpine `
  sleep 3600

# Database (on backend network only)
docker run -d `
  --name database `
  --network backend-net `
  postgres:15

# Try to access from public-web to database
docker exec public-web ping database
# ❌ Cannot reach! They're on different networks (isolated)

# But if we add public-web to backend network too:
docker network connect backend-net public-web

# Now it works:
docker exec public-web ping database
# ✅ Can reach!

# This shows network isolation for security
```

### Exercise 3: Service Discovery

```powershell
# Create network
docker network create service-net

# Run multiple backend instances
docker run -d --name api-1 --network service-net node:18-alpine sleep 3600
docker run -d --name api-2 --network service-net node:18-alpine sleep 3600
docker run -d --name api-3 --network service-net node:18-alpine sleep 3600

# Run client
docker run -d --name client --network service-net alpine sleep 3600

# Client can discover all services
docker exec client ping api-1
docker exec client ping api-2
docker exec client ping api-3
# ✅ All resolve by name!

# Inspect to see IPs
docker network inspect service-net
```

---

## 📊 Network Comparison Table

| Feature | Bridge (Custom) | Bridge (Default) | Host | None |
|---------|----------------|------------------|------|------|
| DNS by name | ✅ Yes | ❌ No | N/A | N/A |
| Container isolation | ✅ Yes | ⚠️ Partial | ❌ No | ✅ Full |
| Port mapping needed | ✅ Yes | ✅ Yes | ❌ No | N/A |
| Best for | Apps | Legacy | Performance | Batch jobs |
| Security | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ |

---

## 🛡️ Security Best Practices

### 1. Use Custom Networks (Not Default)

```powershell
# ❌ Bad: Default bridge, no DNS
docker run -d --name web nginx

# ✅ Good: Custom network, has DNS
docker network create mynet
docker run -d --name web --network mynet nginx
```

### 2. Isolate Sensitive Services

```powershell
# Database on internal network only
docker network create internal
docker run -d --name db --network internal postgres

# Public services can't reach it
docker run -d --name public -p 80:80 nginx
docker exec public ping db
# ❌ Cannot reach (different networks)
```

### 3. Expose Only What's Needed

```powershell
# ❌ Bad: Everything exposed
docker run -d -p 5432:5432 postgres  # Database exposed!

# ✅ Good: Only frontend exposed
docker network create app-net
docker run -d --name db --network app-net postgres  # Not exposed
docker run -d --name api --network app-net my-api   # Not exposed
docker run -d --name web --network app-net -p 80:80 nginx  # Only this exposed
```

---

## ✅ Summary

**Key Concepts:**

1. **Networks = Communication channels**
   - Containers on same network can talk
   - Use container names (DNS)

2. **Types:**
   - **Bridge (custom)**: Most common, has DNS
   - **Host**: Direct host access
   - **None**: No network
   - **Default bridge**: Legacy, no DNS

3. **Best Practices:**
   - Always create custom networks
   - Isolate sensitive services
   - Expose only public-facing containers

**Essential Commands:**
```powershell
docker network create mynet       # Create network
docker network ls                 # List networks
docker network inspect mynet      # View details
docker network rm mynet           # Delete network
docker run --network mynet ...    # Use network
```

**Next:** Day 8 - Docker Compose (Makes networking even easier!) 🚀

Want to build a real multi-tier application with proper networking? Let me know! 😊