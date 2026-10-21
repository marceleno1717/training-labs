# Lab 5: Docker Image Vulnerability Scanning with Trivy

### 1) Clone the repository
```bash
git clone https://github.com/Ibrahim-Adel15/TrivyImage.git
cd TrivyImage
```

### 2) Build initial Docker image from provided Dockerfile
```bash
docker build -t trivy-lab:v1 .
```

### 3) Install Trivy and verify installation
```bash
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sudo sh -s -- -b /usr/local/bin
trivy --version
```

### 4) Scan the Docker image for vulnerabilities
```bash
trivy image trivy-lab:v1
```

### 5) Scan the image to identify only HIGH and CRITICAL vulnerabilities
```bash
trivy image --severity HIGH,CRITICAL trivy-lab:v1
```

### 6) Generate HTML vulnerability report
```bash
trivy image --format template --template "@html.tpl" -o ti.html trivy-lab:v1
```

---

### Bonus: Vulnerability Remediation

### 7) Fix identified vulnerabilities (Dockerfile-opti)
Create an optimized Dockerfile (`Dockerfile-opti`) using a modern Alpine base image, updating package repositories, stripping unnecessary build tooling (`npm`) from the final runtime, and running as non-root user:

```dockerfile
FROM node:26-alpine

WORKDIR /app

RUN apk update && apk upgrade --no-cache

COPY package.json ./

RUN npm install --omit=dev
RUN rm -rf /usr/local/lib/node_modules/npm /usr/local/bin/npm /usr/local/bin/npx /root/.npm

COPY app.js ./

USER node
EXPOSE 3000

CMD ["node", "app.js"]
```

### 8) Rebuild Docker image with remediation
```bash
docker build -f Dockerfile-opti -t ivolve-app:v2 .
```

### 9) Rescan the updated image and generate new HTML report
```bash
trivy image --format template --template "@html.tpl" -o tiopt.html ivolve-app:v2
```

### 10) Verification & Comparison

#### Images Size Comparison:
```bash
docker images | grep -E "trivy-lab|ivolve-app"
```
```
REPOSITORY    TAG   IMAGE ID       SIZE
ivolve-app    v2    7510c0ba0178   286MB
trivy-lab     v1    63363e846863   1.46GB
```

#### Vulnerability Scan Results:
- **`trivy-lab:v1`** (`node:18-bullseye`): Over 2,200 vulnerabilities detected across base OS and packages.
- **`ivolve-app:v2`** (`node:26-alpine` optimized): **0 HIGH or CRITICAL vulnerabilities** (0 vulnerabilities in Alpine OS & Node packages).
