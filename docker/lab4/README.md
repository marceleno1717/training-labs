# Lab 4

### 1) Build Docker Image (Multi-stage Build & Optimization)
```bash
docker build -t multistage .
```

### 2) Run the container
```bash
docker run -d -p 8080:8080 --name multistage-app multistage
```

### 3) Save image to tar archive
```bash
docker save -o multistage.tar multistage
```

### 4) Load image from tar archive
```bash
docker load -i multistage.tar
```

### 5) Images Size Comparison (Multi-stage vs Single-stage with Maven)
```bash
docker images
```
```
REPOSITORY   TAG      IMAGE ID       SIZE
with-mvn     latest   28425863cbf9   861MB
multistage   latest   d5e682af1cb7   522MB
```
