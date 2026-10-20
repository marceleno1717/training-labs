# Lab 3

### 1) Approach 1: Build the app inside Docker (Dockerfile-mvn)
```bash
docker build . -f Dockerfile-mvn -t with-mvn
```

### 2) Approach 2: Build the app locally then copy the jar (Dockerfile-jar)
```bash
docker build . -f Dockerfile-jar -t with-java 
```

### 3) Images Size Comparison
```bash
docker images
```
```
REPOSITORY   TAG      IMAGE ID       SIZE
with-mvn     latest   28425863cbf9   861MB
with-java    latest   702eb437f2f3   485MB
```
