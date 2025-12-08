# Docker Compose for Development

## A. Introduction to Docker Compose

> Docker Compose is a tool for defining and running multi-container Docker applications. It uses a YAML file to configure your application's services, networks, and volumes.

### Why Docker Compose?

| **Challenge** | **Docker Compose Solution** |
|--------------|---------------------------|
| Running multiple containers manually | Define all services in one file |
| Complex docker run commands | Declarative YAML configuration |
| Container networking | Automatic network creation |
| Service dependencies | Depends_on and health checks |
| Environment management | Environment files and variables |

## B. Docker Compose Basics

1. Verify Docker Compose is installed.
```
docker compose version
```

2. Create a directory for a multi-service application.
```
mkdir ~/mycomposeapp
cd ~/mycomposeapp
```

3. Create a simple Flask application.
```
cat << 'EOF' > app.py
from flask import Flask
import redis
import os

app = Flask(__name__)
cache = redis.Redis(host=os.getenv('REDIS_HOST', 'redis'), port=6379)

@app.route('/')
def hello():
    count = cache.incr('hits')
    return f'Hello! This page has been viewed {count} times.\n'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
```

4. Create the requirements file.
```
cat << EOF > requirements.txt
flask>=2.0
redis>=4.0
EOF
```

5. Create a Dockerfile for the Flask app.
```
cat << EOF > Dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
EOF
```

6. Create a Docker Compose file.
```
cat << EOF > docker-compose.yml
services:
  web:
    build: .
    ports:
      - "5000:5000"
    depends_on:
      - redis
    environment:
      - REDIS_HOST=redis

  redis:
    image: redis:alpine
EOF
```

7. Build and start the services.
```
docker compose up --build -d
```

8. Verify the services are running.
```
docker compose ps
```

9. Test the application.
```
curl http://localhost:5000
curl http://localhost:5000
curl http://localhost:5000
```

> Each request should increment the counter, demonstrating that the Flask app is communicating with Redis.

10. View the logs from all services.
```
docker compose logs
```

11. Stop and remove all containers.
```
docker compose down
```

## C. Working with Databases in Development

> Docker Compose is excellent for running databases alongside your application during development.

1. Create a new project directory.
```
mkdir ~/mydbapp
cd ~/mydbapp
```

2. Create a Flask app that connects to PostgreSQL.
```
cat << 'EOF' > app.py
from flask import Flask, jsonify
import psycopg2
import os

app = Flask(__name__)

def get_db_connection():
    conn = psycopg2.connect(
        host=os.getenv('DB_HOST', 'db'),
        database=os.getenv('DB_NAME', 'myapp'),
        user=os.getenv('DB_USER', 'postgres'),
        password=os.getenv('DB_PASSWORD', 'postgres')
    )
    return conn

@app.route('/health')
def health():
    try:
        conn = get_db_connection()
        conn.close()
        return jsonify({"status": "healthy", "database": "connected"})
    except Exception as e:
        return jsonify({"status": "unhealthy", "error": str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
```

3. Create the requirements file.
```
cat << EOF > requirements.txt
flask>=2.0
psycopg2-binary>=2.9
EOF
```

4. Create the Dockerfile.
```
cat << EOF > Dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
EOF
```

5. Create a Docker Compose file with PostgreSQL.
```
cat << EOF > docker-compose.yml
services:
  web:
    build: .
    ports:
      - "5000:5000"
    depends_on:
      db:
        condition: service_healthy
    environment:
      - DB_HOST=db
      - DB_NAME=myapp
      - DB_USER=postgres
      - DB_PASSWORD=postgres

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
EOF
```

6. Start the services.
```
docker compose up --build -d
```

7. Wait for all services to be healthy and test.
```
docker compose ps
```
```
curl http://localhost:5000/health
```

8. Clean up.
```
docker compose down -v
```

## D. Using .env Files

> Environment files allow you to externalize configuration without modifying the compose file.

1. Create a new directory.
```
mkdir ~/myenvapp
cd ~/myenvapp
```

2. Create a .env file.
```
cat << EOF > .env
APP_PORT=8080
DB_PASSWORD=supersecret
DEBUG=true
EOF
```

3. Create a docker-compose.yml that references the environment variables.
```
cat << EOF > docker-compose.yml
services:
  web:
    image: nginx:alpine
    ports:
      - "\${APP_PORT}:80"
    environment:
      - DEBUG=\${DEBUG}
EOF
```

4. Start the service.
```
docker compose up -d
```

5. Verify the port mapping.
```
docker compose ps
```
```
curl http://localhost:8080
```

6. Clean up.
```
docker compose down
```

## E. Development vs Production Configurations

> Docker Compose supports multiple configuration files for different environments.

1. Create a base compose file.
```
mkdir ~/myenvspecificapp
cd ~/myenvspecificapp
```
```
cat << EOF > docker-compose.yml
services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
EOF
```

2. Create a development override file.
```
cat << EOF > docker-compose.dev.yml
services:
  web:
    volumes:
      - ./html:/usr/share/nginx/html:ro
    environment:
      - DEBUG=true
EOF
```

3. Create a production override file.
```
cat << EOF > docker-compose.prod.yml
services:
  web:
    deploy:
      replicas: 3
    environment:
      - DEBUG=false
EOF
```

4. Create a test HTML file.
```
mkdir html
echo "<h1>Development Mode</h1>" > html/index.html
```

5. Start with development configuration.
```
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d
```

6. Test the development setup.
```
curl http://localhost
```

7. Clean up.
```
docker compose down
```

## F. Docker Compose Commands Reference

| **Command** | **Description** |
|-------------|-----------------|
| `docker compose up` | Create and start containers |
| `docker compose up -d` | Start in detached mode |
| `docker compose up --build` | Rebuild images before starting |
| `docker compose down` | Stop and remove containers |
| `docker compose down -v` | Also remove volumes |
| `docker compose ps` | List containers |
| `docker compose logs` | View output from containers |
| `docker compose logs -f` | Follow log output |
| `docker compose exec SERVICE CMD` | Execute command in running container |
| `docker compose build` | Build or rebuild services |
| `docker compose pull` | Pull service images |
| `docker compose restart` | Restart services |

## G. Clean Up

Remove all resources created in this lab.

```
cd ~
rm -rf mycomposeapp mydbapp myenvapp myenvspecificapp
docker system prune -f
```
