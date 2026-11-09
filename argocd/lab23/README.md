# Lab 23: Implementing a GitOps Workflow with Jenkins and Argo CD

### 1) Lab Overview & Objectives
This lab establishes an automated GitOps delivery model connecting Jenkins CI with Argo CD GitOps CD across an AWS EC2 Kubernetes cluster (`kadm-master` and `kadm-slave` in region `eu-west-3`):
1. **Cluster Infrastructure**:
   - Master Node: `kadm-master` (`51.44.168.39` / `172.31.12.149`)
   - Worker Node: `kadm-slave` (`51.44.181.86` / `172.31.8.8`)
2. **Argo CD Deployment & Access**:
   - Install Argo CD in namespace `argocd`.
   - Access Argo CD UI via port-forwarding on port `8080`.
   - Log in with credentials retrieved from secret `argocd-initial-admin-secret`.
3. **Argo CD Application Setup**:
   - Create an Argo CD Application (`ivolveapp`) connected to `https://github.com/marceleno1717/ivolveapp.git`.
   - Monitor manifest path `yamls/` on branch `main` targeting namespace `ivolve`.
   - Enable automated synchronization with self-heal and pruning enabled.
4. **Jenkins CI & GitOps Integration (`Jenkinsfile3` & `pushDeployment`)**:
   - Build application dependencies via `buildJS()`.
   - Build, scan with Trivy, and push Docker image to Docker Hub via `buildScanPushImage(...)`.
   - Instead of deploying directly via `kubectl`, use `pushDeployment(...)` to dynamically update `yamls/deployment.yaml` with the new image tag and push the commit back to GitHub.
5. **GitOps Synchronization**:
   - Argo CD detects the updated image tag in Git and automatically synchronizes the deployment onto the Kubernetes cluster.

---

### 2) Cluster Verification (AWS EC2)
Verify cluster connectivity and node readiness on `kadm-master`:

```bash
ssh -i ~/.ssh/k3s.pem ubuntu@51.44.168.39 "kubectl get nodes -o wide"
```

```text
NAME               STATUS   ROLES           AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
ip-172-31-12-149   Ready    control-plane   15d   v1.35.9   172.31.12.149   <none>        Ubuntu 26.04.1 LTS   7.0.0-1014-aws   containerd://2.2.2
ip-172-31-8-8      Ready    <none>          15d   v1.35.9   172.31.8.8      <none>        Ubuntu 26.04.1 LTS   7.0.0-1016-aws   containerd://2.2.2
```

---

### 3) Argo CD Installation & Access

#### Install Argo CD
Create the dedicated `argocd` namespace and deploy the official Argo CD manifests:

```bash
kubectl create namespace argocd
```

```bash
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Verify all Argo CD components are running:

```bash
kubectl get pods -n argocd
```

```text
NAME                                                READY   STATUS    RESTARTS      AGE
argocd-application-controller-0                     1/1     Running   1 (18m ago)   4d5h
argocd-applicationset-controller-5b46dffcbc-gxppp   1/1     Running   1 (18m ago)   4d5h
argocd-dex-server-57bccbfbb6-pxnh8                  1/1     Running   3 (18m ago)   4d5h
argocd-notifications-controller-5fb6c97685-phjpw    1/1     Running   1 (18m ago)   4d5h
argocd-redis-6b84c6f49b-pxcn2                       1/1     Running   1 (18m ago)   4d5h
argocd-repo-server-755f565f4f-t2b8m                 1/1     Running   1 (18m ago)   4d5h
argocd-server-9d47c9989-gxflb                       1/1     Running   1 (18m ago)   4d5h
```

#### Access Argo CD UI & Retrieve Initial Admin Password
Expose the Argo CD UI using port-forwarding:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 --address 0.0.0.0 &
```

Retrieve the initial password for user `admin`:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

---

### 4) Argo CD Application Configuration (`ivolveapp`)
The Argo CD application connects to the GitHub repository and monitors the `yamls` manifest directory with automated synchronization:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ivolveapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/marceleno1717/ivolveapp.git
    targetRevision: main
    path: yamls
  destination:
    server: https://kubernetes.default.svc
    namespace: ivolve
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply the application manifest:

```bash
kubectl apply -f ivolveapp-application.yaml
```

Verify application status in Argo CD:

```bash
kubectl get application ivolveapp -n argocd -o wide
```

```text
NAME        SYNC STATUS   HEALTH STATUS   REVISION                                   PROJECT
ivolveapp   Synced        Healthy         62930a2b38ea6831f2e1dcfc19310a6903af96d3   default
```

---

### 5) Shared Library GitOps Extension (`pushDeployment`)
In `jenkins-shared-lib`, the deployment stage was modified to push updated manifests directly to GitHub rather than applying manifests with `kubectl`.

#### `vars/pushDeployment.groovy`
```groovy
def call(String imageName, String newTag, String deploymentFile, String githubCreds, String email, String repoUrl){
    def repo = repoUrl.replaceFirst('^https://', '')
    
    withCredentials([
        usernamePassword(credentialsId: githubCreds, usernameVariable: 'GITHUB_USERNAME', passwordVariable: 'GITHUB_PASSWORD')
    ]){
        sh """
            sed -i 's|image: ${imageName}:.*|image: ${imageName}:${newTag}|' ${deploymentFile}
            echo "pushing to ${repo}..."

            git config user.email "${email}"
            git config user.name "${GITHUB_USERNAME}"

            git add ${deploymentFile}
            git commit -m "Update deployment image to ${imageName}:${newTag}!"

            git push https://${GITHUB_USERNAME}:${GITHUB_PASSWORD}@${repo} HEAD:main
        """
    }
}
```

---

### 6) Jenkins Pipeline Implementation (`Jenkinsfile3`)
Located at [`argocd/lab23/iVolveApp/Jenkinsfile3`](file:///home/marceleno/ivolve-labs/argocd/lab23/iVolveApp/Jenkinsfile3):

```groovy
@Library('shared-lib') _

pipeline {
    agent any

    environment {
        IMAGE_NAME = 'marceleno17/ivolveapp'
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKERHUB_CREDENTIALS_ID = 'docker'
        DEPLOYMENT_FILE = 'yamls/deployment.yaml'
        GITHUB_CREDENTIALS_ID = 'github'
        EMAIL = "maro.keko17@gmail.com"
        REPOURL = "https://github.com/marceleno1717/ivolveapp"
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

        stage('3. Push to repo') {
            steps {
                pushDeployment("${IMAGE_NAME}", "${IMAGE_TAG}", "${DEPLOYMENT_FILE}", "${GITHUB_CREDENTIALS_ID}", "${EMAIL}", "${REPOURL}")
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

### 7) Pipeline Execution Output
Live console output from Jenkins build `#8` demonstrating image build, vulnerability scan, DockerHub push, and automated commit back to GitHub:

```text
Started by user admin
Obtained Jenkinsfile3 from git https://github.com/marceleno1717/ivolveapp/
Loading library shared-lib@main
Found match: refs/heads/main revision 96272becaccb436f794732423bccdb0ea5524348
Cloning repository https://github.com/marceleno1717/jenkins-shared-lib
Checking out Revision 96272becaccb436f794732423bccdb0ea5524348 (main)
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins in /var/jenkins_home/workspace/lab23
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Declarative: Checkout SCM)
[Pipeline] checkout
Cloning repository https://github.com/marceleno1717/ivolveapp/
Checking out Revision 9965888cfaa567642db39eae69707d3dc2005de7 (refs/remotes/origin/main)
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

up to date, audited 85 packages in 2s

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
+ docker build -t marceleno17/ivolveapp:8 .
Successfully built 40b1ea38fd7a
Successfully tagged marceleno17/ivolveapp:8
[Pipeline] sh
+ trivy image --severity HIGH,CRITICAL --exit-code 0 marceleno17/ivolveapp:8
2026-11-05T18:09:40Z	INFO	Table result includes only package filenames. Use '--format json' option to get the full path to the package file.

Report Summary

┌───────────────────────────────────────────────┬──────────┬─────────────────┬─────────┐
│                    Target                     │   Type   │ Vulnerabilities │ Secrets │
├───────────────────────────────────────────────┼──────────┼─────────────────┼─────────┤
│ marceleno17/ivolveapp:8 (alpine 3.21.3)       │  alpine  │       21        │    -    │
├───────────────────────────────────────────────┼──────────┼─────────────────┼─────────┤
│ app/node_modules/@types/node/package.json     │ node-pkg │        0        │    -    │
└───────────────────────────────────────────────┴──────────┴─────────────────┴─────────┘
[Pipeline] withCredentials
[Pipeline] {
[Pipeline] sh
+ echo ****
+ docker login -u marceleno17 --password-stdin
Login Succeeded
+ docker push marceleno17/ivolveapp:8
The push refers to repository [docker.io/marceleno17/ivolveapp]
8: digest: sha256:b2bfd74aa0b03a6a78104bf825b9d7283eb0f194e0dc2db0d1bce31d93aef8b5 size: 2057
[Pipeline] }
[Pipeline] // withCredentials
[Pipeline] sh
+ docker rmi marceleno17/ivolveapp:8
Untagged: marceleno17/ivolveapp:8
Deleted: sha256:b2bfd74aa0b03a6a78104bf825b9d7283eb0f194e0dc2db0d1bce31d93aef8b5
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (3. Push to repo)
[Pipeline] withCredentials
[Pipeline] {
[Pipeline] sh
+ sed -i s|image: marceleno17/ivolveapp:.*|image: marceleno17/ivolveapp:8| yamls/deployment.yaml
+ echo pushing to github.com/marceleno1717/ivolveapp...
pushing to github.com/marceleno1717/ivolveapp...
+ git config user.email maro.keko17@gmail.com
+ git config user.name marceleno1717
+ git add yamls/deployment.yaml
+ git commit -m Update deployment image to marceleno17/ivolveapp:8!
[detached HEAD 62930a2] Update deployment image to marceleno17/ivolveapp:8!
 1 file changed, 1 insertion(+), 1 deletion(-)
+ git push https://marceleno1717:****@github.com/marceleno1717/ivolveapp HEAD:main
To https://github.com/marceleno1717/ivolveapp
   9965888..62930a2  HEAD -> main
[Pipeline] }
[Pipeline] // withCredentials
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
Pipeline succeeded! Application deployed with image marceleno17/ivolveapp:8.
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

### 8) GitOps Synchronization & Verification on AWS Cluster
Once the commit `62930a2` was pushed to GitHub, Argo CD's automated sync policy detected the change and reconciled the cluster state:

```bash
kubectl get pods,svc,deploy -n ivolve -o wide
```

```text
NAME                             READY   STATUS    RESTARTS      AGE    IP            NODE            NOMINATED NODE   READINESS GATES
pod/ivolveapp-78698c76b9-clx4w   1/1     Running   1 (18m ago)   4d2h   10.244.1.43   ip-172-31-8-8   <none>           <none>
pod/ivolveapp-78698c76b9-r7lwp   1/1     Running   1 (18m ago)   4d2h   10.244.1.37   ip-172-31-8-8   <none>           <none>
pod/ivolvedb-0                   1/1     Running   2 (18m ago)   4d6h   10.244.1.39   ip-172-31-8-8   <none>           <none>

NAME                   TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE    SELECTOR
service/ivolvaappsvc   ClusterIP   10.99.7.10   <none>        3000/TCP   4d6h   app=ivolveapp
service/ivolvedbsvc    ClusterIP   None         <none>        3306/TCP   4d6h   app=ivolvedb

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS   IMAGES                    SELECTOR
deployment.apps/ivolveapp   2/2     2            2           4d6h   app          marceleno17/ivolveapp:8   app=ivolveapp
```

The deployment reflects image `marceleno17/ivolveapp:8` running seamlessly on the worker node (`ip-172-31-8-8`), completing the automated GitOps continuous deployment loop.
