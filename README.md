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
### Port Mapping
- **Container port:** the port the process listens on inside the container. 
- **Host port:** the port opened on my machine (or server) that clients connect to.
- ###### Example: -p 3001:3000 forwards host port 3001 to container port 3000.

##### Workflow with Docker

A typical flow is: write code → write a Dockerfile → docker build to produce an image → docker tag and docker push to a registry → pull and docker run (or docker compose up) on a laptop or server. In CI/CD, a pipeline automates the build, tag and push stages, so every environment runs the same versioned artifact.

### Docker Compose

Docker Compose defines and runs multi-container applications from a single YAML file.

- Services, ports, environment variables, volumes and dependencies are declared in one version-controlled file, which is easier to maintain and review than long docker run commands. 
- Compose automatically creates a default network for the services in the file, so containers resolve each other by service name without --net. depends_on controls start order only, not readiness. Mongo Express can start before MongoDB accepts connections, which is why restart: always is set on it.

This repo includes:

- [js-app/docker-compose.yaml](./js-app/docker-compose.yaml) — MongoDB, **named volume** `mongo-data`, and **mongo-express** (UI on host port **8081** per the file).
- [js-app/mongo.yaml](./js-app/app/mongo.yaml) — example including a tagged **custom app image**; replace the image reference with your own registry host and tag when you deploy.

Start the stack from `js-app`:

```bash
cd js-app
docker-compose -f docker-compose.yaml up -d
```

Docker Compose V2 also accepts `docker compose` (with a space) if your Docker CLI includes the plugin. Use `docker-compose down` (or `docker compose down`) to stop and remove containers (volumes persist unless you remove them explicitly).

## Dockerfile

A Dockerfile is a set of instructions that docker build uses to create an image. Each instruction creates a layer, and unchanged layers are reused from cache to speed up rebuilds.

- **Build context:** the directory you pass at the end of `docker build` (often `.`). Only copy what you need; use a **`.dockerignore`** to exclude build artifacts and secrets.
- **Base image:** most Dockerfiles start `FROM` an existing image (for example `node:20-alpine` in [js-app/Dockerfile](./js-app/Dockerfile)).

Build the sample image from `js-app` (the trailing `.` is the context path):

```bash
cd js-app
docker build -t my-app:1.0 .
```

In CI/CD, the same Dockerfile usually builds the image artifact that is then **pushed** to a registry and **pulled** on servers or developer laptops.

### Private registries and image references

Working with a **private** registry (for example **AWS ECR**, **Azure Container Registry**, or a registry hosted on **Nexus**) usually follows:

1. **`docker login`** to the registry (authenticate).
2. **`docker tag`** so the image name includes **`registryDomain/imageName:tag`**.
3. **`docker push`** to upload the image.

**Default registry:** if you omit a registry host, Docker assumes **`docker.io`** (Docker Hub). For example:

```bash
docker pull mongo:4.2
```

is equivalent to pulling from Docker Hub’s library namespace. For a private registry you must include the host (and port if non-default), for example:

```text
165.245.222.56:8083/my-app:1.0
```

### Deploy Docker containers on a remote server

On a deployment host, Compose typically pulls a mix of:

- Public images (for example MongoDB and Mongo Express from Docker Hub). 
- Private images (the application, built and pushed to a private registry).

The server needs Docker, network access to the registry (docker login first), firewall rules for the published host ports, and an image built for its CPU architecture.

### Docker volumes

By default, data written inside a container lives in its writable layer and is lost when the container is removed. Volumes store data outside the container's lifecycle.

- **Host volume:** you choose the host path, e.g. `-v /home/mount/data:/var/lib/mysql/data`.
- **Anonymous volume:** Docker picks a directory under `/var/lib/docker/volumes/…`.
- **Named volume:** Docker manages storage under a **name** you choose—good for **production** and Compose (`volumes:` in YAML).

Named volume example from [js-app/docker-compose.yaml](./js-app/docker-compose.yaml):

```yaml
volumes:
  mongo-data:
    driver: local
```

I used a named volume, mounted at MongoDB's data directory, so the database survives the stack being recreated

### Docker best practices (security and maintainability)

- Use official or trusted base images, and pin specific tags (not latest) for reproducible builds.
- Prefer minimal base images (for example Alpine) to reduce image size and attack surface.
- Order Dockerfile instructions for layer caching: copy package*.json and install dependencies before copying source code.
- Use a .dockerignore to keep node_modules, .git and secrets out of the build context. U
- se multi-stage builds to keep build tools out of the final image.
- Run the container as a non-root user (USER node in Node images). 
