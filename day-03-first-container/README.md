# Day 3: Running Your First Real Container

## 📖 What You'll Learn Today
- Deploy a complete web application using containers
- Understand port mapping in detail
- Work with environment variables
- Interact with containerized applications
- Container naming and management

## 🎯 Today's Project: Deploy a Real Web Application

We'll deploy multiple real-world applications to understand how containers work in practice.

## 🚀 Project 1: Static Website with Nginx

### Step 1: Create Custom HTML
```bash
# Create a directory for your website
mkdir my-website
cd my-website

# Create an HTML file
cat > index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>My First Docker Website</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 50px auto;
            padding: 20px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
        }
        .container {
            background: rgba(255,255,255,0.1);
            padding: 30px;
            border-radius: 10px;
        }
        h1 { color: #fff; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🐳 Hello from Docker!</h1>
        <p>This website is running inside a Docker container.</p>
        <p>Current time: <span id="time"></span></p>
    </div>
    <script>
        document.getElementById('time').textContent = new Date().toLocaleString();
    </script>
</body>
</html>
EOF
```

### Step 2: Run Nginx with Your HTML
```bash
# Run nginx and mount your HTML file
docker run -d \
  --name my-website \
  -p 8080:80 \
  -v $(pwd)/index.html:/usr/share/nginx/html/index.html \
  nginx

# Test it
# Open browser: http://localhost:8080
```

### Step 3: Check Logs and Status
```bash
# View logs
docker logs my-website

# Check if running
docker ps

# View resource usage
docker stats my-website --no-stream
```

## 🚀 Project 2: Redis Database

### Deploy Redis
```bash
# Run Redis container
docker run -d \
  --name my-redis \
  -p 6379:6379 \
  redis

# Check logs
docker logs my-redis

# Connect to Redis CLI
docker exec -it my-redis redis-cli

# Inside Redis CLI, try:
SET mykey "Hello from Docker"
GET mykey
PING
INFO
exit
```

## 🚀 Project 3: PostgreSQL Database

### Deploy PostgreSQL
```bash
# Run PostgreSQL with environment variables
docker run -d \
  --name my-postgres \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -e POSTGRES_USER=admin \
  -e POSTGRES_DB=myapp \
  -p 5432:5432 \
  postgres:15

# Check logs (wait for "database system is ready to accept connections")
docker logs -f my-postgres

# Connect to PostgreSQL
docker exec -it my-postgres psql -U admin -d myapp

# Inside PostgreSQL, try:
\l                          -- list databases
\dt                         -- list tables
CREATE TABLE users (id SERIAL PRIMARY KEY, name VARCHAR(100));
\dt
INSERT INTO users (name) VALUES ('Docker User');
SELECT * FROM users;
\q
```

## 🚀 Project 4: Python Web App

### Create a Simple Flask App
```bash
# Create app directory
mkdir flask-app
cd flask-app

# Create app.py
cat > app.py << 'EOF'
from flask import Flask
import socket

app = Flask(__name__)

@app.route('/')
def hello():
    hostname = socket.gethostname()
    return f'''
    <html>
        <body style="font-family: Arial; padding: 50px; background: #f0f0f0;">
            <h1>🐍 Flask App in Docker</h1>
            <p><strong>Container Hostname:</strong> {hostname}</p>
            <p>This Python app is running inside a Docker container!</p>
        </body>
    </html>
    '''

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
```

### Run Python Container
```bash
# Run Python with Flask (mounting your code)
docker run -d \
  --name flask-app \
  -p 5000:5000 \
  -v $(pwd)/app.py:/app/app.py \
  -w /app \
  python:3.11-slim \
  sh -c "pip install flask && python app.py"

# Check logs
docker logs -f flask-app

# Test: http://localhost:5000
```

## 📊 Understanding What We Did

### Port Mapping Explained
```
-p 8080:80
   ↑    ↑
   |    |
   |    └─ Container port (inside)
   └────── Host port (your computer)
```

When you access `localhost:8080`, Docker forwards the request to port `80` inside the container.

### Volume Mounting Explained
```
-v $(pwd)/index.html:/usr/share/nginx/html/index.html
   ↑                  ↑
   |                  |
   |                  └─ Path inside container
   └──────────────────── Path on your computer
```

Changes to your local file are immediately reflected in the container.

### Environment Variables
```bash
-e POSTGRES_PASSWORD=mysecretpassword
```

Passes configuration to the container without modifying the image.

## 💻 Hands-on Exercises

### Exercise 1: Multi-Port Application
```bash
# Run nginx on multiple ports
docker run -d --name web-8081 -p 8081:80 nginx
docker run -d --name web-8082 -p 8082:80 nginx
docker run -d --name web-8083 -p 8083:80 nginx

# Test all three
# http://localhost:8081
# http://localhost:8082
# http://localhost:8083

# Clean up
docker stop web-8081 web-8082 web-8083
docker rm web-8081 web-8082 web-8083
```

### Exercise 2: Container Communication
```bash
# Create a custom network
docker network create myapp-network

# Run containers on same network
docker run -d --name db --network myapp-network postgres:15 -e POSTGRES_PASSWORD=pass
docker run -d --name web --network myapp-network nginx

# They can now talk to each other using container names as hostnames
docker exec web ping db
```

## 🎓 What I Learned Today

**Date**: ___________

**Projects Completed**:
- [ ] Static website with Nginx
- [ ] Redis database
- [ ] PostgreSQL database
- [ ] Python Flask application

**Screenshots**:
*Add screenshots of each running application*

**Commands Used**:
```
[Paste the commands you ran successfully]
```

**Understanding Check**:
1. What does `-p 8080:80` do?
   
2. What is the difference between container name and container ID?
   
3. How do you view logs of a running container?
   

**Challenges Faced**:
```
[Document any issues]
```

**Solutions**:
```
[How you resolved them]
```

## 🧪 Challenge Projects

Try these on your own:

1. **MongoDB**: Run a MongoDB container and connect to it
2. **Apache**: Run Apache web server on port 8090
3. **Node.js**: Run a Node.js application container
4. **Multiple Services**: Run a web server + database simultaneously

## 📝 Important Concepts

### Container Lifecycle
```
docker run → Container CREATED → Container RUNNING
                                        ↓
docker stop → Container STOPPED
                    ↓
docker start → Container RUNNING
                    ↓
docker rm → Container DELETED
```

### Best Practices Learned
1. Always name your containers (`--name`)
2. Use specific image tags, not just `latest`
3. Check logs when things don't work
4. Clean up unused containers regularly
5. Use environment variables for configuration

### Common Issues and Solutions

**Port already in use**:
```bash
# Error: port is already allocated
# Solution: Use different port or stop conflicting service
docker ps  # Find conflicting container
docker stop <container>
```

**Container exits immediately**:
```bash
# Check logs to see why
docker logs <container>
```

**Can't access container**:
```bash
# Verify it's running
docker ps

# Check port mapping
docker port <container>
```

## 🔗 Resources
- [Docker Hub](https://hub.docker.com) - Find more images
- [Nginx Docker Official Image](https://hub.docker.com/_/nginx)
- [PostgreSQL Docker Official Image](https://hub.docker.com/_/postgres)
- [Redis Docker Official Image](https://hub.docker.com/_/redis)

---

**Previous**: [Day 2 - Basic Commands](../day-02-basic-commands/README.md)  
**Next**: [Day 4 - Dockerfile Basics](../day-04-dockerfile-basics/README.md)