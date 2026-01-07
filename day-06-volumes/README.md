# Day 6: Docker Volumes - Complete Deep Dive

## 🎯 Why Do We Need Volumes?

### The Problem Without Volumes

**Imagine this scenario:**

```powershell
# Start a database container
docker run -d --name mydb postgres:15

# Add data to the database
docker exec -it mydb psql -U postgres -c "CREATE TABLE users (name TEXT);"
docker exec -it mydb psql -U postgres -c "INSERT INTO users VALUES ('Alice');"

# Everything works!

# Now remove the container
docker rm -f mydb

# Start a new container
docker run -d --name mydb postgres:15

# Check for data
docker exec -it mydb psql -U postgres -c "SELECT * FROM users;"
# ❌ ERROR: Table doesn't exist!
# 😱 ALL DATA IS GONE!
```

**Why?** 
- Containers are **ephemeral** (temporary)
- When you delete a container, everything inside is deleted
- Data stored in the container = LOST forever

### The Solution: Volumes

Volumes store data **outside** the container:

```
WITHOUT VOLUME:                    WITH VOLUME:
┌────────────────┐                ┌────────────────┐
│   Container    │                │   Container    │
│  ┌──────────┐  │                │       ↓        │
│  │   Data   │  │ ← DELETED!     │       ↓        │
│  └──────────┘  │                └───────┼────────┘
└────────────────┘                        ↓
                                   ┌──────────────┐
                                   │    Volume    │
                                   │  (Survives!) │
                                   └──────────────┘
```

---

## 📚 Three Types of Volumes

### 1️⃣ Named Volumes (Most Common)

**What:** Docker manages the storage location  
**When:** Production databases, persistent app data  
**Pros:** Easy to manage, Docker handles everything  

```powershell
# Create a named volume
docker volume create mydata

# Use it
docker run -v mydata:/data nginx
                ↑       ↑
                |       └─ Path inside container
                └───────── Volume name
```

**Real example:**
```powershell
# Create volume for PostgreSQL
docker volume create postgres-data

# Run database with volume
docker run -d `
  --name mydb `
  -e POSTGRES_PASSWORD=secret `
  -v postgres-data:/var/lib/postgresql/data `
  postgres:15

# Add some data
docker exec -it mydb psql -U postgres -c "CREATE TABLE test (id INT);"

# Delete container
docker rm -f mydb

# Start NEW container with SAME volume
docker run -d `
  --name mydb-new `
  -e POSTGRES_PASSWORD=secret `
  -v postgres-data:/var/lib/postgresql/data `
  postgres:15

# Data is still there!
docker exec -it mydb-new psql -U postgres -c "\dt"
# ✅ Shows: test table exists!
```

**Where is the data stored?**
```powershell
# Find volume location
docker volume inspect postgres-data

# On Windows with WSL2, it's in:
# \\wsl$\docker-desktop-data\data\docker\volumes\postgres-data\_data
```

---

### 2️⃣ Bind Mounts (Development)

**What:** Map a folder from YOUR computer into the container  
**When:** Development, live code editing  
**Pros:** Direct access to files, instant updates  

```powershell
# Mount your local folder
docker run -v C:/my-project:/app nginx
              ↑              ↑
              |              └─ Path in container
              └────────────────Your computer path
```

**Real example - Live Code Editing:**

```powershell
# Create project folder
mkdir C:\my-website
cd C:\my-website

# Create index.html
@"
<h1>Hello Docker!</h1>
<p>Version 1</p>
"@ | Out-File index.html -Encoding UTF8

# Run nginx with bind mount
docker run -d `
  --name web `
  -p 8080:80 `
  -v ${PWD}:/usr/share/nginx/html `
  nginx:alpine

# Open browser
Start-Process "http://localhost:8080"

# Now edit index.html on your computer
@"
<h1>Hello Docker!</h1>
<p>Version 2 - UPDATED!</p>
"@ | Out-File index.html -Encoding UTF8

# Refresh browser - changes appear INSTANTLY! ✨
# No need to rebuild or restart container!
```

**Visual:**
```
Your Computer              Container
┌─────────────┐           ┌─────────────┐
│ C:\project\ │◄─────────►│   /app/     │
│  index.html │  SHARED   │  index.html │
│  style.css  │           │  style.css  │
└─────────────┘           └─────────────┘
   Edit here                Shows here
   instantly!               immediately!
```

---

### 3️⃣ Anonymous Volumes (Rare)

**What:** Docker creates a volume with random name  
**When:** Temporary data, you don't care about the name  
**Pros:** Quick, no naming needed  

```powershell
docker run -v /data nginx
              ↑
              └─ No name = anonymous
```

**Example:**
```powershell
# Run with anonymous volume
docker run -d --name test -v /app/data nginx

# Docker creates random volume
docker volume ls
# DRIVER    VOLUME NAME
# local     a7f3d9e8b2c1...  ← Random name
```

---

## 🎯 Volume Syntax Explained

### Understanding -v Flag

```powershell
docker run -v SOURCE:DESTINATION:OPTIONS
              ↑       ↑           ↑
              |       |           └─ Optional (ro = read-only)
              |       └─────────────Path inside container
              └─────────────────────Volume name OR local path
```

### Examples:

```powershell
# Named volume
docker run -v mydata:/app/data nginx

# Bind mount (Windows)
docker run -v C:/Users/suhas/project:/app nginx

# Bind mount (PowerShell current directory)
docker run -v ${PWD}:/app nginx

# Read-only volume
docker run -v mydata:/app/data:ro nginx

# Anonymous volume
docker run -v /app/data nginx
```

---

## 🗄️ PostgreSQL Persistence - Step by Step

### Complete Example with Explanation

```powershell
# Step 1: Create a named volume for database
docker volume create postgres-data

# What this does:
# Creates storage space managed by Docker
# Location: Docker's internal storage
# Persists even when containers are deleted
```

```powershell
# Step 2: Run PostgreSQL with the volume
docker run -d `
  --name mydb `
  -e POSTGRES_PASSWORD=secret `
  -e POSTGRES_USER=admin `
  -e POSTGRES_DB=myapp `
  -v postgres-data:/var/lib/postgresql/data `
  postgres:15

# Breaking down the command:
# -d                           Run in background
# --name mydb                  Container name
# -e POSTGRES_PASSWORD=secret  Set password
# -v postgres-data:/var/lib... Mount volume
#    ↑                  ↑
#    Volume name        Where PostgreSQL stores data
```

```powershell
# Step 3: Add some data
docker exec -it mydb psql -U admin -d myapp -c "
  CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
  );
"

docker exec -it mydb psql -U admin -d myapp -c "
  INSERT INTO users (name, email) VALUES 
  ('Alice', 'alice@example.com'),
  ('Bob', 'bob@example.com');
"

# Verify data
docker exec -it mydb psql -U admin -d myapp -c "SELECT * FROM users;"
#  id | name  |       email
# ----+-------+------------------
#   1 | Alice | alice@example.com
#   2 | Bob   | bob@example.com
```

```powershell
# Step 4: DESTROY the container!
docker stop mydb
docker rm mydb

# Container is GONE!
docker ps -a
# (empty - no mydb container)
```

```powershell
# Step 5: Create NEW container with SAME volume
docker run -d `
  --name mydb2 `
  -e POSTGRES_PASSWORD=secret `
  -e POSTGRES_USER=admin `
  -e POSTGRES_DB=myapp `
  -v postgres-data:/var/lib/postgresql/data `
  postgres:15

# Wait for startup
Start-Sleep -Seconds 5
```

```powershell
# Step 6: Check if data survived!
docker exec -it mydb2 psql -U admin -d myapp -c "SELECT * FROM users;"

# ✅ MAGIC! Data is still there!
#  id | name  |       email
# ----+-------+------------------
#   1 | Alice | alice@example.com
#   2 | Bob   | bob@example.com
```

**What happened?**
```
Container 1 (mydb)           Container 2 (mydb2)
┌──────────────┐            ┌──────────────┐
│   Deleted    │            │   New        │
└──────┬───────┘            └──────┬───────┘
       │                           │
       └──────► postgres-data ◄────┘
                ┌──────────────┐
                │  Data Lives  │
                │    Here!     │
                └──────────────┘
```

---

## 💻 Development with Live Reload

### Node.js Development Example

```powershell
# Step 1: Create project structure
mkdir C:\dev-project
cd C:\dev-project

# Create package.json
@"
{
  "name": "dev-app",
  "version": "1.0.0",
  "scripts": {
    "start": "node server.js"
  }
}
"@ | Out-File package.json -Encoding UTF8

# Create server.js
@"
const http = require('http');
const fs = require('fs');

const server = http.createServer((req, res) => {
  const html = fs.readFileSync('index.html', 'utf8');
  res.writeHead(200, { 'Content-Type': 'text/html' });
  res.end(html);
});

server.listen(3000, () => {
  console.log('Server running on http://localhost:3000');
});
"@ | Out-File server.js -Encoding UTF8

# Create index.html
@"
<!DOCTYPE html>
<html>
<head>
    <title>Dev Mode</title>
    <style>
        body {
            font-family: Arial;
            max-width: 800px;
            margin: 50px auto;
            padding: 20px;
            background: #f0f0f0;
        }
    </style>
</head>
<body>
    <h1>🚀 Development Mode</h1>
    <p>Edit this file and refresh to see changes!</p>
    <p>Version: 1.0</p>
</body>
</html>
"@ | Out-File index.html -Encoding UTF8
```

```powershell
# Step 2: Run with bind mount
docker run -d `
  --name dev-app `
  -p 3000:3000 `
  -v ${PWD}:/app `
  -w /app `
  node:18-alpine `
  npm start

# -v ${PWD}:/app     Maps current directory to /app
# -w /app            Sets working directory
# npm start          Runs the server
```

```powershell
# Step 3: Open in browser
Start-Process "http://localhost:3000"

# You see: Version 1.0
```

```powershell
# Step 4: Edit index.html WITHOUT stopping container
@"
<!DOCTYPE html>
<html>
<head>
    <title>Dev Mode</title>
    <style>
        body {
            font-family: Arial;
            max-width: 800px;
            margin: 50px auto;
            padding: 20px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
        }
    </style>
</head>
<body>
    <h1>🚀 Development Mode</h1>
    <p>Edit this file and refresh to see changes!</p>
    <p>Version: 2.0 - UPDATED WITH VOLUMES!</p>
</body>
</html>
"@ | Out-File index.html -Encoding UTF8

# Step 5: Just refresh browser!
# ✨ Changes appear instantly!
```

**Why this works:**
```
Your Computer (Windows)       Container (Linux)
┌────────────────────┐       ┌────────────────────┐
│ C:\dev-project\    │◄─────►│ /app/              │
│   index.html       │       │   index.html       │
│   (YOU EDIT HERE)  │       │   (SERVER READS)   │
└────────────────────┘       └────────────────────┘
      Edit file                   File updates
      Save (Ctrl+S)               Server serves new version
                                  Refresh browser → See changes!
```

---

## 🔧 Volume Management Commands

### Listing Volumes

```powershell
# List all volumes
docker volume ls

# Example output:
# DRIVER    VOLUME NAME
# local     postgres-data
# local     mysql-data
# local     a7f3d9e8b2c1...  ← Anonymous volume
```

### Inspecting Volumes

```powershell
# Get detailed information
docker volume inspect postgres-data

# Output (JSON):
# [
#     {
#         "Name": "postgres-data",
#         "Driver": "local",
#         "Mountpoint": "/var/lib/docker/volumes/postgres-data/_data",
#         "CreatedAt": "2026-01-06T10:30:00Z"
#     }
# ]
```

### Creating Volumes

```powershell
# Create with default settings
docker volume create mydata

# Create with specific driver (advanced)
docker volume create --driver local mydata
```

### Removing Volumes

```powershell
# Remove specific volume
docker volume rm mydata

# Remove all unused volumes
docker volume prune

# Force remove (skip confirmation)
docker volume prune -f
```

### Finding Unused Volumes

```powershell
# See which volumes are not being used
docker volume ls -f dangling=true
```

---

## 💾 Backup and Restore Volumes

### Backup a Volume (Windows)

```powershell
# Create backup directory
mkdir C:\backups

# Backup postgres-data volume
docker run --rm `
  -v postgres-data:/data `
  -v C:/backups:/backup `
  ubuntu `
  tar czf /backup/postgres-backup.tar.gz /data

# What this does:
# --rm                     Remove container after done
# -v postgres-data:/data   Mount volume to backup
# -v C:/backups:/backup    Mount Windows folder for output
# tar czf ...              Create compressed archive
```

**Breaking it down:**
```
┌─────────────────────────────────────────────┐
│ Temporary Ubuntu Container                  │
│                                             │
│  /data/  ◄── postgres-data volume          │
│     ↓                                       │
│  tar czf (compress)                         │
│     ↓                                       │
│  /backup/ ◄── C:\backups (Windows)         │
│                                             │
└─────────────────────────────────────────────┘

Result: C:\backups\postgres-backup.tar.gz
```

### Restore a Volume (Windows)

```powershell
# Create new empty volume
docker volume create postgres-data-restored

# Restore from backup
docker run --rm `
  -v postgres-data-restored:/data `
  -v C:/backups:/backup `
  ubuntu `
  tar xzf /backup/postgres-backup.tar.gz -C /

# What this does:
# tar xzf     Extract compressed archive
# -C /        Extract to root (preserves /data/ path)
```

### Complete Backup/Restore Example

```powershell
# ========================================
# SCENARIO: Backup existing database
# ========================================

# 1. Create and populate database
docker volume create mydb-volume
docker run -d --name mydb -v mydb-volume:/var/lib/postgresql/data -e POSTGRES_PASSWORD=secret postgres:15
Start-Sleep -Seconds 10
docker exec mydb psql -U postgres -c "CREATE TABLE test (id INT); INSERT INTO test VALUES (1);"

# 2. Backup the volume
mkdir C:\db-backups -Force
docker run --rm -v mydb-volume:/data -v C:/db-backups:/backup ubuntu tar czf /backup/mydb.tar.gz /data

# 3. Delete everything (simulate disaster)
docker rm -f mydb
docker volume rm mydb-volume

# 4. Restore from backup
docker volume create mydb-volume
docker run --rm -v mydb-volume:/data -v C:/db-backups:/backup ubuntu tar xzf /backup/mydb.tar.gz -C /

# 5. Start database with restored data
docker run -d --name mydb -v mydb-volume:/var/lib/postgresql/data -e POSTGRES_PASSWORD=secret postgres:15
Start-Sleep -Seconds 10

# 6. Verify data is back!
docker exec mydb psql -U postgres -c "SELECT * FROM test;"
# ✅ Shows: id = 1 (Data restored!)
```

---

## 🔄 Sharing Volumes Between Containers

### Multiple Containers, One Volume

```powershell
# Create shared volume
docker volume create shared-data

# Container 1: Writer
docker run -d `
  --name writer `
  -v shared-data:/data `
  alpine `
  sh -c "while true; do date >> /data/log.txt; sleep 5; done"

# Container 2: Reader
docker run -d `
  --name reader `
  -v shared-data:/data `
  alpine `
  sh -c "while true; do tail -n 5 /data/log.txt; sleep 5; done"

# Check writer logs
docker logs -f writer

# Check reader logs
docker logs -f reader
# ✅ Reader sees files written by Writer!
```

**Visual:**
```
Container 1 (Writer)        Container 2 (Reader)
┌──────────────┐            ┌──────────────┐
│ Write data   │            │  Read data   │
└──────┬───────┘            └──────┬───────┘
       │                           │
       └──────► shared-data ◄──────┘
                ┌──────────────┐
                │ log.txt      │
                │ (Shared!)    │
                └──────────────┘
```

---

## 🎯 Practical Exercises (Windows)

### Exercise 1: MySQL with Persistent Volume

```powershell
# Create volume
docker volume create mysql-data

# Run MySQL
docker run -d `
  --name mysql-db `
  -e MYSQL_ROOT_PASSWORD=rootpass `
  -e MYSQL_DATABASE=testdb `
  -e MYSQL_USER=user `
  -e MYSQL_PASSWORD=userpass `
  -v mysql-data:/var/lib/mysql `
  -p 3306:3306 `
  mysql:8.0

# Wait for startup
Start-Sleep -Seconds 20

# Create table and add data
docker exec mysql-db mysql -uuser -puserpass testdb -e "
  CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2)
  );
  INSERT INTO products (name, price) VALUES 
  ('Laptop', 999.99),
  ('Mouse', 29.99);
"

# Verify data
docker exec mysql-db mysql -uuser -puserpass testdb -e "SELECT * FROM products;"

# Delete container
docker rm -f mysql-db

# Create new container with same volume
docker run -d `
  --name mysql-db2 `
  -e MYSQL_ROOT_PASSWORD=rootpass `
  -v mysql-data:/var/lib/mysql `
  -p 3306:3306 `
  mysql:8.0

Start-Sleep -Seconds 20

# Data still there!
docker exec mysql-db2 mysql -uroot -prootpass testdb -e "SELECT * FROM products;"
```

### Exercise 2: Development Environment

```powershell
# Create React-like dev environment
mkdir C:\react-dev
cd C:\react-dev

# Create simple HTML/JS app
@"
<!DOCTYPE html>
<html>
<head>
    <title>Dev App</title>
    <script>
        function updateTime() {
            document.getElementById('time').textContent = new Date().toLocaleTimeString();
        }
        setInterval(updateTime, 1000);
    </script>
</head>
<body onload="updateTime()">
    <h1>Development Mode Active</h1>
    <p>Time: <span id="time"></span></p>
    <p>Edit this file and refresh!</p>
</body>
</html>
"@ | Out-File index.html -Encoding UTF8

# Run with live reload
docker run -d `
  --name dev-server `
  -p 8080:80 `
  -v ${PWD}:/usr/share/nginx/html `
  nginx:alpine

Start-Process "http://localhost:8080"

# Edit index.html, save, refresh browser - see changes!
```

### Exercise 3: Backup and Restore

```powershell
# Complete backup/restore workflow
# 1. Create volume with data
docker volume create important-data
docker run --rm -v important-data:/data alpine sh -c "echo 'Important info!' > /data/file.txt"

# 2. Backup
mkdir C:\volume-backups -Force
docker run --rm -v important-data:/data -v C:/volume-backups:/backup alpine tar czf /backup/data.tar.gz /data

# 3. Verify backup exists
ls C:\volume-backups
# Should show: data.tar.gz

# 4. Delete volume (simulate loss)
docker volume rm important-data

# 5. Restore
docker volume create important-data
docker run --rm -v important-data:/data -v C:/volume-backups:/backup alpine tar xzf /backup/data.tar.gz -C /

# 6. Verify restoration
docker run --rm -v important-data:/data alpine cat /data/file.txt
# ✅ Shows: Important info!
```

---

## 📊 Volume Type Comparison

| Type | Managed By | Use Case | Persistence | Performance |
|------|-----------|----------|-------------|-------------|
| **Named Volume** | Docker | Databases, prod data | ✅ High | ⚡ Fast |
| **Bind Mount** | You | Development, config | ✅ High | ⚡ Fast |
| **Anonymous** | Docker | Temporary data | ⚠️ Medium | ⚡ Fast |

---

## ✅ Best Practices

### DO ✅

1. **Use named volumes for databases**
   ```powershell
   docker run -v postgres-data:/var/lib/postgresql/data postgres
   ```

2. **Use bind mounts for development**
   ```powershell
   docker run -v ${PWD}:/app node:18
   ```

3. **Backup important volumes regularly**
   ```powershell
   docker run --rm -v mydata:/data -v C:/backups:/backup ubuntu tar czf /backup/backup.tar.gz /data
   ```

4. **Clean up unused volumes**
   ```powershell
   docker volume prune
   ```

### DON'T ❌

1. **Don't store data in containers without volumes**
   - Data will be lost when container is deleted

2. **Don't use anonymous volumes in production**
   - Hard to manage, easy to lose

3. **Don't forget to backup volumes**
   - Volumes can be corrupted or deleted

4. **Don't bind mount sensitive files**
   - Security risk if container is compromised

---

## 🎓 Summary

**Three Volume Types:**
1. **Named** → Production databases (`docker volume create mydata`)
2. **Bind Mount** → Development (`-v ${PWD}:/app`)
3. **Anonymous** → Temporary data (`-v /data`)

**Key Concepts:**
- Volumes persist data outside containers
- Data survives container deletion
- Multiple containers can share volumes
- Bind mounts allow live code editing
- Always backup important volumes

**Next:** Day 7 - Docker Networking! 🌐

Want to practice with a specific scenario? I can create custom exercises! 🚀