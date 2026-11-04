# Lab 22: CI/CD Pipeline Implementation with Jenkins Agents and Shared Libraries

### 1) Lab Overview & Objectives
This lab implements a modular, production-ready CI/CD workflow in Jenkins:
1. **Shared Pipeline Library (`shared-lib`)**:
   - Centralizes reusable Groovy pipeline steps (`buildJS`, `buildScanPushImage`, `k8sDeploy`) in an external Git repository (`jenkins-shared-lib`).
   - Configured in Jenkins under **Manage Jenkins > System > Global Pipeline Libraries**.
2. **Dedicated SSH Build Agent (`agent1`)**:
   - Jenkins controller offloads build workloads to a containerized SSH agent (`jenkins/ssh-agent:latest-jdk21`).
   - Agent is mounted with required host toolchains: Docker socket, Node/npm wrappers, Trivy, and kubectl with cluster access.
3. **Pipeline Workflow (`Jenkinsfile2`)**:
   - **Build App**: Executes `buildJS()` to install dependencies.
   - **Build, Scan & Push**: Executes `buildScanPushImage(...)` to build Docker image, scan for vulnerabilities using Trivy (`HIGH,CRITICAL`), log in and push to Docker Hub, and clean up local images.
   - **Deploy on K8s**: Executes `k8sDeploy(...)` to dynamically patch the deployment manifest, apply changes to Kubernetes, and await successful rollout.
   - **Post Actions**: Runs cleanup (`docker logout`) and outputs build status notifications.

---

### 2) Shared Library Implementation (`shared-lib`)
The shared library is hosted at `https://github.com/marceleno1717/jenkins-shared-lib` and structured under [`jenkins/lab22/shared-lib/vars/`](file:///home/marceleno/ivolve-labs/jenkins/lab22/shared-lib/vars):

#### `vars/buildJS.groovy`
```groovy
def call(){
    sh "npm install"
}
```

#### `vars/buildScanPushImage.groovy`
```groovy
def call(String imageName, String dockerCreds) {
    sh "docker build -t ${imageName} ."

    sh "trivy image --severity HIGH,CRITICAL --exit-code 0 ${imageName}"

    withCredentials([usernamePassword(credentialsId: dockerCreds, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
        sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
        sh "docker push ${imageName}"
    }

    sh "docker rmi ${imageName}"
}
```

#### `vars/k8sDeploy.groovy`
```groovy
def call(String imageName, String newTag, String deploymentFile) {
    sh """
    sed -i 's|image: ${imageName}:.*|image: ${imageName}:${newTag}|' ${deploymentFile}

    kubectl apply -f ${deploymentFile}

    DEPLOY_NAME=\$(awk '/^metadata:/{f=1;next} f&&/^  name:/{print \$2;exit}' ${deploymentFile})
    NAMESPACE=\$(awk '/^metadata:/{f=1;next} f&&/^  namespace:/{print \$2;exit}' ${deploymentFile})

    kubectl rollout status deployment/\${DEPLOY_NAME} -n \${NAMESPACE} --timeout=120s
    """
}
```

---

### 3) Jenkins SSH Agent Setup
A dedicated agent container runs using Docker Compose connected to the Minikube network so that both controller and agent share network connectivity.

#### Docker Compose Configuration (`jenkins/lab22/compose.yaml`)
```yaml
services:
  ssh-agent:
    image: jenkins/ssh-agent:latest-jdk21
    container_name: jenkins-agent
    environment:
      JENKINS_AGENT_SSH_PUBKEY: ${JENKINS_AGENT_SSH_PUBKEY}
      PATH: /opt/node/bin:/opt/java/openjdk/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    group_add:
      - "1001"
    volumes:
      - /home/marceleno/.nvm/versions/node/v24.19.0:/opt/node:ro
      - /home/marceleno/node-wrapper:/usr/local/bin/node:ro
      - /home/marceleno/npm-wrapper:/usr/local/bin/npm:ro
      - /home/marceleno/npx-wrapper:/usr/local/bin/npx:ro
      - /snap/kubectl/current/kubectl:/usr/local/bin/kubectl:ro
      - /home/marceleno/.local/bin/trivy:/usr/local/bin/trivy:ro
      - /usr/bin/docker:/usr/local/bin/docker:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - /home/marceleno/kubeconfig-agent:/home/jenkins/.kube/config:ro
    networks:
      - minikube

networks:
  minikube:
    external: true
```

#### Start Agent Container
```bash
docker compose -f jenkins/lab22/compose.yaml up -d
```

#### Jenkins Node Configuration (`agent1`)
In Jenkins under **Manage Jenkins > Nodes > New Node**:
- **Node name**: `agent1`
- **Remote root directory**: `/home/jenkins/agent`
- **Number of executors**: `2`
- **Launch method**: `Launch agents via SSH`
- **Host**: `jenkins-agent` (resolvable inside docker bridge network `minikube`)
- **Port**: `22`
- **Credentials**: `SSH Username with private key` (`jenkins-agent` using generated `id_rsa`)
- **Host Key Verification Strategy**: `Non verifying Verification Strategy`

---

### 4) Pipeline Definition (`Jenkinsfile2`)
Located at [`jenkins/lab22/iVolveApp/Jenkinsfile2`](file:///home/marceleno/ivolve-labs/jenkins/lab22/iVolveApp/Jenkinsfile2):

```groovy
@Library('shared-lib') _

pipeline {
    agent any

    environment {
        IMAGE_NAME = 'marceleno17/ivolveapp'
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKERHUB_CREDENTIALS_ID = 'docker'
        DEPLOYMENT_FILE = 'deployment.yaml'
    }

    stages {
        stage('1. Build App') {
            steps {
                echo "Installing application dependencies via shared library..."
                buildJS()
            }
        }

        stage('2. Build, Scan & Push Docker Image') {
            steps {
                echo "Building, scanning with Trivy, and pushing Docker image via shared library..."
                buildScanPushImage("${IMAGE_NAME}:${IMAGE_TAG}", "${DOCKERHUB_CREDENTIALS_ID}")
            }
        }

        stage('3. Deploy to Kubernetes Cluster') {
            steps {
                echo "Updating deployment manifest and applying to Kubernetes via shared library..."
                k8sDeploy("${IMAGE_NAME}", "${IMAGE_TAG}", "${DEPLOYMENT_FILE}")
            }
        }
    }

    post {
        always {
            echo "Pipeline run completed."
            sh 'docker logout || true'
        }
        success {
            echo "Pipeline succeeded! Application deployed with image ${IMAGE_NAME}:${IMAGE_TAG}."
        }
        failure {
            echo "Pipeline failed! Please inspect logs for details."
        }
    }
}
```

---

### 5) Pipeline Execution Output
Live console output from Jenkins build `#12` running on agent `agent1`:

```text
Started by user admin
Obtained Jenkinsfile2 from git https://github.com/marceleno1717/ivolveapp
Loading library shared-lib@main
Attempting to resolve main from remote references...
 > git ls-remote -h -- https://github.com/marceleno1717/jenkins-shared-lib # timeout=10
Found match: refs/heads/main revision 3b474d979ab986acc11951e94dab40296e145a8f
Cloning the remote Git repository
Cloning repository https://github.com/marceleno1717/jenkins-shared-lib
Checking out Revision 3b474d979ab986acc11951e94dab40296e145a8f (main)
[Pipeline] Start of Pipeline
[Pipeline] node
Running on agent1 in /home/jenkins/agent/workspace/ivolveapp2
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Declarative: Checkout SCM)
[Pipeline] checkout
using credential github
Cloning repository https://github.com/marceleno1717/ivolveapp
Checking out Revision cac768139c540f999e5e7248cf138d91c3f3011a (refs/remotes/origin/main)
Commit message: "jenkins file 2"
[Pipeline] }
[Pipeline] // stage
[Pipeline] withEnv
[Pipeline] {
[Pipeline] withEnv
[Pipeline] {
[Pipeline] stage
[Pipeline] { (1. Build App)
[Pipeline] echo
Installing application dependencies via shared library...
[Pipeline] sh
+ npm install

up to date, audited 85 packages in 3s

23 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (2. Build, Scan & Push Docker Image)
[Pipeline] echo
Building, scanning with Trivy, and pushing Docker image via shared library...
[Pipeline] sh
+ docker build -t marceleno17/ivolveapp:12 .
Sending build context to Docker daemon   7.95MB
Step 1/7 : FROM node:18-alpine
Step 2/7 : WORKDIR /app
Step 3/7 : COPY package.json ./
Step 4/7 : RUN npm install
Step 5/7 : COPY . .
Step 6/7 : EXPOSE 3000
Step 7/7 : CMD ["node", "server.js"]
Successfully built 039750b2ebfa
Successfully tagged marceleno17/ivolveapp:12
[Pipeline] sh
+ trivy image --severity HIGH,CRITICAL --exit-code 0 marceleno17/ivolveapp:12
2026-11-04T19:33:05Z	INFO	Table result includes only package filenames. Use '--format json' option to get the full path to the package file.

Report Summary

┌───────────────────────────────────────────────┬──────────┬─────────────────┬─────────┐
│                    Target                     │   Type   │ Vulnerabilities │ Secrets │
├───────────────────────────────────────────────┼──────────┼─────────────────┼─────────┤
│ marceleno17/ivolveapp:12 (alpine 3.21.3)      │  alpine  │       21        │    -    │
├───────────────────────────────────────────────┼──────────┼─────────────────┼─────────┤
│ app/node_modules/@types/node/package.json     │ node-pkg │        0        │    -    │
└───────────────────────────────────────────────┴──────────┴─────────────────┴─────────┘
[Pipeline] withCredentials
[Pipeline] {
[Pipeline] sh
+ echo ****
+ docker login -u marceleno17 --password-stdin
Login Succeeded
+ docker push marceleno17/ivolveapp:12
The push refers to repository [docker.io/marceleno17/ivolveapp]
12: digest: sha256:039750b2ebfae9034f89d38ad3cf4cf423c8a920e53a0601362e584f294ee5c9 size: 2057
[Pipeline] }
[Pipeline] // withCredentials
[Pipeline] sh
+ docker rmi marceleno17/ivolveapp:12
Untagged: marceleno17/ivolveapp:12
Deleted: sha256:039750b2ebfae9034f89d38ad3cf4cf423c8a920e53a0601362e584f294ee5c9
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (3. Deploy to Kubernetes Cluster)
[Pipeline] echo
Updating deployment manifest and applying to Kubernetes via shared library...
[Pipeline] sh
+ sed -i s|image: marceleno17/ivolveapp:.*|image: marceleno17/ivolveapp:12| deployment.yaml
+ kubectl apply -f deployment.yaml
deployment.apps/ivolveapp configured
+ awk /^metadata:/{f=1;next} f&&/^  name:/{print $2;exit} deployment.yaml
+ DEPLOY_NAME=ivolveapp
+ awk /^metadata:/{f=1;next} f&&/^  namespace:/{print $2;exit} deployment.yaml
+ NAMESPACE=ivolve
+ kubectl rollout status deployment/ivolveapp -n ivolve --timeout=120s
Waiting for deployment "ivolveapp" rollout to finish: 1 out of 2 new replicas have been updated...
deployment "ivolveapp" successfully rolled out
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Declarative: Post Actions)
[Pipeline] echo
Pipeline run completed.
[Pipeline] sh
+ docker logout
Removing login credentials for https://index.docker.io/v1/
[Pipeline] echo
Pipeline succeeded! Application deployed with image marceleno17/ivolveapp:12.
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```

---

### 6) Kubernetes Cluster Verification
Verify that the pods in namespace `ivolve` were updated and are running image `marceleno17/ivolveapp:12`:

```bash
kubectl get pods,svc,deploy -n ivolve -o wide
```

```text
NAME                             READY   STATUS    RESTARTS      AGE     IP               NODE           NOMINATED NODE   READINESS GATES
pod/ivolveapp-7656cff755-kbk84   1/1     Running   0             10m     10.244.205.209   minikube-m02   <none>           <none>
pod/ivolveapp-7656cff755-mwfkq   1/1     Running   0             11m     10.244.205.208   minikube-m02   <none>           <none>
pod/ivolvedb-0                   1/1     Running   2 (43h ago)   2d23h   10.244.205.201   minikube-m02   <none>           <none>
pod/network                      1/1     Running   2 (11h ago)   2d23h   10.244.120.111   minikube       <none>           <none>

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE     SELECTOR
service/ivolvaappsvc   ClusterIP   10.98.243.216   <none>        3000/TCP   4d1h    app=ivolveapp
service/ivolvedbsvc    ClusterIP   None            <none>        3306/TCP   6d23h   app=ivolvedb

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS   IMAGES                     SELECTOR
deployment.apps/ivolveapp   2/2     2            2           4d1h   app          marceleno17/ivolveapp:12   app=ivolveapp
```
