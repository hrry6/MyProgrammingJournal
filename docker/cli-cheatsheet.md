# Docker CLI Cheatsheet

[🏠 Home](../README.md) > [🐳 Docker](./README.md)

## Container Lifecycle

```bash
# Run container in background (-d), map ports (-p host:container), and auto-remove on stop (--rm)
docker run -d --name my-app -p 8080:8080 my-image:latest

# List running containers (-a to include stopped containers)
docker ps -a

# Start, stop, or restart a container
docker start my-app
docker stop my-app
docker restart my-app

# Remove container (-f forces removal even if running)
docker rm -f my-app
```

## Logs & Shell Access

```bash
# Follow live logs (--tail limits initial lines)
docker logs -f --tail 100 my-app

# Open interactive shell inside a running container
docker exec -it my-app sh
# Or bash (for Debian/Ubuntu-based images)
docker exec -it my-app bash

# Run a one-off command without entering shell
docker exec my-app printenv
```

## Image Management

```bash
# Build image from Dockerfile in current directory
docker build -t my-app:1.0.0 .

# List local images
docker images

# Remove an image
docker rmi my-app:1.0.0
```

## Disk Cleanup (`prune`)

```bash
# Remove all stopped containers, unused networks, and dangling images
docker system prune

# Aggressive cleanup: remove ALL unused images (not just dangling) and build cache
docker system prune -a --volumes
```
