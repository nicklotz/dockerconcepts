# Introduction to Docker

## A. Install Docker (if not already)
Docker has a convenient installation script if you're on a *nix-based system. If you're on Windows, [Docker Desktop](https://docs.docker.com/desktop/install/windows-install/) is recommended.
1. In a terminal, download the Docker install script.

```
curl -fsSL https://get.docker.com -o get-docker.sh
```

2. Run the Docker install script.  

```
sh get-docker.sh
```

3. Set permissions so you can run Docker without sudo.

```
sudo groupadd docker
sudo usermod -aG docker $USER
```

4. For the permissions to take effect, logout and back in and/or open a new user shell.

5. Test the Docker installation.

```
docker run hello-world
```

## B. Deploy Sample Microservices App

1. Clone a sample microservices app that deploys a local news application.

```
git clone https://github.com/rithinch/event-driven-microservices-docker-example.git
```

2. Change into the repo's parent directory.

```
cd event-driven-microservices-docker-example/
```

3. Build the microservices app locally.

```
docker-compose build --no-cache
```

4. Bring up the application.

```
docker-compose up
```

5. Get a list of articles maintained by the application.

```
curl http://localhost:3000/api/articles
```

6. Get the list of users in the application's user database.

```
http://localhost:3002/api/users
```

7. List all the running containers.

```
docker ps -a
```

8. Show the application's container build configuration.

```
cat docker-compose.yml
```

> If the `user-management` container for whatever reason went offline, can people still retrieve articles from the site? What does this say about the advantages of microservices?

9. Kill the user-management container.

10. Call the articles API.
