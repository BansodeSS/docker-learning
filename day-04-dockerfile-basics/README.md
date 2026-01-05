# Day 4: Introduction to Dockerfile

## 📖 What You'll Learn Today
- What is a Dockerfile and why we need it
- Dockerfile syntax and instructions
- Creating your first Dockerfile
- Building custom images
- Understanding layers and caching

## 🎯 Understanding Dockerfile

A **Dockerfile** is a text file containing instructions to build a Docker image. Think of it as a recipe that tells Docker:
- What base image to start from
- What files to copy
- What dependencies to install
- What commands to run
- How to start your application

### Why Use Dockerfile?
- **Reproducibility**: Same Dockerfile = Same image every time
- **Version Control**: Track changes to your infrastructure
- **Automation**: Build images automatically in CI/CD
- **Documentation**: Your infrastructure as code

## 📚 Dockerfile Instructions

### Essential Instructions

```dockerfile
# FROM - Base image (always first instruction)
FROM ubuntu:22.04

# RUN - Execute commands during build
RUN apt-get update && apt-get install -y nginx

# COPY - Copy files from host to image
COPY index.html /var/www/html/

# ADD - Like COPY but can extract archives and download URLs
ADD app.tar.gz /app/

# WORKDIR - Set working directory
WORKDIR /app

# ENV - Set environment variables
ENV APP_ENV=production

# EXPOSE - Document which ports the container listens on
EXPOSE 80

# CMD - Default command when container starts
CMD ["nginx", "-g", "daemon off;"]

# ENTRYPOINT - Configure container as executable
ENTRYPOINT ["python", "app.py"]

# USER - Set user for subsequent instructions
USER appuser

# VOLUME - Create mount point
VOLUME /data

# ARG - Build-time variables
ARG VERSION=1.0
```

## 🚀 Project 1: Simple HTML Website

### Create Project Structure
```bash
mkdir docker-html-app
cd docker-html-app
```

### Create index.html
```bash
cat > index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>My Docker App</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 50px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            text-align: center;
        }
        .container {
            background: rgba(255,255,255,0.1);
            padding: 50px;
            border-radius: 15px;
            display: inline-block;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🐳 Built with Docker!</h1>
        <p>This image was built from a custom Dockerfile</p>
        <p>Date: <span id="date"></span></p>
    </div>
    <script>
        document.getElementById('date').textContent = new Date().toLocaleDateString();
    </script>
</body>
</html>
EOF
```

### Create Dockerfile
```bash
cat > Dockerfile << 'EOF'
# Use official nginx image as base
FROM nginx:alpine

# Copy our HTML file to nginx html directory
COPY index.html /usr/share/nginx/html/index.html

# Expose port 80
EXPOSE 80

# Start nginx (CMD is inherited from base image)
EOF
```

### Build and Run
```bash
# Build the image
docker build -t my-html-app .

# Run container from your custom image
docker run -d -p 8080:80 --name html-app my-html-app

# Test: http://localhost:8080

# View image details
docker images my-html-app
```

## 🚀 Project 2: Python Application

### Create Project Structure
```bash
mkdir docker-python-app
cd docker-python-app
```

### Create app.py
```bash
cat > app.py << 'EOF'
from flask import Flask, jsonify
import socket
import datetime

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify({
        'message': 'Hello from Dockerized Flask!',
        'container': socket.gethostname(),
        'timestamp': datetime.datetime.now().isoformat()
    })

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
EOF
```

### Create requirements.txt
```bash
cat > requirements.txt << 'EOF'
flask==3.0.0
EOF
```

### Create Dockerfile
```bash
cat > Dockerfile << 'EOF'
# Use official Python runtime as base image
FROM python:3.11-slim

# Set working directory in container
WORKDIR /app

# Copy requirements file
COPY requirements.txt .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY app.py .

# Expose port
EXPOSE 5000

# Set environment variable
ENV FLASK_APP=app.py

# Run the application
CMD ["python", "app.py"]
EOF
```

### Build and Run
```bash
# Build image
docker build -t my-python-app .

# Run container
docker run -d -p 5000:5000 --name python-app my-python-app

# Test endpoints
curl http://localhost:5000/
curl http://localhost:5000/health

# View logs
docker logs python-app
```

## 🚀 Project 3: Node.js Application

### Create Project Structure
```bash
mkdir docker-node-app
cd docker-node-app
```

### Create app.js
```bash
cat > app.js << 'EOF'
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Node.js app running in Docker!',
    version: process.version,
    platform: process.platform,
    uptime: process.uptime()
  });
});

app.listen(port, '0.0.0.0', () => {
  console.log(`App listening on port ${port}`);
});
EOF
```

### Create package.json
```bash
cat > package.json << 'EOF'
{
  "name": "docker-node-app",
  "version": "1.0.0",
  "main": "app.js",
  "dependencies": {
    "express": "^4.18.2"
  },
  "scripts": {
    "start": "node app.js"
  }
}
EOF
```

### Create Dockerfile
```bash
cat > Dockerfile << 'EOF'
# Use Node.js official image
FROM node:18-alpine

# Create app directory
WORKDIR /usr/src/app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy app source
COPY app.js .

# Expose port
EXPOSE 3000

# Start app
CMD ["npm", "start"]
EOF
```

### Build and Run
```bash
# Build
docker build -t my-node-app .

# Run
docker run -d -p 3000:3000 --name node-app my-node-app

# Test
curl http://localhost:3000/
```

## 📊 Understanding Docker Build

### Build Process
```
1. Read Dockerfile
2. Execute each instruction
3. Create a new layer for each instruction
4. Save final image with all layers
```

### Layering Example
```dockerfile
FROM python:3.11         # Layer 1: Base image
WORKDIR /app             # Layer 2: Create directory
COPY requirements.txt .  # Layer 3: Copy file
RUN pip install -r ...   # Layer 4: Install packages
COPY app.py .           # Layer 5: Copy app
CMD ["python", "app.py"] # Layer 6: Set command
```

### Build Cache
Docker caches each layer. If a layer hasn't changed, Docker reuses it:

```bash
# First build - builds all layers
docker build -t myapp .

# Change only app.py
# Second build - reuses cached layers up to COPY app.py
docker build -t myapp .
```

## 💻 Hands-on Exercises

### Exercise 1: Multi-Stage Build Awareness
```dockerfile
# We'll learn multi-stage builds later, but here's a preview
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Exercise 2: With Build Arguments
```dockerfile
FROM python:3.11-slim

ARG APP_VERSION=1.0
ENV VERSION=$APP_VERSION

WORKDIR /app
COPY . .

RUN echo "Building version ${VERSION}"
CMD ["python", "app.py"]
```

Build with arguments:
```bash
docker build --build-arg APP_VERSION=2.0 -t myapp:2.0 .
```

## 🎓 What I Learned Today

**Date**: ___________

**Projects Built**:
- [ ] HTML static website
- [ ] Python Flask application
- [ ] Node.js Express application

**Dockerfiles Created**:
```
[Paste your Dockerfiles here]
```

**Build Commands**:
```
[Commands you used to build images]
```

**Understanding Check**:
1. What's the difference between COPY and ADD?
   
2. What does WORKDIR do?
   
3. Why is layer caching important?
   

**Challenges**:
```
[Any build errors or issues]
```

**Solutions**:
```
[How you fixed them]
```

## 📝 Best Practices Learned

1. **Order matters**: Put frequently changing instructions last
2. **Combine RUN commands**: Reduces layers
   ```dockerfile
   # Bad
   RUN apt-get update
   RUN apt-get install -y nginx
   
   # Good
   RUN apt-get update && apt-get install -y nginx
   ```

3. **Use specific tags**: Don't rely on `latest`
   ```dockerfile
   # Vague
   FROM python:latest
   
   # Better
   FROM python:3.11-slim
   ```

4. **Clean up in same layer**:
   ```dockerfile
   RUN apt-get update && \
       apt-get install -y nginx && \
       apt-get clean && \
       rm -rf /var/lib/apt/lists/*
   ```

5. **Use .dockerignore**: Like .gitignore for Docker
   ```bash
   cat > .dockerignore << 'EOF'
   __pycache__
   *.pyc
   .git
   .env
   node_modules
   EOF
   ```

## 🧪 Challenge Projects

1. Create a Dockerfile for a Go application
2. Build a multi-language app (Python + Node.js)
3. Create a Dockerfile that accepts environment variables
4. Build an image with health checks

## 📋 Command Reference

```bash
# Build image
docker build -t name:tag .

# Build with no cache
docker build --no-cache -t name:tag .

# Build with build arguments
docker build --build-arg VAR=value -t name:tag .

# View build history
docker history image-name

# Remove dangling images
docker image prune
```

---

**Previous**: [Day 3 - First Container](../day-03-first-container/README.md)  
**Next**: [Day 5 - Building Images](../day-05-building-images/README.md)