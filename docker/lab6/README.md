# Lab 6: Docker Environment Variables and Resource Management

### 1) Clone the application code
```bash
git clone https://github.com/Ibrahim-Adel15/PythonApp.git
cd PythonApp
```

### 2) Create Dockerfile
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

### 3) Build ivolve1 Docker image
```bash
docker build -t ivolve1 .
```

### 4) Run container1 with environment variables and resource limits
```bash
docker run -d -p 5000:5000 -e APP_NAME=iVolveApp1 -e APP_ENV=stg --cpus="1" --memory="256m" --name container1 ivolve1
```

### 5) Verify application is working
```bash
curl http://localhost:5000
```
```json
{
  "application": "iVolveApp1",
  "environment": "stg",
  "message": "Welcome to the Docker Env & Resource Management Lab!"
}
```

### 6) Verify environment variables inside container
```bash
docker exec -it container1 env
```
```bash
APP_NAME=iVolveApp1
APP_ENV=stg
```

### 7) Monitor container CPU and memory usage
```bash
docker stats container1 --no-stream
```
```
CONTAINER ID   NAME         CPU %     MEM USAGE / LIMIT   MEM %     NET I/O         BLOCK I/O   PIDS
8c29c06c7bdd   container1   0.01%     22.89MiB / 256MiB   8.94%     2.94kB / 829B   143kB / 0B  1
```

---

### 8) Create env.txt file
```text
APP_NAME=iVolveApp2
APP_ENV=prod
```

### 9) Run container2 using env.txt and ivolve1 image
```bash
docker run -d -p 5000:5000 --env-file env.txt --name container2 ivolve1
```

### 10) Verify container2 output and environment variables
```bash
curl http://localhost:5000
```
```json
{
  "application": "iVolveApp2",
  "environment": "prod",
  "message": "Welcome to the Docker Env & Resource Management Lab!"
}
```
```bash
docker exec -it container2 env
```
```bash
APP_NAME=iVolveApp2
APP_ENV=prod
```

---

### 11) Create Dockerfile-env with hardcoded environment variables
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
ENV APP_NAME=iVolveApp3
ENV APP_ENV=dev
EXPOSE 5000
CMD ["python", "app.py"]
```

### 12) Build ivolve3 Docker image
```bash
docker build -f Dockerfile-env -t ivolve3 .
```

### 13) Run container2 using ivolve3 image
```bash
docker run -d -p 5000:5000 --name container2 ivolve3
```

### 14) Verify container2 output and environment variables
```bash
curl http://localhost:5000
```
```json
{
  "application": "iVolveApp3",
  "environment": "dev",
  "message": "Welcome to the Docker Env & Resource Management Lab!"
}
```
```bash
docker exec -it container2 env
```
```bash
APP_NAME=iVolveApp3
APP_ENV=dev
```
