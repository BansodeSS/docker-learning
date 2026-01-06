# Day 5: Building Optimized Docker Images - Deep Dive

## 🎯 Why Image Optimization Matters

### The Problem with Unoptimized Images

**Example: A Simple Node.js App**

**❌ Bad Dockerfile (Unoptimized):**
```dockerfile
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
CMD ["npm", "start"]
```

**Result:**
- Image size: **~1.1 GB** 😱
- Contains: Node.js, npm, build tools, source code, node_modules, tests, documentation
- Security risk: More software = more vulnerabilities
- Slow to download and deploy

**✅ Good Dockerfile (Optimized):**
```dockerfile
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]
```

**Result:**
- Image size: **~150 MB** 🎉
- Contains: Only runtime files, production dependencies
- Faster deployment
- More secure

---

## 🏗️ Multi-Stage Builds Explained

### What is a Multi-Stage Build?

Think of it like **building a house**:

1. **Stage 1 (Construction Site)**: You bring ALL the tools, materials, workers, scaffolding
2. **Stage 2 (Final House)**: You only keep the finished house, throw away the tools and scaffolding

### Visual Breakdown

```
┌─────────────────────────────────────────┐
│  STAGE 1: Build Stage (Temporary)      │
│  ┌─────────────────────────────────┐   │
│  │ • Node.js + npm                 │   │
│  │ • Build tools (webpack, babel)  │   │
│  │ • Dev dependencies              │   │
│  │ • Source code (.jsx, .tsx)      │   │
│  │ • Tests                         │   │
│  │                                 │   │
│  │  BUILD → Creates /app/dist/     │   │
│  └─────────────────────────────────┘   │
│            ↓                            │
│      (Copy only dist/)                  │
│            ↓                            │
│  ┌─────────────────────────────────┐   │
│  │  STAGE 2: Production (Final)    │   │
│  │  • Minimal Node.js runtime      │   │
│  │  • Only /dist/ folder           │   │
│  │  • Production dependencies only │   │
│  │  Size: 10x smaller! ✨          │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

---

## 📝 Line-by-Line Explanation: Node.js Multi-Stage Build

```dockerfile
# STAGE 1: BUILD
# ============================================================
FROM node:18 AS build
# ↑ Start with full Node.js 18 image
# ↑ Name this stage "build" so we can reference it later
# Size: ~900MB (includes everything needed to build)

WORKDIR /app
# Create and enter /app directory

COPY package*.json ./
# Copy ONLY package.json and package-lock.json first
# Why? Docker caches layers - if these don't change, 
# the next RUN command won't re-execute

RUN npm install
# Install ALL dependencies (dev + production)
# Includes webpack, babel, typescript, testing tools, etc.

COPY . .
# Copy all source code

RUN npm run build
# Build the application
# This creates optimized production files in /app/dist/
# Files are minified, bundled, optimized


# STAGE 2: PRODUCTION
# ============================================================
FROM nginx:alpine
# ↑ Start FRESH with a new, minimal image
# ↑ Only ~40MB! Previous stage is DISCARDED
# This is the magic - we throw away all build tools!

COPY --from=build /app/dist /usr/share/nginx/html
#      ↑            ↑         ↑
#      |            |         └─ Destination in new image
#      |            └─────────── Source from build stage
#      └──────────────────────── Get files from "build" stage

EXPOSE 80
# Document that nginx listens on port 80

CMD ["nginx", "-g", "daemon off;"]
# Start nginx server
```

**What happens to Stage 1?**
- ❌ **Completely discarded!** 
- Only the files you `COPY --from=build` are kept
- All build tools, source code, dev dependencies = GONE

---

## 🐍 Python Multi-Stage Build Explained

```dockerfile
# STAGE 1: BUILDER
# ============================================================
FROM python:3.11 AS builder
# Full Python image with pip, build tools
# Size: ~900MB

WORKDIR /app

COPY requirements.txt .
# Copy dependency list

RUN pip install --user -r requirements.txt
#               ↑
#               └─ --user installs to /root/.local/
#                  (user's home directory, not system-wide)
# This makes it easy to copy in next stage

# Note: We DON'T copy source code here yet
# Build dependencies are in /root/.local/ now


# STAGE 2: RUNTIME
# ============================================================
FROM python:3.11-slim
# ↑ Smaller Python image WITHOUT build tools
# Size: ~120MB (vs 900MB)
# No gcc, no development headers, just Python runtime

WORKDIR /app

COPY --from=builder /root/.local /root/.local
# ↑ Copy ONLY the installed packages from builder
# Not copying pip, setuptools, or build tools

COPY . .
# Copy application code

ENV PATH=/root/.local/bin:$PATH
# ↑ Make sure Python can find the packages we copied

CMD ["python", "app.py"]
```

### Why This Works

**Stage 1 (Builder):**
- Has `gcc`, `make`, build tools
- Can compile C extensions for Python packages
- Installs everything to `/root/.local/`

**Stage 2 (Runtime):**
- Doesn't need build tools (already compiled!)
- Just needs Python interpreter
- Copies pre-compiled packages from Stage 1

**Result:** Image is **7-8x smaller!**

---

## 📦 Size Comparison Examples

### Example 1: Node.js Application

```dockerfile
# ❌ Single Stage (Bad)
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
CMD ["npm", "start"]
# Final size: 1.1 GB
```

```dockerfile
# ✅ Multi-Stage (Good)
FROM node:18 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]
# Final size: 150 MB (7x smaller!)
```

### Example 2: Go Application (Most Dramatic)

```dockerfile
# ❌ Single Stage
FROM golang:1.21
WORKDIR /app
COPY . .
RUN go build -o main .
CMD ["./main"]
# Final size: 800 MB
```

```dockerfile
# ✅ Multi-Stage with Scratch
FROM golang:1.21 AS build
WORKDIR /app
COPY . .
RUN go build -o main .

FROM scratch
# ↑ Completely empty image (0 MB base!)
COPY --from=build /app/main /main
CMD ["/main"]
# Final size: 15 MB (50x smaller!)
```

---

## 🔍 Alpine Images Explained

### What is Alpine?

Alpine Linux is a **minimal Linux distribution**:
- Size: ~5 MB base image
- vs Ubuntu: ~77 MB base image

### Comparison

```dockerfile
# Standard Python (Debian-based)
FROM python:3.11
# Size: ~900 MB
# Includes: Full Debian OS, many utilities

# Alpine Python
FROM python:3.11-alpine
# Size: ~50 MB
# Includes: Minimal Alpine OS, Python only

# Slim Python (Debian-slim)
FROM python:3.11-slim
# Size: ~120 MB
# Includes: Minimal Debian, Python
```

### When to Use Alpine

**✅ Good for:**
- Simple applications
- Production deployments
- Size-critical scenarios

**❌ Not ideal for:**
- Apps with C dependencies (compilation issues)
- Complex system requirements
- Development (missing debugging tools)

**💡 Best choice:** `python:3.11-slim` - Good balance!

---

## 🎯 Practical Example: Before & After

### Create a Real Node.js App

**Step 1: Create project files**

```powershell
# Create directory
mkdir docker-optimization-demo
cd docker-optimization-demo

# Create package.json
@"
{
  "name": "optimization-demo",
  "version": "1.0.0",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
"@ | Out-File -FilePath "package.json" -Encoding UTF8

# Create server.js
@"
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.send('<h1>Optimized Docker App!</h1>');
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
"@ | Out-File -FilePath "server.js" -Encoding UTF8
```

**Step 2: Create UNOPTIMIZED Dockerfile**

```dockerfile
# Dockerfile.unoptimized
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 3000
CMD ["npm", "start"]
```

**Step 3: Build and check size**

```powershell
# Build unoptimized
docker build -f Dockerfile.unoptimized -t myapp:unoptimized .

# Check size
docker images myapp:unoptimized
# SIZE: ~1.1 GB 😱
```

**Step 4: Create OPTIMIZED Dockerfile**

```dockerfile
# Dockerfile.optimized
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

FROM node:18-alpine
WORKDIR /app
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/server.js ./server.js
EXPOSE 3000
CMD ["node", "server.js"]
```

**Step 5: Build optimized version**

```powershell
# Build optimized
docker build -f Dockerfile.optimized -t myapp:optimized .

# Check size
docker images myapp:optimized
# SIZE: ~150 MB 🎉

# Compare
docker images | Select-String myapp
```

**Step 6: Test both**

```powershell
# Test unoptimized
docker run -d -p 3001:3000 --name test-unoptimized myapp:unoptimized

# Test optimized
docker run -d -p 3002:3000 --name test-optimized myapp:optimized

# Both work the same!
# http://localhost:3001
# http://localhost:3002
```

**Result:**
- **Same functionality**
- **7x smaller size**
- **Faster deployment**
- **More secure**

---

## 🛡️ .dockerignore Explained

### What is .dockerignore?

Like `.gitignore` but for Docker - tells Docker **what NOT to copy** into the image.

### Why It Matters

```dockerfile
COPY . .
# ↑ Without .dockerignore, copies EVERYTHING:
# - node_modules (hundreds of MB)
# - .git history (can be huge)
# - Test files
# - Documentation
# - Your personal files
```

### Create .dockerignore

```powershell
# Create .dockerignore file
@"
# Dependencies
node_modules
npm-debug.log

# Git
.git
.gitignore

# Documentation
*.md
docs/

# Tests
test/
*.test.js
coverage/

# Environment
.env
.env.local

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db

# Build artifacts
dist/
build/
*.log

# Python specific
__pycache__/
*.pyc
*.pyo
venv/
"@ | Out-File -FilePath ".dockerignore" -Encoding UTF8
```

### Impact Example

**Without .dockerignore:**
```
COPY . .
# Copies: 500 MB (includes node_modules, .git, etc.)
```

**With .dockerignore:**
```
COPY . .
# Copies: 2 MB (only source code)
```

---

## 📊 Command Combination Explained

### ❌ Bad: Multiple RUN Commands

```dockerfile
FROM python:3.11-alpine

RUN apk add gcc        # Layer 1: ~200 MB
RUN apk add musl-dev   # Layer 2: ~50 MB
RUN pip install flask  # Layer 3: ~100 MB
RUN apk del gcc        # Layer 4: Still ~350 MB total!
```

**Problem:** Each RUN creates a new layer. `apk del gcc` removes files but **the layer still exists!**

### ✅ Good: Combined RUN Command

```dockerfile
FROM python:3.11-alpine

RUN apk add --no-cache gcc musl-dev && \
    pip install --no-cache-dir flask && \
    apk del gcc musl-dev
# Single layer: ~100 MB
```

**Benefit:** Build tools are installed, used, and removed in **one layer**.

### Why This Works

```
Layer anatomy:
┌─────────────────────────┐
│ Layer 1: Base image     │
├─────────────────────────┤
│ Layer 2: RUN command    │
│ - Install gcc           │
│ - Install flask         │
│ - Remove gcc            │
│ Net result: Only flask  │
└─────────────────────────┘
```

---

## 🎓 Complete Optimization Checklist

### ✅ Image Size Optimization

- [ ] Use Alpine or Slim base images
- [ ] Use multi-stage builds
- [ ] Combine RUN commands
- [ ] Use .dockerignore
- [ ] Don't install unnecessary packages
- [ ] Clean package manager cache

```dockerfile
# All optimizations in one Dockerfile
FROM node:18-alpine AS build
WORKDIR /app

# Optimization 1: Copy only package files first
COPY package*.json ./

# Optimization 2: Use --only=production
RUN npm ci --only=production

COPY . .
RUN npm run build

# Optimization 3: Multi-stage - start fresh
FROM node:18-alpine

WORKDIR /app

# Optimization 4: Copy only what's needed
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules

# Optimization 5: Use non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001
USER nodejs

EXPOSE 3000
CMD ["node", "dist/server.js"]
```

---

## 🧪 Hands-On Exercise

### Exercise: Optimize a Python Flask App

**Step 1: Create unoptimized version**

```dockerfile
# Dockerfile.before
FROM python:3.11
WORKDIR /app
COPY . .
RUN pip install flask
CMD ["python", "app.py"]
```

**Step 2: Build and measure**
```powershell
docker build -f Dockerfile.before -t flask:before .
docker images flask:before
# Note the size
```

**Step 3: Create optimized version**

```dockerfile
# Dockerfile.after
FROM python:3.11-alpine AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.11-alpine
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY app.py .
ENV PATH=/root/.local/bin:$PATH
USER nobody
CMD ["python", "app.py"]
```

**Step 4: Build and compare**
```powershell
docker build -f Dockerfile.after -t flask:after .
docker images | Select-String flask

# Compare sizes!
```

---

## 💡 Key Takeaways

1. **Multi-stage builds** = Separate build environment from runtime
2. **Alpine images** = Smaller base images (~5MB vs ~77MB)
3. **Layer awareness** = Combine commands, clean up in same layer
4. **.dockerignore** = Don't copy unnecessary files
5. **Security** = Smaller images = fewer vulnerabilities

**The goal:** Ship only what you need to run your app, nothing more!

---

## 🎯 Summary: Size Reduction Strategies

| Strategy | Size Reduction | Difficulty |
|----------|---------------|------------|
| Use Alpine | 10-15x smaller | Easy ⭐ |
| Multi-stage | 5-10x smaller | Medium ⭐⭐ |
| .dockerignore | 2-5x smaller | Easy ⭐ |
| Combine RUN | 1.5-3x smaller | Easy ⭐ |
| All combined | 15-20x smaller | Medium ⭐⭐ |

Start with Alpine and .dockerignore, then add multi-stage builds once comfortable!

---

Ready to try optimizing your own Dockerfile? Want me to help you optimize a specific application? 🚀