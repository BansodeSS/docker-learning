# Day 2: Basic Docker Commands

## 📖 What You'll Learn Today
- Essential Docker CLI commands
- Working with images and containers
- Container lifecycle management
- Inspecting and monitoring containers

## 🎯 Core Concepts

### Images vs Containers
- **Image**: A template/blueprint (like a class in OOP)
- **Container**: A running instance of an image (like an object)

```
Image (nginx)  →  docker run  →  Container (running nginx)
```

## 📚 Essential Commands

### Working with Images

```bash
# List all local images
docker images
# or
docker image ls

# Pull an image from Docker Hub
docker pull nginx
docker pull nginx:latest
docker pull nginx:1.24

# Search for images on Docker Hub
docker search nginx

# Remove an image
docker rmi nginx:latest
# or
docker image rm nginx:latest

# Remove all unused images
docker image prune
```

### Working with Containers

```bash
# Run a container (pulls image if not local)
docker run nginx

# Run container in detached mode (background)
docker run -d nginx

# Run with a custom name
docker run -d --name my-nginx nginx

# Run with port mapping (host:container)
docker run -d -p 8080:80 nginx

# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# Stop a container
docker stop my-nginx
# or by container ID
docker stop abc123

# Start a stopped container
docker start my-nginx

# Restart a container
docker restart my-nginx

# Remove a container
docker rm my-nginx

# Remove a running container (force)
docker rm -f my-nginx

# Remove all stopped containers
docker container prune
```

### Inspecting Containers

```bash
# View container logs
docker logs my-nginx

# Follow logs (real-time)
docker logs -f my-nginx

# View container details
docker inspect my-nginx

# View resource usage statistics
docker stats

# View running processes in container
docker top my-nginx

# Execute command in running container
docker exec my-nginx ls /usr/share/nginx/html

# Get interactive shell in container
docker exec -it my-nginx /bin/bash
# or
docker exec -it my-nginx sh
```

## 💻 Hands-on Exercises

### Exercise 1: Basic Container Operations
```bash
# Pull ubuntu image
docker pull ubuntu:22.04

# Run ubuntu container interactively
docker run -it --name my-ubuntu ubuntu:22.04 /bin/bash

# Inside container, try:
pwd
ls
cat /etc/os-release
exit

# List all containers
docker ps -a

# Restart and attach to container
docker start my-ubuntu
docker attach my-ubuntu
```

### Exercise 2: Web Server Practice
```bash
# Run nginx web server
docker run -d -p 8080:80 --name webserver nginx

# Check if it's running
docker ps

# Test in browser: http://localhost:8080

# View nginx logs
docker logs webserver

# Execute command inside container
docker exec webserver cat /etc/nginx/nginx.conf

# Get shell access
docker exec -it webserver /bin/bash

# Stop and remove
docker stop webserver
docker rm webserver
```

### Exercise 3: Multiple Containers
```bash
# Run multiple nginx containers on different ports
docker run -d -p 8081:80 --name web1 nginx
docker run -d -p 8082:80 --name web2 nginx
docker run -d -p 8083:80 --name web3 nginx

# List all running containers
docker ps

# Stop all containers
docker stop web1 web2 web3

# Remove all
docker rm web1 web2 web3
```

## 📊 Command Reference Cheat Sheet

| Command | Description |
|---------|-------------|
| `docker pull <image>` | Download image from registry |
| `docker run <image>` | Create and start container |
| `docker ps` | List running containers |
| `docker ps -a` | List all containers |
| `docker stop <container>` | Stop running container |
| `docker start <container>` | Start stopped container |
| `docker restart <container>` | Restart container |
| `docker rm <container>` | Remove container |
| `docker rmi <image>` | Remove image |
| `docker logs <container>` | View container logs |
| `docker exec -it <container> <cmd>` | Execute command in container |
| `docker inspect <container>` | View detailed info |

## 🎓 What I Learned Today

**Date**: ___________

**Commands Practiced**:
- [ ] Pulled images from Docker Hub
- [ ] Created and ran containers
- [ ] Managed container lifecycle
- [ ] Inspected running containers
- [ ] Accessed container shells

**Practice Results**:
```
[Paste outputs of your commands]
```

**Challenges Faced**:
```
[Any errors or confusion]
```

**Key Insights**:
1. Difference between `docker run` and `docker start`:
   
2. When to use `-d` flag:
   
3. Port mapping syntax:
   

## 🧪 Try These Challenges

1. Run a MySQL database container (check Docker Hub for instructions)
2. Run a container that automatically removes itself after stopping (`--rm` flag)
3. Run a container with environment variables (`-e` flag)
4. Find and use 3 different images from Docker Hub

## 📝 Notes

### Common Flags
- `-d`: Detached mode (background)
- `-it`: Interactive terminal
- `-p`: Port mapping (host:container)
- `--name`: Custom container name
- `--rm`: Auto-remove when stopped
- `-e`: Environment variable
- `-v`: Volume mount

### Tips
- Container names must be unique
- Use `docker ps -a` to see stopped containers
- Container IDs can be shortened (first 3-4 chars usually enough)
- Use `docker system prune` to clean up everything unused

---

**Previous**: [Day 1 - Installation](../day-01-installation/README.md)  
**Next**: [Day 3 - Running Your First Container](../day-03-first-container/README.md)