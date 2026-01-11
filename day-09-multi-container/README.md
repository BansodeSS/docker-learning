# Day 9: Multi-Container Blog Platform - Complete Guide

## 🎯 What We're Building

A **complete blog platform** with 5 services working together:

```
User Browser
     ↓
┌────────────────────────────────────────────┐
│  Nginx (Reverse Proxy)                     │
│  Routes traffic                            │
└──────┬─────────────────────┬───────────────┘
       │                     │
       ↓                     ↓
┌──────────────┐      ┌──────────────┐
│  Frontend    │      │  Backend     │
│  (React/HTML)│      │  (Python API)│
└──────────────┘      └──────┬───────┘
                             │
              ┌──────────────┴──────────────┐
              ↓                             ↓
       ┌──────────────┐              ┌──────────────┐
       │  Database    │              │  Redis       │
       │  (PostgreSQL)│              │  (Cache)     │
       └──────────────┘              └──────────────┘
```

**Why 5 services?**
- **Nginx**: Handles incoming requests, routes to frontend/backend
- **Frontend**: User interface (HTML/React)
- **Backend**: API for data operations
- **Database**: Stores blog posts, users
- **Redis**: Caches frequently accessed data

---

## 📁 Understanding Project Structure

```
blog-platform/
├── docker-compose.yml          ← Orchestrates everything
├── nginx/
│   └── nginx.conf              ← Proxy configuration
├── frontend/
│   ├── Dockerfile              ← How to build frontend
│   ├── package.json            ← Frontend dependencies
│   └── index.html              ← Frontend code
├── backend/
│   ├── Dockerfile              ← How to build backend
│   ├── requirements.txt        ← Python dependencies
│   └── main.py                 ← Backend API code
└── .env                        ← Environment variables
```

---

## 🚀 Building the Complete Project

### Step 1: Create Project Structure

```powershell
# Create main directory
New-Item -ItemType Directory -Path "C:\blog-platform" -Force
cd C:\blog-platform

# Create subdirectories
New-Item -ItemType Directory -Path "nginx", "frontend", "backend" -Force

Write-Host "✅ Project structure created" -ForegroundColor Green
```

---

### Step 2: Create Nginx Configuration

**What is Nginx doing here?**
- Acts as a **reverse proxy**
- Routes `/` → Frontend
- Routes `/api` → Backend
- Handles CORS, SSL (in production)

```powershell
# Create nginx.conf
@"
events {
    worker_connections 1024;
}

http {
    # Upstream for backend API
    upstream backend {
        server backend:8000;
    }

    # Upstream for frontend
    upstream frontend {
        server frontend:80;
    }

    server {
        listen 80;
        server_name localhost;

        # Frontend - serve all non-API requests
        location / {
            proxy_pass http://frontend;
            proxy_set_header Host `$host;
            proxy_set_header X-Real-IP `$remote_addr;
        }

        # Backend API - proxy /api requests
        location /api {
            # Remove /api prefix before forwarding
            rewrite ^/api/(.*) /`$1 break;
            
            proxy_pass http://backend;
            proxy_set_header Host `$host;
            proxy_set_header X-Real-IP `$remote_addr;
            
            # CORS headers
            add_header Access-Control-Allow-Origin *;
            add_header Access-Control-Allow-Methods 'GET, POST, PUT, DELETE, OPTIONS';
            add_header Access-Control-Allow-Headers 'Content-Type, Authorization';
        }

        # Health check endpoint
        location /health {
            access_log off;
            return 200 'healthy';
            add_header Content-Type text/plain;
        }
    }
}
"@ | Out-File -FilePath "nginx\nginx.conf" -Encoding UTF8

Write-Host "✅ Nginx config created" -ForegroundColor Green
```

**Breaking down nginx.conf:**

```nginx
upstream backend {
    server backend:8000;
}
# ↑ Defines backend service
# "backend" is the service name from docker-compose.yml
# Docker DNS resolves it automatically!

location /api {
    rewrite ^/api/(.*) /$1 break;
    proxy_pass http://backend;
}
# ↑ Routes /api/* to backend
# Strips "/api" prefix before forwarding
# Example: /api/posts → backend:8000/posts

location / {
    proxy_pass http://frontend;
}
# ↑ Routes everything else to frontend
```

---

### Step 3: Create Backend (Python API)

```powershell
# Create backend/main.py
@"
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List
import os
from datetime import datetime

app = FastAPI(title="Blog API")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# In-memory storage (in real app, use database)
posts = []

class Post(BaseModel):
    id: int = None
    title: str
    content: str
    author: str
    created_at: str = None

@app.get("/")
def read_root():
    return {
        "message": "Blog API",
        "version": "1.0",
        "database": os.getenv("DATABASE_URL", "Not configured"),
        "redis": os.getenv("REDIS_URL", "Not configured")
    }

@app.get("/health")
def health_check():
    return {"status": "healthy"}

@app.get("/posts", response_model=List[Post])
def get_posts():
    return posts

@app.get("/posts/{post_id}")
def get_post(post_id: int):
    post = next((p for p in posts if p['id'] == post_id), None)
    if not post:
        raise HTTPException(status_code=404, detail="Post not found")
    return post

@app.post("/posts", response_model=Post)
def create_post(post: Post):
    post.id = len(posts) + 1
    post.created_at = datetime.now().isoformat()
    posts.append(post.dict())
    return post

@app.delete("/posts/{post_id}")
def delete_post(post_id: int):
    global posts
    posts = [p for p in posts if p['id'] != post_id]
    return {"message": "Post deleted"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
"@ | Out-File -FilePath "backend\main.py" -Encoding UTF8

# Create backend/requirements.txt
@"
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.5.0
python-multipart==0.0.6
"@ | Out-File -FilePath "backend\requirements.txt" -Encoding UTF8

# Create backend/Dockerfile
@"
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Expose port
EXPOSE 8000

# Run the application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
"@ | Out-File -FilePath "backend\Dockerfile" -Encoding UTF8

Write-Host "✅ Backend created" -ForegroundColor Green
```

**Understanding the Backend:**

```python
@app.get("/posts")
def get_posts():
    return posts
# ↑ API endpoint: GET /posts
# Returns list of all blog posts

@app.post("/posts")
def create_post(post: Post):
    # Add post to list
    return post
# ↑ API endpoint: POST /posts
# Creates new blog post
```

---

### Step 4: Create Frontend (HTML + JavaScript)

```powershell
# Create frontend/index.html
@"
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Blog Platform</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            padding: 20px;
        }
        .container {
            max-width: 1000px;
            margin: 0 auto;
        }
        h1 {
            color: white;
            text-align: center;
            margin: 40px 0;
            font-size: 3em;
        }
        .status {
            background: rgba(255,255,255,0.1);
            padding: 15px;
            border-radius: 10px;
            color: white;
            margin-bottom: 20px;
            text-align: center;
        }
        .status.online { background: rgba(76, 175, 80, 0.3); }
        .status.offline { background: rgba(244, 67, 54, 0.3); }
        .form-card {
            background: white;
            padding: 30px;
            border-radius: 15px;
            box-shadow: 0 10px 30px rgba(0,0,0,0.3);
            margin-bottom: 30px;
        }
        .form-card h2 {
            margin-bottom: 20px;
            color: #333;
        }
        input, textarea {
            width: 100%;
            padding: 12px;
            margin: 10px 0;
            border: 2px solid #ddd;
            border-radius: 8px;
            font-size: 16px;
            font-family: inherit;
        }
        textarea {
            min-height: 150px;
            resize: vertical;
        }
        button {
            background: #667eea;
            color: white;
            border: none;
            padding: 12px 30px;
            border-radius: 8px;
            font-size: 16px;
            cursor: pointer;
            transition: all 0.3s;
        }
        button:hover {
            background: #5568d3;
            transform: translateY(-2px);
        }
        .posts {
            display: grid;
            gap: 20px;
        }
        .post {
            background: white;
            padding: 25px;
            border-radius: 15px;
            box-shadow: 0 5px 15px rgba(0,0,0,0.2);
        }
        .post h3 {
            color: #667eea;
            margin-bottom: 10px;
        }
        .post-meta {
            color: #999;
            font-size: 14px;
            margin-bottom: 15px;
        }
        .post-content {
            color: #333;
            line-height: 1.6;
        }
        .delete-btn {
            background: #f44336;
            padding: 8px 16px;
            font-size: 14px;
            margin-top: 10px;
        }
        .delete-btn:hover {
            background: #d32f2f;
        }
        .architecture {
            background: rgba(255,255,255,0.1);
            padding: 20px;
            border-radius: 10px;
            color: white;
            margin-top: 30px;
        }
        .architecture h2 {
            margin-bottom: 15px;
        }
        .architecture ul {
            list-style: none;
        }
        .architecture li {
            padding: 8px 0;
            border-bottom: 1px solid rgba(255,255,255,0.2);
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🐳 Docker Blog Platform</h1>
        
        <div id="status" class="status">
            <span id="status-text">Checking backend...</span>
        </div>

        <div class="form-card">
            <h2>Create New Post</h2>
            <input type="text" id="title" placeholder="Post Title" />
            <input type="text" id="author" placeholder="Author Name" />
            <textarea id="content" placeholder="Write your post content here..."></textarea>
            <button onclick="createPost()">Publish Post</button>
        </div>

        <div class="form-card">
            <h2>All Posts</h2>
            <button onclick="loadPosts()">Refresh Posts</button>
            <div id="posts" class="posts" style="margin-top: 20px;"></div>
        </div>

        <div class="architecture">
            <h2>System Architecture</h2>
            <ul>
                <li>🌐 <strong>Nginx:</strong> Reverse proxy (port 80)</li>
                <li>🎨 <strong>Frontend:</strong> This page (HTML/JS)</li>
                <li>⚙️ <strong>Backend:</strong> Python FastAPI (port 8000)</li>
                <li>🗄️ <strong>Database:</strong> PostgreSQL (internal)</li>
                <li>⚡ <strong>Redis:</strong> Cache (internal)</li>
            </ul>
        </div>
    </div>

    <script>
        const API_URL = '/api';

        // Check backend health
        async function checkHealth() {
            try {
                const res = await fetch(API_URL + '/');
                const data = await res.json();
                document.getElementById('status').className = 'status online';
                document.getElementById('status-text').textContent = 
                    '✓ Backend Online - ' + data.message;
            } catch (err) {
                document.getElementById('status').className = 'status offline';
                document.getElementById('status-text').textContent = 
                    '✗ Backend Offline - ' + err.message;
            }
        }

        // Load all posts
        async function loadPosts() {
            try {
                const res = await fetch(API_URL + '/posts');
                const posts = await res.json();
                
                const postsDiv = document.getElementById('posts');
                
                if (posts.length === 0) {
                    postsDiv.innerHTML = '<p style="color: #999; text-align: center;">No posts yet. Create your first post!</p>';
                    return;
                }
                
                postsDiv.innerHTML = posts.map(post => \`
                    <div class="post">
                        <h3>\${post.title}</h3>
                        <div class="post-meta">
                            By \${post.author} • 
                            \${new Date(post.created_at).toLocaleString()}
                        </div>
                        <div class="post-content">\${post.content}</div>
                        <button class="delete-btn" onclick="deletePost(\${post.id})">
                            Delete
                        </button>
                    </div>
                \`).join('');
            } catch (err) {
                alert('Error loading posts: ' + err.message);
            }
        }

        // Create new post
        async function createPost() {
            const title = document.getElementById('title').value;
            const author = document.getElementById('author').value;
            const content = document.getElementById('content').value;

            if (!title || !author || !content) {
                alert('Please fill in all fields');
                return;
            }

            try {
                const res = await fetch(API_URL + '/posts', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify({ title, author, content })
                });

                if (res.ok) {
                    document.getElementById('title').value = '';
                    document.getElementById('author').value = '';
                    document.getElementById('content').value = '';
                    loadPosts();
                    alert('Post created successfully!');
                }
            } catch (err) {
                alert('Error creating post: ' + err.message);
            }
        }

        // Delete post
        async function deletePost(postId) {
            if (!confirm('Delete this post?')) return;

            try {
                const res = await fetch(API_URL + '/posts/' + postId, {
                    method: 'DELETE'
                });

                if (res.ok) {
                    loadPosts();
                }
            } catch (err) {
                alert('Error deleting post: ' + err.message);
            }
        }

        // Initialize
        window.onload = () => {
            checkHealth();
            loadPosts();
        };
    </script>
</body>
</html>
"@ | Out-File -FilePath "frontend\index.html" -Encoding UTF8

# Create frontend/Dockerfile
@"
FROM nginx:alpine

# Copy our HTML file
COPY index.html /usr/share/nginx/html/index.html

# Expose port 80
EXPOSE 80

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
"@ | Out-File -FilePath "frontend\Dockerfile" -Encoding UTF8

Write-Host "✅ Frontend created" -ForegroundColor Green
```

**Understanding the Frontend:**

```javascript
const API_URL = '/api';
// ↑ Routes to backend through Nginx
// Nginx forwards /api/* to backend:8000

async function createPost() {
    fetch(API_URL + '/posts', {
        method: 'POST',
        body: JSON.stringify({ title, content })
    });
}
// ↑ Sends POST request to create blog post
// Goes to: nginx → backend → creates post
```

---

### Step 5: Create docker-compose.yml

```powershell
# Create docker-compose.yml
@"
version: '3.8'

services:
  # Nginx Reverse Proxy
  nginx:
    image: nginx:alpine
    container_name: blog-nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - frontend
      - backend
    networks:
      - blog-network
    restart: unless-stopped

  # Frontend Service
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: blog-frontend
    networks:
      - blog-network
    restart: unless-stopped

  # Backend API Service
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: blog-backend
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/blog
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
    networks:
      - blog-network
    restart: unless-stopped

  # PostgreSQL Database
  db:
    image: postgres:15-alpine
    container_name: blog-db
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: blog
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - blog-network
    restart: unless-stopped

  # Redis Cache
  redis:
    image: redis:alpine
    container_name: blog-redis
    volumes:
      - redis-data:/data
    networks:
      - blog-network
    restart: unless-stopped

volumes:
  postgres-data:
    name: blog-postgres-data
  redis-data:
    name: blog-redis-data

networks:
  blog-network:
    name: blog-network
    driver: bridge
"@ | Out-File -FilePath "docker-compose.yml" -Encoding UTF8

Write-Host "✅ docker-compose.yml created" -ForegroundColor Green
```

**Understanding docker-compose.yml:**

```yaml
nginx:
  depends_on:
    - frontend
    - backend
  # ↑ Nginx starts AFTER frontend and backend

backend:
  environment:
    - DATABASE_URL=postgresql://user:pass@db:5432/blog
  # ↑ Backend can connect to database using "db" hostname
  depends_on:
    - db
    - redis
  # ↑ Backend starts AFTER db and redis

networks:
  blog-network:
  # ↑ All services on same network = can communicate
```

---

### Step 6: Start the Complete Application

```powershell
Write-Host "`n🚀 Starting Blog Platform..." -ForegroundColor Cyan

# Build and start all services
docker compose up -d --build

Write-Host "`n⏳ Waiting for services to start..." -ForegroundColor Yellow
Start-Sleep -Seconds 15

# Show status
Write-Host "`n📊 Service Status:" -ForegroundColor Green
docker compose ps

# Show logs
Write-Host "`n📜 Recent Logs:" -ForegroundColor Green
docker compose logs --tail=5

# Test backend health
Write-Host "`n🔍 Testing Backend..." -ForegroundColor Yellow
try {
    $response = Invoke-WebRequest -Uri "http://localhost/api/" -UseBasicParsing
    Write-Host "✅ Backend: " $response.StatusCode -ForegroundColor Green
} catch {
    Write-Host "⚠️ Backend not ready yet" -ForegroundColor Yellow
}

# Open browser
Write-Host "`n🌐 Opening Blog Platform..." -ForegroundColor Cyan
Start-Process "http://localhost"

Write-Host "`n✅ Blog Platform is running!" -ForegroundColor Green
Write-Host "`nURL: http://localhost" -ForegroundColor Cyan
Write-Host "API: http://localhost/api" -ForegroundColor Cyan
Write-Host "`nCommands:" -ForegroundColor Yellow
Write-Host "  docker compose logs -f         # View logs"
Write-Host "  docker compose ps              # Check status"
Write-Host "  docker compose down            # Stop everything"
Write-Host "  docker compose restart backend # Restart service"
```

---

## 🎯 How Everything Works Together

### Request Flow

```
1. User opens http://localhost
   ↓
2. Request goes to Nginx (port 80)
   ↓
3. Nginx routes "/" to Frontend
   ↓
4. User sees HTML page
   ↓
5. User creates post, clicks "Publish"
   ↓
6. JavaScript sends POST to /api/posts
   ↓
7. Nginx routes "/api/*" to Backend
   ↓
8. Backend creates post, stores in memory
   (In real app: stores in PostgreSQL)
   ↓
9. Backend returns success
   ↓
10. Frontend refreshes and shows new post
```

### Service Communication

```
Frontend ──────► Backend ──────► Database
   ↑                ↓
   │              Redis
   │
   └────── All through Docker Network ──────┘
```

**Key Points:**
- Services use **container names** as hostnames
- `backend` in docker-compose.yml → accessible as `http://backend:8000`
- Docker handles DNS automatically!

---

## 🔧 Managing the Application

### View Logs

```powershell
# All services
docker compose logs

# Follow logs (real-time)
docker compose logs -f

# Specific service
docker compose logs backend

# Last 20 lines
docker compose logs --tail=20

# With timestamps
docker compose logs -t
```

### Restart Services

```powershell
# Restart all
docker compose restart

# Restart one service
docker compose restart backend

# Restart and rebuild
docker compose up -d --build backend
```

### Scale Services

```powershell
# Run 3 backend instances
docker compose up -d --scale backend=3

# Check
docker compose ps
```

### Execute Commands

```powershell
# Get shell in backend
docker compose exec backend sh

# Run Python in backend
docker compose exec backend python

# Check database
docker compose exec db psql -U user -d blog
```

### Stop and Clean Up

```powershell
# Stop services (keep data)
docker compose stop

# Stop and remove containers
docker compose down

# Remove everything including volumes
docker compose down -v

# Remove everything including images
docker compose down --rmi all -v
```

---

## 📊 Troubleshooting

### Service Won't Start

```powershell
# Check logs
docker compose logs backend

# Check if port is in use
netstat -ano | findstr :80

# Rebuild from scratch
docker compose down -v
docker compose up -d --build
```

### Can't Connect to Backend

```powershell
# Check if backend is running
docker compose ps

# Test backend directly
docker compose exec backend curl http://localhost:8000

# Check nginx config
docker compose exec nginx nginx -t
```

### Database Connection Issues

```powershell
# Check if database is ready
docker compose exec db pg_isready -U user

# Connect to database
docker compose exec db psql -U user -d blog

# Check logs
docker compose logs db
```

---

## ✅ Summary

**What You Built:**
- Complete blog platform
- 5 interconnected services
- Production-ready architecture

**Architecture:**
```
Nginx (Reverse Proxy)
├── Frontend (User Interface)
└── Backend (API)
    ├── Database (PostgreSQL)
    └── Cache (Redis)
```

**Key Learnings:**
1. **docker-compose.yml** orchestrates multiple services
2. **depends_on** controls startup order
3. Services communicate using **container names**
4. **Nginx** acts as reverse proxy
5. **Volumes** persist data
6. **Networks** enable communication

**One Command Deployment:**
```powershell
docker compose up -d
# ✨ Entire platform runs!
```

---

## 🚀 Next Steps

### Enhancements You Can Add:

1. **Database Integration**
   - Replace in-memory storage with PostgreSQL
   - Use SQLAlchemy or similar ORM

2. **Redis Caching**
   - Cache frequently accessed posts
   - Session storage

3. **Authentication**
   - User login/signup
   - JWT tokens

4. **File Uploads**
   - Image uploads for posts
   - Store in volumes

5. **Search Functionality**
   - Search posts by title/content
   - Use Elasticsearch

Want me to show you how to add any of these features? 😊

---

**Try it now!** Run the commands above and you'll have a complete blog platform running in minutes! 🎉