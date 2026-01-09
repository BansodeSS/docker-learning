# Day 8: Docker Compose - Complete Deep Dive

## 🎯 What is Docker Compose and Why Do We Need It?

### The Problem Without Docker Compose

**Imagine running a web app with database, cache, and API:**

```powershell
# You have to type all these commands manually:

# 1. Create network
docker network create myapp-network

# 2. Create volumes
docker volume create postgres-data
docker volume create redis-data

# 3. Start database
docker run -d `
  --name postgres `
  --network myapp-network `
  -e POSTGRES_USER=user `
  -e POSTGRES_PASSWORD=pass `
  -e POSTGRES_DB=myapp `
  -v postgres-data:/var/lib/postgresql/data `
  postgres:15

# 4. Start Redis
docker run -d `
  --name redis `
  --network myapp-network `
  -v redis-data:/data `
  redis:alpine

# 5. Start backend
docker run -d `
  --name backend `
  --network myapp-network `
  -e DATABASE_URL=postgresql://user:pass@postgres:5432/myapp `
  -e REDIS_URL=redis://redis:6379 `
  -v ${PWD}/backend:/app `
  -p 8000:8000 `
  my-backend-image

# 6. Start frontend
docker run -d `
  --name frontend `
  --network myapp-network `
  -e API_URL=http://backend:8000 `
  -p 3000:3000 `
  my-frontend-image

# 😰 That's a LOT of commands!
# 😰 Hard to remember
# 😰 Error-prone
# 😰 Can't easily share with team
```

### The Solution: Docker Compose

**Put everything in ONE file:**

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: myapp
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:alpine
    volumes:
      - redis-data:/data

  backend:
    build: ./backend
    environment:
      DATABASE_URL: postgresql://user:pass@postgres:5432/myapp
      REDIS_URL: redis://redis:6379
    volumes:
      - ./backend:/app
    ports:
      - "8000:8000"
    depends_on:
      - postgres
      - redis

  frontend:
    build: ./frontend
    environment:
      API_URL: http://backend:8000
    ports:
      - "3000:3000"
    depends_on:
      - backend

volumes:
  postgres-data:
  redis-data:
```

**Then run ONE command:**
```powershell
docker compose up
```

**🎉 Everything starts automatically!**

---

## 📚 Understanding docker-compose.yml

### Structure Breakdown

```yaml
version: '3.8'           # ← Docker Compose file version

services:                # ← Define your containers here
  service1:              # ← Container name
    image: nginx         # ← What image to use
    ports:               # ← Port mapping
      - "8080:80"
  
  service2:
    build: ./app         # ← Build from Dockerfile
    
volumes:                 # ← Named volumes
  data:                  # ← Volume name

networks:                # ← Custom networks
  mynetwork:             # ← Network name
```

### Line-by-Line Explanation

```yaml
version: '3.8'
# ↑ Specifies Docker Compose file format version
# Use '3.8' or higher (latest features)
# Don't worry too much about this, just use 3.8

services:
# ↑ This section defines all your containers
# Each container is called a "service"

  web:
  # ↑ Name of your service (becomes container name)
  # You can use this name for networking
  
    image: nginx:alpine
    # ↑ Which Docker image to use
    # Format: image:tag
    
    ports:
      - "8080:80"
    # ↑ Port mapping: "host:container"
    # Maps port 8080 on your computer to port 80 in container
    
    volumes:
      - ./html:/usr/share/nginx/html
    # ↑ Volume mounting
    # Maps ./html folder to /usr/share/nginx/html in container
    
    environment:
      - NODE_ENV=production
    # ↑ Environment variables
    # Key-value pairs passed to container
    
    depends_on:
      - db
    # ↑ Start order: db must start before web
    # Note: Doesn't wait for db to be "ready", just "started"

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db-data:/var/lib/postgresql/data
    # ↑ Named volume (defined in volumes section below)

volumes:
  db-data:
  # ↑ Named volumes that persist data
  # Docker manages where these are stored

networks:
  default:
  # ↑ Custom network (optional)
  # By default, Compose creates a network for you
```

---

## 🚀 Simple Example - Step by Step

### Example 1: Static Website + Database

```powershell
# Step 1: Create project directory
mkdir C:\compose-demo
cd C:\compose-demo

# Step 2: Create HTML file
mkdir html
@"
<!DOCTYPE html>
<html>
<head>
    <title>Compose Demo</title>
    <style>
        body {
            font-family: Arial;
            max-width: 800px;
            margin: 50px auto;
            padding: 20px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
        }
        .card {
            background: rgba(255,255,255,0.1);
            padding: 20px;
            border-radius: 10px;
            margin: 20px 0;
        }
    </style>
</head>
<body>
    <h1>🐳 Docker Compose Demo</h1>
    <div class="card">
        <h2>Services Running:</h2>
        <ul>
            <li>✅ Nginx Web Server</li>
            <li>✅ PostgreSQL Database</li>
        </ul>
    </div>
    <div class="card">
        <p>All started with one command:</p>
        <code>docker compose up</code>
    </div>
</body>
</html>
"@ | Out-File html\index.html -Encoding UTF8

# Step 3: Create docker-compose.yml
@"
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
"@ | Out-File docker-compose.yml -Encoding UTF8

# Step 4: Start everything!
docker compose up -d

# Step 5: Check status
docker compose ps

# Step 6: Open browser
Start-Process "http://localhost:8080"

# Step 7: Check logs
docker compose logs

# Step 8: Stop everything
docker compose down
```

**What just happened?**

```
docker compose up
        ↓
┌───────────────────────────────┐
│ 1. Creates network            │
│    "compose-demo_default"     │
├───────────────────────────────┤
│ 2. Creates volume             │
│    "db-data"                  │
├───────────────────────────────┤
│ 3. Starts db container        │
│    postgres:15                │
├───────────────────────────────┤
│ 4. Starts web container       │
│    nginx:alpine               │
│    (waits for db via          │
│     depends_on)               │
└───────────────────────────────┘
```

---

## 🏗️ Full Stack Application - Complete Example

Let's build a real **3-tier application** with Compose!

### Architecture
```
Frontend (React/HTML)
        ↓
Backend (Node.js API)
        ↓
Database (PostgreSQL)
```

### Step 1: Create Project Structure

```powershell
# Create directories
mkdir C:\fullstack-compose
cd C:\fullstack-compose
mkdir frontend, backend

Write-Host "✅ Project structure created" -ForegroundColor Green
```

### Step 2: Create Backend

```powershell
# backend/server.js
@"
const http = require('http');
const url = require('url');

const server = http.createServer((req, res) => {
  const pathname = url.parse(req.url).pathname;
  
  // CORS headers
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Content-Type', 'application/json');
  
  if (pathname === '/api/health') {
    res.writeHead(200);
    res.end(JSON.stringify({
      status: 'healthy',
      service: 'backend',
      timestamp: new Date().toISOString(),
      database: 'postgres://db:5432'
    }));
  } else if (pathname === '/api/data') {
    res.writeHead(200);
    res.end(JSON.stringify({
      message: 'Hello from Backend!',
      users: ['Alice', 'Bob', 'Charlie'],
      timestamp: new Date().toISOString()
    }));
  } else {
    res.writeHead(404);
    res.end(JSON.stringify({ error: 'Not found' }));
  }
});

server.listen(8000, '0.0.0.0', () => {
  console.log('Backend API running on port 8000');
  console.log('Database: ' + (process.env.DATABASE_URL || 'not configured'));
});
"@ | Out-File backend\server.js -Encoding UTF8

# backend/package.json
@"
{
  "name": "backend",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  }
}
"@ | Out-File backend\package.json -Encoding UTF8

# backend/Dockerfile
@"
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 8000
CMD ["npm", "start"]
"@ | Out-File backend\Dockerfile -Encoding UTF8

Write-Host "✅ Backend created" -ForegroundColor Green
```

### Step 3: Create Frontend

```powershell
# frontend/index.html
@"
<!DOCTYPE html>
<html>
<head>
    <title>Fullstack Compose App</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: 'Segoe UI', Arial, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            padding: 20px;
        }
        .container {
            max-width: 900px;
            margin: 0 auto;
        }
        h1 {
            color: white;
            text-align: center;
            margin: 40px 0;
            font-size: 2.5em;
        }
        .card {
            background: white;
            border-radius: 15px;
            padding: 30px;
            margin: 20px 0;
            box-shadow: 0 10px 30px rgba(0,0,0,0.3);
        }
        button {
            background: #667eea;
            color: white;
            border: none;
            padding: 12px 24px;
            border-radius: 8px;
            cursor: pointer;
            font-size: 16px;
            margin: 5px;
            transition: all 0.3s;
        }
        button:hover {
            background: #5568d3;
            transform: translateY(-2px);
        }
        .response {
            background: #f5f5f5;
            padding: 20px;
            border-radius: 10px;
            margin-top: 15px;
            font-family: monospace;
            white-space: pre-wrap;
        }
        .status {
            display: inline-block;
            padding: 5px 15px;
            border-radius: 20px;
            font-size: 14px;
            margin: 5px;
        }
        .status.online { background: #4caf50; color: white; }
        .status.offline { background: #f44336; color: white; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🐳 Fullstack Docker Compose App</h1>
        
        <div class="card">
            <h2>Service Status</h2>
            <div id="status"></div>
        </div>

        <div class="card">
            <h2>Test Backend API</h2>
            <button onclick="checkHealth()">Check Health</button>
            <button onclick="getData()">Get Data</button>
            <div id="response" class="response" style="display:none;"></div>
        </div>

        <div class="card">
            <h2>Architecture</h2>
            <p>🌐 <strong>Frontend:</strong> This page (Nginx)</p>
            <p>⚙️ <strong>Backend:</strong> Node.js API (port 8000)</p>
            <p>🗄️ <strong>Database:</strong> PostgreSQL (internal)</p>
            <p style="margin-top: 15px; color: #666;">
                All services communicate via Docker network!
            </p>
        </div>
    </div>

    <script>
        const API_URL = 'http://localhost:8000';

        async function checkHealth() {
            showResponse('Checking backend health...');
            try {
                const res = await fetch(API_URL + '/api/health');
                const data = await res.json();
                showResponse(JSON.stringify(data, null, 2));
                updateStatus('online');
            } catch (err) {
                showResponse('Error: ' + err.message);
                updateStatus('offline');
            }
        }

        async function getData() {
            showResponse('Fetching data from backend...');
            try {
                const res = await fetch(API_URL + '/api/data');
                const data = await res.json();
                showResponse(JSON.stringify(data, null, 2));
            } catch (err) {
                showResponse('Error: ' + err.message);
            }
        }

        function showResponse(text) {
            const div = document.getElementById('response');
            div.textContent = text;
            div.style.display = 'block';
        }

        function updateStatus(status) {
            const statusDiv = document.getElementById('status');
            if (status === 'online') {
                statusDiv.innerHTML = '<span class="status online">✓ Backend Online</span>';
            } else {
                statusDiv.innerHTML = '<span class="status offline">✗ Backend Offline</span>';
            }
        }

        // Check status on load
        window.onload = () => checkHealth();
    </script>
</body>
</html>
"@ | Out-File frontend\index.html -Encoding UTF8

# frontend/Dockerfile
@"
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
"@ | Out-File frontend\Dockerfile -Encoding UTF8

Write-Host "✅ Frontend created" -ForegroundColor Green
```

### Step 4: Create docker-compose.yml

```powershell
@"
version: '3.8'

services:
  # Frontend Service
  frontend:
    build: ./frontend
    ports:
      - "3000:80"
    depends_on:
      - backend
    networks:
      - app-network

  # Backend API Service
  backend:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/myapp
      - NODE_ENV=development
    depends_on:
      - db
    volumes:
      - ./backend:/app
      - /app/node_modules
    networks:
      - app-network

  # Database Service
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: myapp
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - app-network

  # Redis Cache (bonus!)
  redis:
    image: redis:alpine
    networks:
      - app-network

volumes:
  postgres-data:

networks:
  app-network:
    driver: bridge
"@ | Out-File docker-compose.yml -Encoding UTF8

Write-Host "✅ docker-compose.yml created" -ForegroundColor Green
```

### Step 5: Start Everything!

```powershell
# Build and start all services
Write-Host "`n🚀 Starting all services..." -ForegroundColor Cyan
docker compose up -d --build

# Wait for services to start
Write-Host "`n⏳ Waiting for services to start..." -ForegroundColor Yellow
Start-Sleep -Seconds 10

# Show status
Write-Host "`n📊 Service Status:" -ForegroundColor Green
docker compose ps

# Show logs
Write-Host "`n📜 Recent Logs:" -ForegroundColor Green
docker compose logs --tail=10

# Open browser
Write-Host "`n🌐 Opening browser..." -ForegroundColor Cyan
Start-Process "http://localhost:3000"

Write-Host "`n✅ All services are running!" -ForegroundColor Green
Write-Host "Frontend: http://localhost:3000" -ForegroundColor Cyan
Write-Host "Backend API: http://localhost:8000" -ForegroundColor Cyan
Write-Host "`nPress Ctrl+C to stop" -ForegroundColor Yellow
```

**What's Running:**
```
┌─────────────────────────────────────┐
│  app-network (Custom Bridge)        │
│                                     │
│  ┌──────────┐                      │
│  │ frontend │ :80 → 3000           │
│  │  Nginx   │                      │
│  └────┬─────┘                      │
│       │ depends_on                  │
│       ↓                             │
│  ┌──────────┐                      │
│  │ backend  │ :8000 → 8000         │
│  │ Node.js  │                      │
│  └────┬─────┘                      │
│       │ depends_on                  │
│       ↓                             │
│  ┌──────────┐      ┌──────────┐   │
│  │    db    │      │  redis   │   │
│  │PostgreSQL│      │  Cache   │   │
│  └──────────┘      └──────────┘   │
│                                     │
└─────────────────────────────────────┘
```

---

## 🎮 Docker Compose Commands

### Starting and Stopping

```powershell
# Start all services (foreground)
docker compose up

# Start all services (background/detached)
docker compose up -d

# Start and rebuild images
docker compose up --build

# Start specific service
docker compose up frontend

# Stop all services (keeps containers)
docker compose stop

# Stop and remove containers, networks
docker compose down

# Stop and remove everything including volumes
docker compose down -v

# Stop and remove including images
docker compose down --rmi all
```

### Viewing Information

```powershell
# List running services
docker compose ps

# List all services (including stopped)
docker compose ps -a

# View logs from all services
docker compose logs

# Follow logs in real-time
docker compose logs -f

# View logs from specific service
docker compose logs backend

# View last 20 lines
docker compose logs --tail=20

# View logs with timestamps
docker compose logs -t
```

### Managing Services

```powershell
# Restart all services
docker compose restart

# Restart specific service
docker compose restart backend

# Pause services
docker compose pause

# Unpause services
docker compose unpause

# Remove stopped containers
docker compose rm

# Execute command in running service
docker compose exec backend sh

# Execute one-off command
docker compose run backend npm install
```

### Scaling Services

```powershell
# Scale backend to 3 instances
docker compose up -d --scale backend=3

# Scale multiple services
docker compose up -d --scale backend=3 --scale frontend=2
```

### Building

```powershell
# Build all services
docker compose build

# Build specific service
docker compose build backend

# Build with no cache
docker compose build --no-cache

# Pull images without starting
docker compose pull
```

---

## 📊 Understanding depends_on

```yaml
services:
  web:
    depends_on:
      - db
  # ↑ Means: Start db BEFORE web
  
  db:
    image: postgres
```

**Important:** `depends_on` only controls START order, NOT readiness!

```powershell
# What actually happens:
# 1. Docker starts db container
# 2. Immediately starts web container
# 3. Web might fail if db isn't ready yet!
```

**Solution: Health Checks (Advanced)**

```yaml
services:
  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 5

  backend:
    depends_on:
      db:
        condition: service_healthy
    # ↑ Now waits for db to be HEALTHY, not just started
```

---

## 🔧 Environment Variables

### Three Ways to Set Environment Variables

#### 1. Inline in docker-compose.yml

```yaml
services:
  app:
    environment:
      - NODE_ENV=production
      - API_KEY=abc123
      - DATABASE_URL=postgres://db:5432
```

#### 2. From .env File

```powershell
# Create .env file
@"
NODE_ENV=production
API_KEY=abc123
DATABASE_URL=postgres://db:5432
"@ | Out-File .env -Encoding UTF8
```

```yaml
# docker-compose.yml
services:
  app:
    env_file:
      - .env
```

#### 3. From Host Environment

```yaml
services:
  app:
    environment:
      - API_KEY=${API_KEY}
      # ↑ Uses API_KEY from your computer's environment
```

---

## 🎯 Practical Example: WordPress

### Complete WordPress + MySQL Setup

```powershell
# Create directory
mkdir C:\wordpress-compose
cd C:\wordpress-compose

# Create docker-compose.yml
@"
version: '3.8'

services:
  wordpress:
    image: wordpress:latest
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wordpress-data:/var/www/html
    depends_on:
      - db

  db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass
      MYSQL_ROOT_PASSWORD: rootpass
    volumes:
      - db-data:/var/lib/mysql

volumes:
  wordpress-data:
  db-data:
"@ | Out-File docker-compose.yml -Encoding UTF8

# Start WordPress
docker compose up -d

# Wait for startup
Write-Host "⏳ Starting WordPress..." -ForegroundColor Yellow
Start-Sleep -Seconds 20

# Open WordPress
Start-Process "http://localhost:8080"

Write-Host "✅ WordPress ready!" -ForegroundColor Green
Write-Host "URL: http://localhost:8080" -ForegroundColor Cyan
```

**In 3 minutes, you have a full WordPress site!** 🎉

---

## 📋 Development vs Production

### Development Setup

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app              # Mount source code
      - /app/node_modules   # Don't overwrite node_modules
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - DEBUG=true
    command: npm run dev    # Hot reload
```

### Production Setup

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.prod
    # No volume mounts (use built image)
    ports:
      - "80:80"
    environment:
      - NODE_ENV=production
    restart: unless-stopped
```

---

## ✅ Best Practices

### DO ✅

1. **Use .env files for secrets**
   ```yaml
   env_file:
     - .env
   ```

2. **Name your networks and volumes**
   ```yaml
   networks:
     app-net:
       name: myapp-network
   ```

3. **Use depends_on for order**
   ```yaml
   depends_on:
     - db
   ```

4. **Add health checks**
   ```yaml
   healthcheck:
     test: ["CMD", "curl", "-f", "http://localhost"]
   ```

### DON'T ❌

1. **Don't put secrets in docker-compose.yml**
   ```yaml
   # ❌ Bad
   environment:
     - PASSWORD=secret123
   
   # ✅ Good
   env_file:
     - .env
   ```

2. **Don't use latest tag in production**
   ```yaml
   # ❌ Bad
   image: nginx:latest
   
   # ✅ Good
   image: nginx:1.24-alpine
   ```

---

## 🎓 Summary

**Docker Compose = Multi-container made easy!**

**Key Concepts:**
1. **One file** (`docker-compose.yml`) defines everything
2. **One command** (`docker compose up`) starts everything
3. **Services** = Your containers
4. **Networks** = Automatic container communication
5. **Volumes** = Data persistence
6. **depends_on** = Start order

**Essential Commands:**
```powershell
docker compose up -d        # Start
docker compose down         # Stop
docker compose ps           # Status
docker compose logs -f      # Logs
docker compose restart      # Restart
```

**Next:** Day 9 - Build real projects with Compose! 🚀

---

Ready to build your own multi-container application? What would you like to create:
1. Blog platform (WordPress-style)
2. E-commerce site
3. Social media app
4. Your own idea?

Let me know! 😊