# Docker Networking

## A. Create and use a Docker network

> Docker exposes several **network drivers**: `bridge` (default — isolates containers on a single host), `host` (the container shares the host's network stack), `none` (no networking at all), and `overlay` (multi-host, covered in Section B). A *custom* bridge network like the one we're about to create has one practical advantage over the built-in `bridge`: containers attached to it can resolve each other's names via Docker's embedded DNS server, not just by IP.

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

> Because **mybridgenet** is a custom bridge with embedded DNS, we could just as well have written `docker exec nginx1 curl http://nginx2` — the service name resolves to the right IP automatically. The default `bridge` network does not offer this; on that network you'd be stuck passing IPs around (or using the deprecated `--link` flag).

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

> So far the containers we've created have only been reachable from inside Docker's networks. **Publishing a port** (`-p host:container`) is the standard way to expose a service to the host machine — and through the host to the outside world. **Container linking** (`--link`), explored second, is an older mechanism that predates user-defined networks: it injects environment variables and `/etc/hosts` entries into the linked container. It still works but has been deprecated, because user-defined bridge networks (Section A) cover the same use case more cleanly.

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

> So far every container has lived on a single Docker host. **Overlay networks** are how containers communicate when they're spread across multiple hosts. They require a control plane that tracks which container is on which host — which is why this section starts by initializing Docker Swarm: even with a single node, swarm mode is what enables the overlay driver. (Kubernetes solves the same problem with its own CNI plugins, but the underlying primitives are similar.)


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
