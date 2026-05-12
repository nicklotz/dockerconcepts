# Docker Images

## A. Image Immutability

> A core property of a Docker image: once built, the filesystem it contains is fixed. When you run a container, you get a thin **writable layer** on top of those read-only image layers — anything you write there belongs to the container, not the image. The next steps prove this by writing a file inside one container and then starting a fresh container from the same image to look for it.

1. Pull an Ubuntu Linux image and connect to a shell session inside it.
```
cd ~
```
```
docker run -it --name testcontainer ubuntu /bin/bash
```

2. Inside the container, create a new file.
```
touch /file1
```

3. Exit the container.
```
exit
```

4. Inspect the locally stored images pulled from Docker.
```
docker image list
```

5. Store the local Ubuntu image ID as an environment variable.
```
# The --format flag uses Go templating to extract just the image ID
# {{.ID}} outputs only the ID field from the image listing
IMAGE_ID=$(docker images ubuntu:latest --format "{{.ID}}")
```

6. Start a second container from the same Ubuntu image and connect to it.
```
docker run -it --name testcontainer2 $IMAGE_ID /bin/bash
```

7. Inspect the filesystem. Is a `/file1` there? Why or why not?
```
ls
```

8. Exit the container.
```
exit
```

> The second container started clean — `/file1` lived only in the writable layer of **testcontainer**, which was discarded when that container exited. The image itself was never modified, so any container launched from it always begins from the same state. This is what people mean when they say containers are *reproducible*.

## B. Layers and Processes

> Docker images are built as a **stack of read-only layers**, each capturing the filesystem changes from one build instruction. When a container starts, Docker mounts those layers together with a writable layer on top, using a *union filesystem*. The same base layer is shared by every image and container that uses it — that's why a hundred Ubuntu containers don't consume a hundred times the disk of one.

1. Explore the layers in the **hello-world** image.
```
docker history hello-world
```

2. Explore the layers in the **ubuntu** image.
```
docker history ubuntu
```

3. Explore the layers in the **example-voting-app-vote** image.
```
docker history example-voting-app-vote
```

> Why do you see "missing" as the image ID for most of the layers? This is because Docker only stores layer IDs for images built locally. Layers pulled from a registry show as "missing" because they were built elsewhere. Images are **immutable** once pushed to a registry - the layers cannot be changed.

4. Run an Ubuntu container instance and launch a process inside the container.
```
docker run -d --name singleprocesscontainer ubuntu sleep infinity
```

5. Explore the process running in the container.
```
docker top singleprocesscontainer
```

6. Stop the running container.
```
docker stop singleprocesscontainer
```

## C. Build From an Existing Container

> `docker commit` snapshots a running container's filesystem into a new image. It's useful for ad-hoc experimentation — "save what I have right now" — but it's a poor fit for production. The resulting image is opaque: there's no record of what was installed, in what order, or from what source. Dockerfiles (Section D) solve that problem by making the build steps explicit and reproducible.

1. Restart the Ubuntu **testcontainer**.
```
docker start testcontainer
```

2. Modify the running container by adding a new file
```
docker exec -it testcontainer touch /file2
```

3. Save and tag the modified container as a new image.
```
# docker commit creates a new image from a container's changes
# This adds a new layer on top of the base image containing /file2
docker commit testcontainer testimage:v2
```

4. Stop the existing running **testcontainer**.
```
docker stop testcontainer
```

5. Launch a container from the newly saved image.
```
docker run -dit --name testcontainerv2 testimage:v2
```

6. Inspect the container's filesystem.
```
docker exec -it testcontainerv2 ls
```

> Do the filesystem contents match that of the saved image? Yes - you should see both `/file1` and `/file2` because the committed image captured all filesystem changes from the original container.

7. Stop the container.
```
docker stop testcontainerv2
```

## D. Building From Dockerfile

> A **Dockerfile** is a text recipe for building an image. `FROM` declares the base image; `RUN` executes a command and commits the result as a new layer; `COPY` brings files from the build context into the image; `CMD` sets the default command for containers started from the image. Because each instruction creates a layer, ordering matters for **caching** — put rarely-changing instructions (package installs) above frequently-changing ones (your source code), so an edit to your code doesn't invalidate the install layer.

1. Create a local directory to hold a container build configuration.
```
mkdir mynginx/
```

2. Change into the **mynginx/** directory.
```
cd mynginx/
```

3. Create a Dockerfile that builds a custom NGINX image from an Ubuntu base OS.
```
cat << EOF > Dockerfile
# Dockerfile
FROM ubuntu
# Combine apt-get update and install in one RUN to reduce layers
# Using && ensures both commands succeed; -y auto-confirms prompts
RUN apt-get update && apt-get install -y nginx
COPY . /usr/share/nginx/html
# Run nginx in foreground (daemon off) so container doesn't exit
CMD ["nginx", "-g", "daemon off;"]
EOF
```

4. Build a Docker image from the local Dockerfile.
```
docker build -t mynginx .
```

5. View the new **mynginx** image among your local images.
```
docker image list
```

6. Inspect the new **mynginx** image.
```
docker image inspect mynginx
```

## E. Multi-Stage Builds

> A **multi-stage build** uses multiple `FROM` instructions in one Dockerfile. Each `FROM` starts a fresh image, and you can `COPY --from=<stage>` to selectively pull artifacts out of an earlier stage. The win is that the final image ships only the compiled output, not the toolchain that built it — that shrinks the image, speeds pulls, and removes a class of attack surface (no `gcc` lying around for an intruder to misuse).

1. Create a new directory to hold the configuration for a simple web app.
```
cd ~
```
```
mkdir mysimpleapp/
```

2. Change into the new configuration directory.
```
cd mysimpleapp/
```

3. Create a short and simple C++ program.
```
cat << EOF > main.cpp
#include <iostream>
int main() {
    std::cout << "Hello, World!" << std::endl;
    return 0;
}
EOF
```

4. Create a Dockerfile containing the configuration for a multi-stage build.
```
cat << EOF > Dockerfile
# Build stage
FROM gcc:latest AS builder
WORKDIR /app
COPY main.cpp .
RUN g++ -o main main.cpp

# Final stage
FROM ubuntu:latest
COPY --from=builder /app/main /app/main
CMD ["/app/main"]
EOF
```

5. Build the Docker image.
```
docker build -t mysimpleapp .
```

6. Run the container, executing the program.
```
docker run mysimpleapp
```

## F. Docker Hub

> A **registry** is where built images live and get distributed; Docker Hub is the default public one. Authenticating with an **access token** rather than your account password is the standard practice: tokens are scoped (e.g., read-only vs. push-and-delete), individually revocable, and don't expose the credentials you use to log into the web UI.

1. Navigate to Docker Hub in your web browser.
```
https://hub.docker.com
```

2. Either **Sign In** to your existing account or **Sign Up** for a new account if needed.

3. Once logged in, select your user file in the upper right corner, then click **Account Settings**. 

4. Select the **Security** tab. Under **Access Tokens**, click **New Access Token**.

5. Under **Access Token Description**, enter a description of your choice. Give the token **Read, Write, and Delete** permissions, then click **Generate**.

6. In the window showing the generated credential, click **Copy and Close** to copy the token to your clipboard. Paste the token in a safe place you can refer back to.

7. Back in your terminal, run the following to create an environment variable for your Docker username. 
```
read -p "Enter your name [Richard]: " MY_DOCKER_USERNAME
```

8. Authenticate to Docker Hub.
```
docker login
```

9. When prompted for a **Username**, enter your Docker ID or email address.

10. When prompted for a **Password**, paste the token copied and saved in step 6.

11. After logging in, run the following to verify Docker has locally stored a Docker Hub registry credential.
```
cat ~/.docker/config.json
```

> To upload a local Docker image to Docker Hub, the name must include the path in the registry where it will live.

12. Tag your local Docker image to include your Docker Username as well as an explicit version tag.
```
docker tag mysimpleapp $MY_DOCKER_USERNAME/mysimpleapp:0.0.1
```

13. Push to Docker Hub.
```
docker push $MY_DOCKER_USERNAME/mysimpleapp:0.0.1
```

14. In Docker Hub, navigate to **Repositories** and click into your **mysimpleapp** image repository. Click into the 0.0.1 image tag.

> Do the image layers resemble the image you pushed?

## G. Clean Up

> `docker image prune` removes **dangling** images — image layers no longer referenced by a tag, usually leftovers from rebuilds. `docker system prune` goes further, removing stopped containers, unused networks, and the build cache as well. Routine cleanup keeps disk usage in check on a development machine.

Run the following to clean up unused images and containers. Enter `y` if prompted.

```
docker image prune
docker system prune
```
