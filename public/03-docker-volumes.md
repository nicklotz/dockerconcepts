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

> Docker container also by default run on networks networks separate from the underlying host.

1. Create a web service container from an NGINX image. 
```
docker run -d --name webserver0 nginx
```

2. Inspect the details of the running **webserver0** container.
```
docker inspect webserver0
```

> At the bottom of the output, what do you see regarding the container's networking configuration?

3. Store the container IP address as a variable.
```
WEBSERVER_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' webserver0)
```
```
echo $WEBSERVER_IP
```

4. Use **curl** to query the container's IP address. Alternatively, navigate to the container's IP address with your browser.
```
curl $WEBSERVER_IP
```

> While accessing Docker bridge network works from the Docker host machine, complications arise when you want the container to be accessible to computers outside host. One solution is to map container ports to host ports.

5. Remove the existing **webserver0** container.

```
docker stop webserver0
```
```
docker rm webserver0
```

6. Re-launch the **webserver0** container with a port mapping to the localhost.
```
docker run -d -p 8080:80 --name webserver0 nginx 
```

7. Open http://localhost:8080 in your browser or **curl** from the command line.
```
curl http://localhost:8080
```

8. Clean up the current container.
```
docker stop webserver0
```
```
docker rm webserver0
```

# C. Environment Variables

> Docker supports environment variables that can be defined in the application code and in the Dockerfile itself. Let's practice with a simple Flask app.

1. Create a directory to hold a Python web app and accompanying Dockerfile.
```
mkdir ~/myenvvarapp
```

2. Navigate into the **myenvvarapp** directory.
```
cd ~/myenvvarapp/
```

3. Run the following to create a Flask app that displays a short message on its homepage.
```
cat << EOF > app.py
from flask import Flask
import os

app = Flask(__name__)

@app.route('/')
def home():
    app_env_var = os.getenv('MY_APP_ENV_VAR', 'John Doe')
    return f'Hello, {app_env_var}\n'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
EOF
```

4. Run the following to create a **requirements.txt** listing the appropriate Flask package and version.
```
cat << EOF > requirements.txt
Flask >= 2.2.2
EOF
```

5. Run the following to create a Dockerfile that builds the Flask app into a container image.
```
cat << EOF > Dockerfile
FROM python:3.8-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
EOF
```

6. Build and tag the Docker image.
```
docker build -t myenvvarapp .
```

7. Run a container instance of the Flask app, first specifying no value for the environment variable.
```
docker run -d -p 8080:8080 --name myenvvarapp myenvvarapp
```

8. Open **http://localhost:8080** in your browser or run the following **curl** command.
```
curl http://localhost:8080
```

> What message do you see? Why do you see it?

9. Stop and remove the existing Flask container.
```
docker stop myenvvarapp
```
```
docker rm myenvvarapp
```

10. Run the following command, and enter your first name when prompted.
```
read -p "Enter your first name: " MY_FIRST_NAME
```

11. Run a new container instance of the Flask app, now specifying a value for the environment variable.
```
docker run -d -p 8080:8080 -e MY_APP_ENV_VAR=$MY_FIRST_NAME --name myenvvarapp myenvvarapp
```

12. Open **http://localhost:8080** in your browser or run the following **curl** command.
```
curl http://localhost:8080
```

> What message do you see? Why do you see it?

13. Clean up the running Flask container.
```
docker stop myenvvarapp
```
```
docker rm myenvvarapp
```
