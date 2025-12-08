# Docker Networking

## A. Create and use a Docker network

1. Create a custom bridge network. Bridge networks provide isolated networking on a single Docker host, with automatic DNS resolution between containers.
```
docker network create --driver bridge mybridgenet
```

2. Create two NGINX containers on the **mybridgenet** network.
```
docker run -dit --name nginx1 --network mybridgenet nginx
docker run -dit --name nginx2 --network mybridgenet nginx
```

3. Inspect the new network with containers attached.
```
docker network inspect mybridgenet
```

4. Save the IP addresses of both containers.
```
# Using $() syntax instead of backticks for better readability
# The Go template extracts the IP address from the container's network settings
NGINX1_IP=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' nginx1)
NGINX2_IP=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' nginx2)
```
```
echo "IP address of nginx1 is $NGINX1_IP"
echo "IP address of nginx2 is $NGINX2_IP"
```

5. Test connectivity between the two containers.
```
docker exec nginx1 curl http://$NGINX2_IP
docker exec nginx2 curl http://$NGINX1_IP
```

6. Create a container **not** yet connected to your custom network.
```
docker run -dit --name nginx3 nginx
```

7. Connect the new container to your existing custom network.
```
docker network connect mybridgenet nginx3
```

8. Save the IP address of the third container.
```
NGINX3_IP=$(docker inspect nginx3 -f '{{.NetworkSettings.Networks.mybridgenet.IPAddress}}')
echo $NGINX3_IP
```

9. Test connectivity to nginx3 from another container.
```
docker exec nginx1 curl http://$NGINX3_IP
```

10. Disconnect nginx3 from your custom network.
```
docker network disconnect mybridgenet nginx3 
```

11. Try communicating with nginx3 again. Does it work?
```
docker exec nginx1 curl http://$NGINX3_IP
```

> The connection attempt should hang. Type **CTRL-C** to kill the connection attempt.

12. Clean up the resources you've created so far.
```
docker container stop nginx1 nginx2 nginx3
docker container rm nginx1 nginx2 nginx3
docker network rm mybridgenet
```

## B. Expose and link containers

1. Run a new NGINX container with port 80 exposed to port 8080 on localhost.
```
docker run -d --name nginxpublic -p 8080:80 nginx
```

2. Test accessibility by navigating to **https://localhost:8080** in your web browser or by using curl.
```
curl http://localhost:8080
```

> **Container linking** is a legacy feature that creates a direct connection between two containers. It's commonly seen in older applications but is deprecated in favor of user-defined networks with DNS resolution.

3. Test container linking.
```
docker run -d --name linkednginx --link nginxpublic:nginxpub nginx
```

4. Verify connectivity between the two linked containers.
```
NGINXPUB_IP=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' nginxpublic)
```
```
docker exec linkednginx curl http://$NGINXPUB_IP
```

> Docker Swarm uses **overlay networks** to enable communication between containers running on different Docker hosts. An overlay network creates a distributed network among multiple Docker daemon hosts, abstracting away the underlying network infrastructure.


5. Initialize Docker swarm.
```
docker swarm init
```

6. Create an overlay network.
```
# -d overlay specifies the overlay driver for multi-host networking
docker network create -d overlay myoverlaynet
```

7. Deploy an NGINX service to the overlay network.

```
docker service create --name mywebservice --network myoverlaynet nginx
```

By deploying a service to an overlay network, it can communicate across multiple hosts within the swam (if we had them).

8. Clean up resources.

```
docker service rm mywebservice
docker container stop linkednginx nginxpublic
docker container rm linkednginx nginxpublic
docker network rm myoverlaynet
docker swarm leave --force
```
