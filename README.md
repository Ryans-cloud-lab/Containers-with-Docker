# Containers-with-Docker

**Project:** Containerise a Node.js application with Docker and Docker Compose (MongoDB) and publish the image to a private Docker registry on Nexus.

**Technologies used:** Docker, Docker Compose, Linux, Node.js (Express), MongoDB, Mongo Express, Sonatype Nexus, DigitalOcean

**Project Description**

- Ran MongoDB and Mongo Express as containers on a user-defined Docker network to support local development
- Used core Docker CLI commands (run, pull, start, stop, logs, exec, ps) to manage the container lifecycle
- Mapped container ports to the host and orchestrated a multi-container stack with Docker Compose
- Persisted database data across container restarts using a named Docker volume
- Wrote a Dockerfile to build the Node.js application into a Docker image
- Created a private Docker registry on Nexus, then authenticated, tagged and pushed the application image to it
- Ran the full stack (application + database + admin UI) from Docker Compose using the image from the private registry


  ## Description

###### What is Docker? What is a container?

Docker is an open-source containerisation platform. It packages an application together with its dependencies, runtime and configuration into a single portable, standardised artifact (an image) that runs the same way in development, testing and production. Containers existed before Docker (for example Linux namespaces, cgroups and LXC) however Docker made the workflow accessible and mainstream.

###### Container registries

Container images are stored in container registries. Registries can be public (for example Docker Hub) or private (for example AWS ECR, Azure Container Registry or a Nexus Docker repository) for company-internal images. Images are pulled to be run and pushed so that other environments, teams or servers can consume the exact same artifact.

###### Docker image vs Docker container

- **Image:** the immutable, packaged artifact (a stack of read-only filesystem layers plus metadata). It is not running. Images are versioned with tags and moved between machines and registries. 
- **Container:** a running instance of an image, with its own process, a writable filesystem layer and network interfaces, plus port bindings so traffic can reach the application inside. An image becomes a container when it is started with docker run. Many containers can run from one image.

###### Containers vs virtual machines

*Both provide isolation, but at different levels:*

- **VMs** virtualise the hardware and run a full guest operating system (kernel + OS) per VM. They are larger (GBs), slower to start (minutes) and offer stronger isolation and OS flexibility. 
- **Containers** virtualise at the application layer and share the host's kernel, using Linux namespaces and cgroups. They are smaller (MBs) and start in seconds.

***Compatibility note:** Linux containers require a Linux kernel. On macOS and Windows, Docker Desktop runs a lightweight Linux VM so Linux containers can be built and run locally.*

###### Docker architecture (client, server, registry)

Docker bundles several components:

Docker CLI (client) on my machine sends commands to the Docker daemon (dockerd) through the Docker API. The daemon (engine) pulls and caches images, builds images from Dockerfiles, creates and runs containers, and manages networks and volumes. Registries host images; docker pull and docker push interact with them.

#### Prerequisites

- Docker installed locally: Docker Desktop on macOS/Windows, or Docker Engine on Linux
- Node.js and npm (only needed to run the app on the host while the databases run in containers)
- Example project: js-app, a Node.js + Express + MongoDB application with a Dockerfile, docker-compose.yaml and mongo.yaml

### Install Docker

I installed Docker Desktop on macOS (Apple Silicon) and verified the installation:

```bash
docker version
```

---

## Main Docker commands

These map to the handout’s core and debug commands:

| Command | Purpose |
|--------|---------|
| `docker run` | Create and start a container from an image |
| `docker pull` | Download an image from a registry |
| `docker start` / `docker stop` | Start or stop existing container(s) |
| `docker images` | List images stored locally |
| `docker ps` | List running containers; `docker ps -a` includes stopped |
| `docker logs` | View logs from a container |
| `docker exec -it` | Run a shell or command inside a running container (for debugging) |

Examples:

```bash
docker pull mongo:7
docker run -d --name mongodb -p 27017:27017 mongo:7
docker ps
docker logs mongodb
docker stop mongodb
```

---
