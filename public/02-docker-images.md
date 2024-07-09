# Docker Images


## A. Image Immutability

1. Pull an Ubuntu Linux image and connect to a shell session inside it.
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

## B. Layers and Processes

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

> Why do you see "missing" as the image ID for most of the layers? *Hint*: how changeable are images once they are pushed to a public registry?

4. Run an Ubuntu container instance and launch a process inside the container.
```
docker run -d --name singleprocesscontainer ubuntu sleep infinity
```

5. Explor the process running in the container.
```
docker top singleprocesscontainer
```

## C. Build From an Existing Container

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
docker commit testcontainer testimage:v2
```

4. Stop the existing running **testcontainer**.
```
docker stop testcontainer
```

5. Launch a container from the newly saved image.
```
docker run -d --name testcontainerv2 testimage:v2
```

6. Inspect the container's filesystem.
```
docker exec -it testcontainerv2 ls
```

> Do the filesystem contents match that of the saved image?

## D. Building From Dockerfile

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
RUN apt-get update && apt-get install -y nginx
COPY . /usr/share/nginx/html
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


