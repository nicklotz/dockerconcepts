# Advanced Docker Concepts

## A. Running Docker in Rootless Mode

> Installing Docker in rootless mode allows running Docker without requiring root access to the host. Rootless mode uses **user namespaces** (a Linux kernel feature that maps container UIDs to unprivileged host UIDs) so Docker can run the daemon as non-root. This improves security by preventing container escapes from gaining root privileges on the host.

> **NOTE:** The example commands shown here are intended for a Linux system. Docker Desktop and Rancher on Mac systems run Docker within a virtual machine on the host.

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

> If needed, follow the instructions at the end of the installation such as updating your **$PATH**.

5. After installation, show the available Docker contexts.
```
docker context ls
```

> Note the presence of the default (root-based) as well as rootless contexts.

6. Show the currently used context.
```
docker context show
```

> The current context should be **rootless**.

7. Run a container in rootless mode.
```
# --user specifies the UID:GID to run as inside the container
# $UID is your user ID, $(id -g $USER) gets your group ID
docker run --user $UID:$(id -g $USER) -it ubuntu whoami
```

> What user does the **whoami** command output? It should show a numeric UID rather than "root", demonstrating the container is not running as the root user.

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

> The slim image should be significantly smaller. This demonstrates the trade-off: slim images save space but may be missing tools you need for debugging.

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
