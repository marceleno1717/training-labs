# Lab 7: Multi-Network Microservices, Volumes, and Bind Mounts

### 1) Clone the application code
```bash
git clone https://github.com/Ibrahim-Adel15/MicroserviceApp.git
cd MicroserviceApp
```

### 2) Create Dockerfiles for frontend, backend, and notification services
Create `frontend/Dockerfile`:
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 3000
CMD ["python", "app.py"]
```

Create `backend/Dockerfile`:
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
RUN mkdir -p /app/data /app/logs
EXPOSE 5000
CMD ["python", "app.py"]
```

Create `notification/Dockerfile`:
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
RUN mkdir -p /app/logs
EXPOSE 5001
CMD ["python", "app.py"]
```

### 3) Build Docker images for each service
```bash
docker build -t lab7-frontend:latest ./frontend
```
```bash
docker build -t lab7-backend:latest ./backend
```
```bash
docker build -t lab7-notification:latest ./notification
```

### 4) Create bridge networks with separate subnets
```bash
docker network create --driver bridge --subnet 10.0.1.0/24 frontend-backend
```
```bash
docker network create --driver bridge --subnet 10.0.2.0/24 backend-notification
```

Verify networks:
```bash
docker network ls
```
```
NETWORK ID     NAME                 DRIVER    SCOPE
1227709f752d   frontend-backend     bridge    local
228848cd2b13   backend-notification bridge    local
```

### 5) Create Docker volumes for logs
```bash
docker volume create backendlogs
```
```bash
docker volume create notilogs
```

Verify volumes:
```bash
docker volume ls
```
```
DRIVER    VOLUME NAME
local     backendlogs
local     notilogs
```

### 6) Create host directory for backend persistent data bind mount
```bash
mkdir -p /home/marceleno/ivolve-labs/docker/lab7/backend-data
```

### 7) Run the notification container on backend-notification network
```bash
docker run -d --name notification --network backend-notification -v notilogs:/app/logs -p 5001:5001 lab7-notification:latest
```

### 8) Run the backend container on both networks and configure mounts
Run backend on the `frontend-backend` network with bind mount and volume:
```bash
docker run -d --name backend --network frontend-backend -v /home/marceleno/ivolve-labs/docker/lab7/backend-data:/app/data -v backendlogs:/app/logs -e NOTIFICATION_URL=http://notification:5001/notify -p 5000:5000 lab7-backend:latest
```

Connect backend to the `backend-notification` network:
```bash
docker network connect backend-notification backend
```

Verify backend connected networks:
```bash
docker inspect backend --format '{{json .NetworkSettings.Networks}}'
```

### 9) Run the frontend container on frontend-backend network
```bash
docker run -d --name frontend --network frontend-backend -e BACKEND_URL=http://backend:5000 -p 3000:3000 lab7-frontend:latest
```

### 10) Verify running containers
```bash
docker ps
```
```
CONTAINER ID   IMAGE                      COMMAND           STATUS         PORTS                    NAMES
a74d9bf2cb86   lab7-frontend:latest       "python app.py"   Up 30 minutes  0.0.0.0:3000->3000/tcp   frontend
03f099a1ee1e   lab7-backend:latest        "python app.py"   Up 30 minutes  0.0.0.0:5000->5000/tcp   backend
d769f4fc65bb   lab7-notification:latest   "python app.py"   Up 30 minutes  0.0.0.0:5001->5001/tcp   notification
```

### 11) Verify application functionality and connectivity

#### Test creating a message via Backend:
```bash
curl -s -X POST -H "Content-Type: application/json" -d '{"message":"Lab 7 verification"}' http://localhost:5000/messages
```
```json
{"message":"Lab 7 verification","status":"success"}
```

#### Test retrieving messages via Backend:
```bash
curl -s http://localhost:5000/messages
```
```json
{"messages":["alo","tes","zxczxc","fdsdfvvdfdfvsdfdfv","123","Lab 7 verification"]}
```

#### Access Web UI via Frontend:
```bash
curl -s http://localhost:3000
```
```html
<!DOCTYPE html>
<html>
<head>
    <title>Lab 7 - Microservices</title>
</head>
<body>
    <h1>Docker Microservices Lab</h1>

    <h2>Create a Message</h2>

    <form action="/send" method="POST">
        <input type="text" name="message" placeholder="Enter your message" required>
        <button type="submit">Send</button>
    </form>

    <h2>Messages</h2>
    <ul>
        <li>alo</li>
        <li>tes</li>
        <li>zxczxc</li>
        <li>fdsdfvvdfdfvsdfdfv</li>
        <li>123</li>
        <li>Lab 7 verification</li>
    </ul>

    <a href="/">Refresh Messages</a>
</body>
</html>
```

#### Verify Notification service received the event:
```bash
docker logs --tail 3 notification
```
```
2026-10-22 16:42:45,501 INFO Notification received for message: Lab 7 verification
2026-10-22 16:42:45,568 INFO 10.0.2.2 - - [22/Oct/2026 16:42:45] "POST /notify HTTP/1.1" 200 -
```
