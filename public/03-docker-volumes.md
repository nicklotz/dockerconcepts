# Working with Docker Containers

## A. Isolation

> By default, multiple containers created from the same image are isolated from each other. Let's prove this.

1. Launch two separate containers from the same Ubuntu image.
```
docker run -d --name container1 ubuntu sleep 1000
```
```
docker run -d --name container2 ubuntu sleep 1000
```

2. Verify the containers running as separate instances and the launched processes have different PIDs.
```
docker ps
```
```
docker top container1
```
```
docker top container2
```

> Similarly, container filesystems are completely isolated from each other.

3. Create a file in **container1**.
```
docker exec -it container1 bash -c "echo 'Hello from Container1' > /isolated_file.txt"
```

4. Check of the file exists in **container2**.
```
docker exec -it container2 bash -c "cat /isolation_test.txt"
```

> It is shown that, absent any shared storage, containers from the same image do not share filesystem resources.

## B. Port Mapping

> Docker container also by default have their networks isolated from the underlying host.

1. Create a web service container from an NGINX image. 
```
docker run -d --name webserver0 nginx
```


