# DevOps Learning Journal — October 13, 2026

## Topic: Docker Fundamentals, Volumes, Images, ENTRYPOINT, CMD, and Docker Compose

### 1. Learning Objectives

Today, I continued my hands-on Docker learning by exploring Docker images, persistent storage, container startup configuration, and an introduction to Docker Compose.

### 2. Topics Covered

- **Docker Images:** Learned more about using images as reusable packages for creating containers.
- **Docker Volumes:** Learned how Docker volumes help persist application data beyond the lifecycle of an individual container.
- **ENTRYPOINT and CMD:** Explored how these Dockerfile instructions define the executable a container runs and its default arguments or command.
- **Docker Compose:** Got an introduction to defining and managing application services through a YAML configuration file instead of managing each container independently.

### 3. Key Concepts Learned

**Docker Volumes**

Containers can be removed and recreated, so important data should not depend solely on a container's writable layer. Docker volumes provide persistent storage that can survive container removal.

**ENTRYPOINT and CMD**

These instructions control container startup behavior. `ENTRYPOINT` commonly defines the main executable, while `CMD` supplies default arguments or a default command. Their interaction depends on how they are defined and how the container is started.

**Docker Compose**

Docker Compose allows developers and DevOps engineers to define an application's services, networks, volumes, and configuration in a YAML file. The configuration can then be used to start and manage the application's services together.

### 4. Practical Significance

Today's topics helped me understand that Docker is more than simply creating and running containers. It also provides ways to preserve data, configure application startup behavior, and manage applications consisting of multiple services.

These concepts will be useful when deploying applications that depend on databases, caches, and other supporting services.

### 5. Next Steps

- Practise creating and mounting Docker volumes.
- Experiment with the interaction between `ENTRYPOINT` and `CMD`.
- Build a small application environment using Docker Compose.
- Continue practising container lifecycle management and image configuration.
- Apply these concepts to realistic deployment scenarios as my Docker knowledge develops.

### 6. Personal Reflection

Today, I expanded my understanding of Docker by moving from basic image and container operations into storage, startup configuration, and multi-service orchestration. I am gradually building a stronger foundation in containerization, which will support my broader DevOps and cloud engineering journey.