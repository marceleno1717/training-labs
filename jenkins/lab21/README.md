# Lab 21: Jenkins Pipeline for Application Deployment

### 1) Pipeline Overview
This pipeline automates the end-to-end CI/CD lifecycle for the Node.js application (`iVolveApp`):
1. **Build App**: Installs dependencies (`npm install`).
2. **Build Docker Image**: Builds the Docker image tagged with build number (`marceleno17/ivolveapp:${BUILD_NUMBER}`).
3. **Scan Docker Image**: Scans the built image with Trivy for HIGH and CRITICAL vulnerabilities.
4. **Push Image to DockerHub**: Logs in and pushes the tagged image to Docker Hub repository.
5. **Delete Image Locally**: Removes the local image to conserve disk space on the build node.
6. **Update Image in Deployment File**: Uses `sed` to update the image tag dynamically in `deployment.yaml`.
7. **Deploy to Kubernetes Cluster**: Applies the updated `deployment.yaml` and verifies rollout status in namespace `ivolve`.
8. **Post Actions**: Handles `always`, `success`, and `failure` pipeline states.

---

### 2) Jenkinsfile Implementation
The pipeline script is located at [`jenkins/lab21/iVolveApp/Jenkinsfile`](file:///home/marceleno/ivolve-labs/jenkins/lab21/iVolveApp/Jenkinsfile):

```groovy
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'marceleno17/ivolveapp'
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKERHUB_CREDENTIALS_ID = 'docker'
    }

    stages {
        stage('1. Build App') {
            steps {
                echo "Installing application dependencies..."
                sh 'npm install'
            }
        }

        stage('2. Build Docker Image') {
            steps {
                echo "Building Docker image ${IMAGE_NAME}:${IMAGE_TAG}..."
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('3. Scan Docker Image') {
            steps {
                echo "Scanning Docker image with Trivy..."
                sh "trivy image --severity HIGH,CRITICAL --exit-code 0 ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('4. Push Image to DockerHub') {
            steps {
                echo "Pushing Docker image to DockerHub..."
                withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                    sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('5. Delete Image Locally') {
            steps {
                echo "Cleaning up local Docker images..."
                sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('6. Update Image in Deployment File') {
            steps {
                echo "Updating deployment.yaml with image tag ${IMAGE_TAG}..."
                sh "sed -i 's|image: ${IMAGE_NAME}:.*|image: ${IMAGE_NAME}:${IMAGE_TAG}|' deployment.yaml"
            }
        }

        stage('7. Deploy to Kubernetes Cluster') {
            steps {
                echo "Applying manifests to Kubernetes cluster..."
                sh 'kubectl apply -f deployment.yaml'
                sh 'kubectl rollout status deployment/ivolveapp -n ivolve --timeout=120s'
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

### 3) Required Jenkins Configuration & Prerequisites
1. **DockerHub Credentials**:
   - Kind: `Username with password`
   - ID: `docker`
2. **Kubernetes Access**:
   - Kubeconfig / cluster context configured so Jenkins controller or agent can execute `kubectl apply` and `kubectl rollout status`.
3. **Build Dependencies**:
   - Node.js & npm installed in PATH (via NVM symlinks).
   - Trivy scanner installed (`trivy 0.76.1`).

---

### 4) Pipeline Execution Output
Execution logs from build #6 demonstrating successful completion of all stages:

```text
Started by user admin
Obtained Jenkinsfile from git https://github.com/marceleno1717/ivolveapp
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins in /var/jenkins_home/workspace/ivolveapp
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Declarative: Checkout SCM)
[Pipeline] checkout
using credential github
Cloning the remote Git repository
Cloning repository https://github.com/marceleno1717/ivolveapp
Checking out Revision b078d62d09ae7d72e6b26bde11555747925e6ba7 (refs/remotes/origin/main)
Commit message: "jen"
[Pipeline] }
[Pipeline] // stage
[Pipeline] withEnv
[Pipeline] {
[Pipeline] withEnv
[Pipeline] {
[Pipeline] stage
[Pipeline] { (1. Build App)
[Pipeline] echo
Installing application dependencies...
[Pipeline] sh
+ npm install

up to date, audited 85 packages in 2s

23 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (2. Build Docker Image)
[Pipeline] echo
Building Docker image marceleno17/ivolveapp:6...
[Pipeline] sh
+ docker build -t marceleno17/ivolveapp:6 .
Sending build context to Docker daemon  7.944MB
Step 1/7 : FROM node:18-alpine
Step 2/7 : WORKDIR /app
Step 3/7 : COPY package.json ./
Step 4/7 : RUN npm install
Step 5/7 : COPY . .
Step 6/7 : EXPOSE 3000
Step 7/7 : CMD ["node", "server.js"]
Successfully built fe653b006c33
Successfully tagged marceleno17/ivolveapp:6
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (3. Scan Docker Image)
[Pipeline] echo
Scanning Docker image with Trivy...
[Pipeline] sh
+ trivy image --severity HIGH,CRITICAL --exit-code 0 marceleno17/ivolveapp:6
2026-11-04T10:39:10Z	INFO	[vuln] Vulnerability scanning is enabled
2026-11-04T10:39:10Z	INFO	[secret] Secret scanning is enabled
2026-11-04T10:39:16Z	INFO	Detected OS	family="alpine" version="3.21.3"
2026-11-04T10:39:16Z	INFO	[alpine] Detecting vulnerabilities...	os_version="3.21" repository="3.21" pkg_num=17
2026-11-04T10:39:16Z	INFO	Number of language-specific files	num=1
2026-11-04T10:39:16Z	INFO	[node-pkg] Detecting vulnerabilities...

Report Summary

┌───────────────────────────────────────────────┬──────────┬─────────────────┬─────────┐
│                    Target                     │   Type   │ Vulnerabilities │ Secrets │
├───────────────────────────────────────────────┼──────────┼─────────────────┼─────────┤
│ marceleno17/ivolveapp:6 (alpine 3.21.3)       │  alpine  │       21        │    -    │
├───────────────────────────────────────────────┼──────────┼─────────────────┼─────────┤
│ app/node_modules/@types/node/package.json     │ node-pkg │        0        │    -    │
└───────────────────────────────────────────────┴──────────┴─────────────────┴─────────┘
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (4. Push Image to DockerHub)
[Pipeline] echo
Pushing Docker image to DockerHub...
[Pipeline] withCredentials
[Pipeline] {
[Pipeline] sh
+ echo ****
+ docker login -u marceleno17 --password-stdin
WARNING! Your password will be stored unencrypted in /var/jenkins_home/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credential-stores

Login Succeeded
+ docker push marceleno17/ivolveapp:6
The push refers to repository [docker.io/marceleno17/ivolveapp]
6: digest: sha256:fe653b006c333718b11ab8f323f3c8fa24d33d19ca8acc10c592ecba2f2eab78 size: 2057
[Pipeline] }
[Pipeline] // withCredentials
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (5. Delete Image Locally)
[Pipeline] echo
Cleaning up local Docker images...
[Pipeline] sh
+ docker rmi marceleno17/ivolveapp:6
Untagged: marceleno17/ivolveapp:6
Deleted: sha256:fe653b006c333718b11ab8f323f3c8fa24d33d19ca8acc10c592ecba2f2eab78
Deleted: sha256:4065ba0f21f477d05908294787c57f23f385f491fa4ccb64adf80d8a0ea0bb4c
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (6. Update Image in Deployment File)
[Pipeline] echo
Updating deployment.yaml with image tag 6...
[Pipeline] sh
+ sed -i s|image: marceleno17/ivolveapp:.*|image: marceleno17/ivolveapp:6| deployment.yaml
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (7. Deploy to Kubernetes Cluster)
[Pipeline] echo
Applying manifests to Kubernetes cluster...
[Pipeline] sh
+ kubectl apply -f deployment.yaml
deployment.apps/ivolveapp configured
[Pipeline] sh
+ kubectl rollout status deployment/ivolveapp -n ivolve --timeout=120s
Waiting for deployment "ivolveapp" rollout to finish: 0 out of 2 new replicas have been updated...
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
Pipeline succeeded! Application deployed with image marceleno17/ivolveapp:6.
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

### 5) Kubernetes Cluster Verification
Verify the pods and rollout in the `ivolve` namespace:

```bash
kubectl get pods,svc,deploy -n ivolve -o wide
```

```text
NAME                             READY   STATUS    RESTARTS       AGE     IP               NODE           NOMINATED NODE   READINESS GATES
pod/ivolveapp-685f6864d8-25xt2   1/1     Running   0              5m21s   10.244.205.204   minikube-m02   <none>           <none>
pod/ivolveapp-685f6864d8-z5ph2   1/1     Running   0              5m13s   10.244.205.205   minikube-m02   <none>           <none>
pod/ivolvedb-0                   1/1     Running   2 (34h ago)    2d14h   10.244.205.201   minikube-m02   <none>           <none>
pod/network                      1/1     Running   2 (125m ago)   2d14h   10.244.120.111   minikube       <none>           <none>

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE     SELECTOR
service/ivolvaappsvc   ClusterIP   10.98.243.216   <none>        3000/TCP   3d16h   app=ivolveapp
service/ivolvedbsvc    ClusterIP   None            <none>        3306/TCP   6d14h   app=ivolvedb

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES                        SELECTOR
deployment.apps/ivolveapp   2/2     2            2           3d16h   app          marceleno17/lab8-app:latest   app=ivolveapp
```
