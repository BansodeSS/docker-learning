# Docker Learning Resources

## 📚 Official Documentation
- [Docker Official Docs](https://docs.docker.com/)
- [Docker Hub](https://hub.docker.com/)
- [Docker Compose Docs](https://docs.docker.com/compose/)
- [Dockerfile Reference](https://docs.docker.com/engine/reference/builder/)

## 🎓 Learning Platforms
- [Docker's Getting Started Tutorial](https://docs.docker.com/get-started/)
- [Play with Docker](https://labs.play-with-docker.com/) - Free online playground
- [Docker Curriculum](https://docker-curriculum.com/)
- [Katacoda Docker Scenarios](https://www.katacoda.com/courses/docker)

## 📖 Books
- "Docker Deep Dive" by Nigel Poulton
- "Docker in Action" by Jeff Nickoloff
- "The Docker Book" by James Turnbull

## 🎥 Video Courses
- Docker Mastery on Udemy
- Docker for Developers (LinkedIn Learning)
- FreeCodeCamp Docker Tutorial (YouTube)

## 🛠️ Tools & Extensions
- **Docker Desktop** - GUI for Docker
- **VSCode Docker Extension** - Container management in VSCode
- **Portainer** - Web UI for Docker
- **Lazydocker** - Terminal UI for Docker
- **Dive** - Tool for exploring image layers

## 📝 Cheat Sheets

### Essential Commands
```bash
# Images
docker pull <image>          # Download image
docker images                # List images
docker rmi <image>          # Remove image
docker build -t <name> .    # Build image

# Containers
docker run <image>          # Create & start
docker ps                   # List running
docker ps -a               # List all
docker stop <container>    # Stop
docker start <container>   # Start
docker rm <container>      # Remove
docker logs <container>    # View logs
docker exec -it <container> <cmd>  # Execute command

# System
docker system df           # Disk usage
docker system prune       # Clean up
docker info              # System info

# Compose
docker compose up         # Start services
docker compose down       # Stop services
docker compose logs       # View logs
docker compose ps         # List services
```

### Dockerfile Instructions
```dockerfile
FROM        # Base image
RUN         # Execute command
COPY        # Copy files
ADD         # Copy (with extras)
WORKDIR     # Set directory
ENV         # Environment variable
EXPOSE      # Document port
CMD         # Default command
ENTRYPOINT  # Main executable
USER        # Set user
VOLUME      # Mount point
ARG         # Build argument
LABEL       # Metadata
```

### docker-compose.yml Template
```yaml
version: '3.8'

services:
  service1:
    build: ./path
    # or
    image: image:tag
    container_name: name
    ports:
      - "host:container"
    environment:
      - KEY=value
    volumes:
      - ./local:/container
    depends_on:
      - service2
    networks:
      - network1
    restart: unless-stopped

volumes:
  volume1:

networks:
  network1:
```

## 🔧 Common Dockerfile Patterns

### Node.js App
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

### Python App
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["python", "app.py"]
```

### Go App
```dockerfile
FROM golang:1.21-alpine AS build
WORKDIR /app
COPY go.* ./
RUN go mod download
COPY . .
RUN go build -o main .

FROM alpine:latest
WORKDIR /app
COPY --from=build /app/main .
EXPOSE 8080
CMD ["./main"]
```

### Java App
```dockerfile
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 🐛 Troubleshooting Guide

### Container Won't Start
```bash
# Check logs
docker logs <container>

# Try running interactively
docker run -it <image> /bin/sh

# Check if port is already in use
netstat -ano | findstr :8080  # Windows
lsof -i :8080                 # Linux/Mac
```

### Build Failures
```bash
# Build with no cache
docker build --no-cache -t <image> .

# Verbose output
docker build --progress=plain -t <image> .

# Check Dockerfile syntax
docker build --check .
```

### Network Issues
```bash
# List networks
docker network ls

# Inspect network
docker network inspect bridge

# Check container network
docker inspect <container> | grep -i network
```

### Volume Issues
```bash
# List volumes
docker volume ls

# Inspect volume
docker volume inspect <volume>

# Check permissions
docker run -it -v <volume>:/data alpine ls -la /data
```

## 🎯 Best Practices Summary

### DO ✅
- Use specific image tags
- Implement multi-stage builds
- Run as non-root user
- Use .dockerignore
- Keep images small
- One process per container
- Use health checks
- Document exposed ports
- Version your images
- Clean up regularly

### DON'T ❌
- Don't use `latest` tag in production
- Don't store secrets in images
- Don't run unnecessary services
- Don't ignore security updates
- Don't use root user
- Don't make images too large
- Don't forget to clean up
- Don't skip health checks

## 🔐 Security Checklist
- [ ] Scan images for vulnerabilities
- [ ] Use minimal base images
- [ ] Run as non-root user
- [ ] Keep base images updated
- [ ] Don't store secrets in images
- [ ] Use secrets management
- [ ] Limit container resources
- [ ] Enable read-only root filesystem
- [ ] Drop unnecessary capabilities

## 📊 Performance Tips
- Optimize layer caching
- Use multi-stage builds
- Minimize layer count
- Use .dockerignore
- Choose appropriate base images
- Clean up in same RUN command
- Use specific COPY commands
- Leverage build cache

## 🌐 Community Resources
- [Docker Community Forums](https://forums.docker.com/)
- [Docker Subreddit](https://reddit.com/r/docker)
- [Stack Overflow Docker Tag](https://stackoverflow.com/questions/tagged/docker)
- [Docker GitHub](https://github.com/docker)
- [Awesome Docker](https://github.com/veggiemonk/awesome-docker)

## 🚀 Next Steps After This Course
1. **Container Orchestration**: Learn Kubernetes
2. **CI/CD**: Integrate Docker into pipelines
3. **Microservices**: Build distributed systems
4. **Cloud**: Deploy to AWS ECS, Azure Container Instances, GCP Cloud Run
5. **Monitoring**: Prometheus, Grafana, ELK Stack
6. **Security**: Docker Bench, Trivy, Clair

## 📝 Project Ideas to Practice
1. Personal blog with CMS
2. E-commerce platform
3. Task management app
4. Chat application
5. File sharing service
6. API gateway
7. Monitoring dashboard
8. CI/CD pipeline
9. Microservices demo
10. Full-stack CRUD app

---

Keep this guide handy as your Docker reference! Update it with your own learnings and discoveries.