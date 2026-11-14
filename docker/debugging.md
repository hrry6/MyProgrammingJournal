# Debugging & Troubleshooting

[🏠 Home](../README.md) > [🐳 Docker](./README.md)

## Monitoring & Inspection

```bash
# Live CPU, memory, and network I/O usage
docker stats

# Inspect full JSON metadata (IP address, mounted volumes, env vars, health status)
docker inspect my-app

# Extract only the container's IP address
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-app

# Check processes running inside a container
docker top my-app
```

## Common Exit Codes

| Exit Code | Meaning | Typical Cause |
| :--- | :--- | :--- |
| `0` | Clean exit | Main process finished normally (or container had no foreground process) |
| `1` | Application error | Unhandled exception, invalid config, or failed startup command |
| `137` | `SIGKILL` (128 + 9) | Container killed by OS **OOMKiller** (out of memory) or `docker kill` |
| `143` | `SIGTERM` (128 + 15) | Graceful shutdown triggered by `docker stop` |

## Debugging a Crashing Container

If a container exits immediately before you can run `docker exec`:

```bash
# Override ENTRYPOINT to inspect the filesystem of a broken image
docker run --rm -it --entrypoint sh my-broken-image:latest
```
