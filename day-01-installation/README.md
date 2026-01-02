# Day 1: Docker Installation & Setup

## 📅 Date: __________

## 🎯 Objectives
- Install Docker
- Verify installation
- Understand Docker architecture

## 📝 Notes
[# Day 1: Docker Installation & Setup

## 📖 What You'll Learn Today
- What is Docker and why use it
- Docker architecture basics
- Installing Docker on your system
- Verifying installation
- Understanding Docker Desktop

## 🎯 Concepts

### What is Docker?
Docker is a platform for developing, shipping, and running applications in containers. Containers package your application with all its dependencies, ensuring it runs consistently across different environments.

### Key Benefits
- **Consistency**: "Works on my machine" → "Works everywhere"
- **Isolation**: Each container runs independently
- **Efficiency**: Lighter than virtual machines
- **Portability**: Run anywhere Docker is installed
- **Scalability**: Easy to scale up/down

### Docker Architecture
```
┌─────────────────────────────────────┐
│         Docker Client               │
│         (docker CLI)                │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│         Docker Daemon               │
│         (dockerd)                   │
│  ┌──────────┐  ┌──────────┐         │
│  │ Images   │  │Containers│         │
│  └──────────┘  └──────────┘         │
└─────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│         Docker Registry             │
│         (Docker Hub)                │
└─────────────────────────────────────┘
```

## 🛠️ Installation Steps

### For Windows
1. Download Docker Desktop from https://www.docker.com/products/docker-desktop
2. Run the installer
3. Enable WSL 2 (Windows Subsystem for Linux) if prompted
4. Restart your computer
5. Launch Docker Desktop

### For macOS
1. Download Docker Desktop for Mac
2. Drag to Applications folder
3. Launch Docker Desktop
4. Grant necessary permissions

### For Linux (Ubuntu/Debian)
```bash
# Update package index
sudo apt-get update

# Install prerequisites
sudo apt-get install ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add your user to docker group (to run without sudo)
sudo usermod -aG docker $USER
```

## ✅ Verification Commands

```bash
# Check Docker version
docker --version
# Expected output: Docker version 24.x.x, build xxxxx

# Check Docker Compose version
docker compose version
# Expected output: Docker Compose version v2.x.x

# Verify Docker is running
docker info

# Run hello-world container (test installation)
docker run hello-world
```

## 📝 Today's Exercise

1. Install Docker on your system
2. Run all verification commands
3. Take screenshots of successful outputs
4. Document any issues you faced and how you resolved them

## 🎓 What I Learned Today

**Date**: 2026/02/01

**Installation Experience**:
- [ ] Installation completed successfully
- [ ] Docker Desktop is running
- [ ] hello-world container ran successfully

**Challenges Faced**:
```
[Document any errors or issues here]
```

**Solutions Found**:
```
[wsl --update
wsl: WSL installation appears to be corrupted (Error code: Wsl/CallMsi/Install/REGDB_E_CLASSNOTREG).
Press any key to repair WSL, or CTRL-C to cancel.
This prompt will time out in 60 seconds.
Updating Windows Subsystem for Linux to version: 2.6.3.]
```

**Key Takeaways**:
1. 
2. 
3. 

## 🔗 Useful Resources
- [Docker Official Documentation](https://docs.docker.com/)
- [Docker Desktop Manual](https://docs.docker.com/desktop/)
- [Docker Hub](https://hub.docker.com/)

## 📸 Screenshots
*Add your screenshots here showing successful installation*

---

**Next**: [Day 2 - Basic Docker Commands](../day-02-basic-commands/README.md)]

## ✅ Tasks Completed
- [✅] Downloaded Docker Desktop
- [✅] Installed Docker
- [✅] Ran hello-world container
- [✅] Verified installation

## 💻 Commands Practiced
```bash
docker --version
docker run hello-world
docker ps
```

## 🐛 Issues Encountered
[C:\Users\suhas>docker run hello-world
docker: error during connect: Head "http://%2F%2F.%2Fpipe%2FdockerDesktopLinuxEngine/_ping": open //./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified.

Run 'docker run --help' for more information]

## 💡 Solutions Found
[How you solved them]

## 📸 Screenshots
[![docker-version](docker-version.jpg)]

## 🔗 References
- [Link to helpful resource 1]
- [Link to helpful resource 2]

## ⏭️ Next Steps
Tomorrow: Basic Docker Commands