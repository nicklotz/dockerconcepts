# Docker Images


## A. Image Immutability

1. Pull an Ubuntu Linux image and connect to a shell session inside it.
```
docker run -it --name test-container ubuntu /bin/bash
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
docker images list
```

5. Store the local Ubuntu image ID as an environment variable.
```
IMAGE_ID=$(docker images ubuntu:latest --format "{{.ID}}")
```

6. Start a second container from the same Ubuntu image and connect to it.
```
docker run -it --name test-container2 $IMAGE_ID /bin/bash
```

7. Inspect the filesystem. Is a `/file1` there? Why or why not?
```
ls
```

8. Exit the container.
```
exit
```

##. B. Layers and Processes

1. 
