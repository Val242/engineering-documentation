# DevOps Learning Journal — October 12, 2026

## Topic: Docker Fundamentals and Hands-on Container Management

### 1. Learning Objective

Today's objective was to strengthen my understanding of Docker images and containers, learn essential Docker commands, and apply these concepts through a practical deployment of Nginx on an AWS EC2 instance.

Although my motivation was lower than usual this morning, I remained disciplined and completed my planned learning session. This reinforced the importance of consistency and discipline even when motivation is not high.

### 2. Concepts Learned

**Docker Images**

I learned that a Docker image is a reusable package containing an application and the files, dependencies, and runtime components required to run it. The exact contents depend on how the image is built.

I also understood that multiple containers can be created from the same image, similar to how multiple EC2 instances can be launched from the same AMI.

**Docker Containers**

A container is a running or stopped instance created from an image. Containers provide isolated environments for applications to execute.

I learned the distinction between images and containers: removing a container does not automatically remove the image used to create it.

**Docker Image and Container Commands**

I practiced the purpose of the following commands:

- `docker pull` — downloads an image from a container registry.
- `docker run` — creates and starts a new container.
- `docker start` — starts an existing stopped container.
- `docker stop` — stops a running container.
- `docker ps` — lists running containers.
- `docker ps -a` — lists all containers, including stopped ones.
- `docker images` — lists locally available images.
- `docker logs` — displays a container's logged output.
- `docker exec -it` — executes a command interactively inside a running container.
- `docker rm` — removes a container.

**Container Logs and Interactive Access**

I learned how to inspect application logs to understand startup behavior, incoming HTTP requests, and errors. I also practiced entering a running container through an interactive shell to explore its filesystem.

### 3. Practical Implementation: Nginx on AWS EC2

I deployed the official Nginx Docker image on an Ubuntu EC2 instance.

**Environment and configuration**
- Cloud provider: AWS
- Compute service: Amazon EC2
- Operating system: Ubuntu
- Container image: `nginx:latest`
- Container name: `nginx-lab`
- Host port: `3001`
- Container port: `80`

I pulled the Nginx image, created and started the container, and configured port mapping to access the web server through the EC2 instance's public IP address.

I verified that the Nginx welcome page was accessible in my browser.

### 4. Logs and Troubleshooting

I inspected the Nginx container logs and observed:

- Successful Nginx initialization and configuration.
- Nginx worker processes starting.
- An HTTP GET request to `/` returning status code `200`.
- A request for `/favicon.ico` returning status code `404` because the favicon file was not present.

I learned that the missing favicon did not indicate that the web server had failed; the main page was served successfully.

I also encountered minor command errors caused by misspelling container names and separating a container name incorrectly. I corrected these errors and continued the exercise.

### 5. Container Lifecycle Management

I practiced the complete container lifecycle:

1. Created and started the Nginx container.
2. Inspected its status using Docker commands.
3. Opened an interactive shell inside the container and explored its filesystem.
4. Exited the shell without stopping the container.
5. Stopped the container and verified that it appeared in the list of stopped containers.
6. Restarted the same container rather than creating another one.
7. Stopped and removed the container after completing the exercise.

Finally, I listed the local images and confirmed that `nginx:latest` remained available even after the container was removed.

### 6. Key Takeaways

- Docker images are reusable templates from which containers are created.
- Multiple containers can use the same image.
- `docker run` creates and starts a new container, while `docker start` starts an existing stopped container.
- Container logs help with application monitoring and troubleshooting.
- `docker exec -it` provides interactive access to a running container.
- Containers and images are separate resources with independent lifecycles.
- Docker can run on an AWS EC2 instance, allowing applications to be packaged and executed in isolated environments.
- Practical mistakes are opportunities to improve command-line accuracy and troubleshooting skills.

### 7. Reflection

Today's session helped me move beyond simply recognizing Docker commands to understanding how they work together in practice.

Deploying Nginx on an actual EC2 instance gave me hands-on experience with container creation, port mapping, log inspection, interactive access, stopping and restarting containers, and cleanup.

Most importantly, I continued learning despite not feeling highly motivated. I was reminded that progress in DevOps requires consistency, practical experimentation, and a willingness to troubleshoot problems independently.

### 8. Next Steps

- Continue learning Docker images, containers, and their underlying architecture.
- Practice writing Dockerfiles to package my own backend applications.
- Explore container networking and persistent storage.
- Apply Docker knowledge to application deployment and CI/CD workflows as my learning progresses.