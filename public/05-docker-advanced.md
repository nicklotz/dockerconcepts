# Advanced Docker Concepts

## A. Running Docker in Rootless Mode

> Installing Docker in rootless mode allows running Docker without requiring root access to the host. Rootless mode uses **user namespaces** (a Linux kernel feature that maps container UIDs to unprivileged host UIDs) so Docker can run the daemon as non-root. This improves security by preventing container escapes from gaining root privileges on the host.

> **NOTE:** The example commands shown here are intended for a Linux system. Docker Desktop and Rancher on Mac systems run Docker within a virtual machine on the host.

### Prerequisites (Ubuntu 22.04 LTS)

Before setting up rootless Docker, install the required dependency:

```
# Install uidmap (provides newuidmap/newgidmap for user namespace mapping)
sudo apt-get update && sudo apt-get -y install uidmap
```

> **Note for Ubuntu 23.10+:** Newer Ubuntu versions require additional AppArmor configuration for rootlesskit. See the [Docker rootless documentation](https://docs.docker.com/engine/security/rootless/) for details.

### Setup Steps

1. Stop the docker daemon.
```
systemctl stop docker
```

2. Disable Docker services so Docker in root mode doesn't automatically start after reboot.
```
sudo systemctl disable --now docker.service docker.socket
```

3. Reboot the system.
```
sudo reboot
```

4. After reboot, install Docker rootless. Follow the instructions provided by the installation if you are missing any dependencies such as **uidmap** (provides user namespace mapping utilities) or need to restart services. You might need to rerun the installer a couple times, following the instructions after each time.
```
# -f: fail silently on HTTP errors
# -s: silent mode (no progress)
# -S: show errors when silent
# -L: follow redirects
curl -fsSL https://get.docker.com/rootless | sh
```

> After installation completes, you must configure your environment. Add the following to your `~/.bashrc`:

```
# Add rootless Docker binaries to PATH
export PATH=/home/$USER/bin:$PATH

# Point Docker CLI to the rootless daemon socket
export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock
```

```
# Apply the changes to your current session
source ~/.bashrc
```

5. After installation, show the available Docker contexts.
```
docker context ls
```

> A Docker **context** is a named connection target for the CLI — a (daemon socket, TLS config) pair that the `docker` command knows how to talk to. The listing here should show both the **default** root-based context and the new **rootless** one. Switching context is how you change which daemon a `docker` command operates against, without restarting anything.

6. Show the currently used context.
```
docker context show
```

> The current context should be **rootless**.

7. Run a container in rootless mode.
```
# --rm removes the container after it exits
# --user specifies the UID:GID to run as inside the container
# $UID is your user ID, $(id -g $USER) gets your group ID
docker run --rm --user $UID:$(id -g $USER) ubuntu whoami
```

> What user does the **whoami** command output? It should show your username (e.g., "ubuntu") or a numeric UID - but NOT "root". This demonstrates the container process is running as a non-root user. Without the `--user` flag, containers default to running as root inside the container.

8. Switch the Docker context back to default.
```
docker context use default
```

## B. Lightweight Docker Images

> Many distributions offer a **slim** version of their image, which strips out non-essential packages, documentation, and development tools. Slim images can be 50-70% smaller than full images, reducing storage costs and attack surface while improving pull times.

1. Create two Dockerfiles, each to run two different Node builds.
```
cat << EOF > Dockerfile.debian
FROM node:bookworm
EOF
```
```
cat << EOF > Dockerfile.slim
FROM node:bookworm-slim
EOF
```

2. Run image builds of both containers.
```
docker build -f Dockerfile.debian -t node-debian .
docker build -f Dockerfile.slim -t node-slim .
```

3. Compare the sizes of each image.
```
# grep filters output to show only lines containing "node-"
# You should see a significant size difference between the two images
docker images | grep node-
```

> The slim image should be significantly smaller. There's a trade-off here: slim images strip out common debugging tools (`curl`, `vim`, sometimes even most of a shell), so when something breaks in production you may not have your usual diagnostic kit available. For even smaller images, look at **Alpine** (5–10MB base, swaps glibc for musl) or **distroless** images from Google (no shell at all — just your binary and its runtime). Each step down trades convenience for less storage cost, faster pulls, and a smaller attack surface.

4. Clean up.
```
docker stop $(docker ps -a -q)
```
```
docker rm $(docker ps -a -q)
```
```
docker image rm $(docker image ls -q)
```
