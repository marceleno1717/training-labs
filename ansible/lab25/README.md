# Lab 25: Automated Web Server Configuration Using Ansible Playbooks

### 1) Lab Overview & Objectives
This lab automates the end-to-end installation and configuration of core DevOps tools (**Docker**, **Kubectl**, and **Jenkins**) on a remote managed node using Ansible playbooks:
1. **Infrastructure**:
   - **Control Node (`ansible-master`)**: AWS EC2 instance (`13.37.227.35` / `172.31.10.22`) running Ansible.
   - **Managed Node (`ansible-slave`)**: AWS EC2 instance (`52.47.108.81` / `172.31.0.30`) target of the playbook.
2. **Automated Tasks**:
   - Install prerequisite packages (`ca-certificates`, `curl`, `gnupg`, `openjdk-21-jre`).
   - Configure official repositories and GPG keys for Docker, Kubernetes, and Jenkins.
   - Install Docker CE, start and enable the Docker service, and grant the `ubuntu` user Docker group access.
   - Install `kubectl` client binary.
   - Install Jenkins, start and enable the systemd service.
   - Connect directly to `ansible-slave` to verify that all binaries and systemd services are active and responding.

---

### 2) Inventory & Playbook Setup on Control Node (`ansible-master`)

#### Inventory (`ansible/lab25/inventory`)
```ini
[slaves]
172.31.0.30 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa
```

#### Playbook (`ansible/lab25/tools_playbook.yaml`)
```yaml
---
- name: Install Docker, Kubectl and Jenkins
  hosts: slaves
  become: true

  tasks:
    - name: Install prerequisites
      apt:
        name:
          - ca-certificates
          - curl
          - gnupg
          - openjdk-21-jre
        state: present
        update_cache: true

    # Docker
    - name: Add Docker GPG key
      shell: |
        install -m 0755 -d /etc/apt/keyrings
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg |
          gpg --dearmor -o /etc/apt/keyrings/docker.gpg
        chmod a+r /etc/apt/keyrings/docker.gpg
      args:
        creates: /etc/apt/keyrings/docker.gpg

    - name: Add Docker repository
      apt_repository:
        repo: "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
        filename: docker
        state: present

    - name: Install Docker
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-buildx-plugin
          - docker-compose-plugin
        state: present
        update_cache: true

    - name: Start and enable Docker
      systemd:
        name: docker
        state: started
        enabled: true

    - name: Add ubuntu user to Docker group
      user:
        name: ubuntu
        groups: docker
        append: true

    # Kubectl
    - name: Add Kubernetes GPG key
      shell: |
        curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key |
          gpg --dearmor -o /etc/apt/keyrings/kubernetes.gpg
      args:
        creates: /etc/apt/keyrings/kubernetes.gpg

    - name: Add Kubernetes repository
      apt_repository:
        repo: "deb [signed-by=/etc/apt/keyrings/kubernetes.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /"
        filename: kubernetes
        state: present

    - name: Install Kubectl
      apt:
        name: kubectl
        state: present
        update_cache: true

    # Jenkins
    - name: Add Jenkins GPG key
      get_url:
        url: https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
        dest: /etc/apt/keyrings/jenkins-keyring.asc
        mode: '0644'

    - name: Add Jenkins repository
      apt_repository:
        repo: "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/"
        filename: jenkins
        state: present

    - name: Install Jenkins
      apt:
        name: jenkins
        state: present
        update_cache: true

    - name: Start and enable Jenkins
      systemd:
        name: jenkins
        state: started
        enabled: true
```

---

### 3) Playbook Execution
Run the playbook from `ansible-master`:

```bash
ansible-playbook -i inventory tools_playbook.yaml
```

```text
PLAY [Install Docker, Kubectl and Jenkins] *************************************

TASK [Gathering Facts] *********************************************************
ok: [172.31.0.30]

TASK [Install prerequisites] ***************************************************
ok: [172.31.0.30]

TASK [Add Docker GPG key] ******************************************************
ok: [172.31.0.30]

TASK [Add Docker repository] ***************************************************
ok: [172.31.0.30]

TASK [Install Docker] **********************************************************
ok: [172.31.0.30]

TASK [Start and enable Docker] *************************************************
ok: [172.31.0.30]

TASK [Add ubuntu user to Docker group] *****************************************
ok: [172.31.0.30]

TASK [Add Kubernetes GPG key] **************************************************
ok: [172.31.0.30]

TASK [Add Kubernetes repository] ***********************************************
ok: [172.31.0.30]

TASK [Install Kubectl] *********************************************************
ok: [172.31.0.30]

TASK [Add Jenkins GPG key] *****************************************************
ok: [172.31.0.30]

TASK [Add Jenkins repository] **************************************************
ok: [172.31.0.30]

TASK [Install Jenkins] *********************************************************
ok: [172.31.0.30]

TASK [Start and enable Jenkins] ************************************************
ok: [172.31.0.30]

PLAY RECAP *********************************************************************
172.31.0.30                : ok=14   changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

---

### 4) Verification on Managed Node (`ansible-slave`)

#### 1. Verify Installed Tool Versions
```bash
docker --version && kubectl version --client && jenkins --version
```

```text
Docker version 29.9.3, build 5457199
Client Version: v1.35.10
Kustomize Version: v5.7.1
2.580.2
```

#### 2. Verify Service Active Status
```bash
systemctl is-active docker && systemctl is-active jenkins
```

```text
active
active
```

#### 3. Inspect Docker Service Status
```bash
systemctl status docker --no-pager | head -n 12
```

```text
● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-11-09 20:15:33 UTC; 9min ago
 Invocation: a5ccde2d79fa4c2e958ef59b217d1f3e
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 14270 (dockerd)
      Tasks: 9
     Memory: 29.2M (peak: 29.6M)
        CPU: 605ms
     CGroup: /system.slice/docker.service
             └─14270 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

#### 4. Inspect Jenkins Service Status
```bash
systemctl status jenkins --no-pager | head -n 12
```

```text
● jenkins.service - Jenkins Continuous Integration Server
     Loaded: loaded (/usr/lib/systemd/system/jenkins.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-11-09 20:16:17 UTC; 8min ago
 Invocation: 7f3958dc2e3a44b5a6de67cbb01b76b0
   Main PID: 15546 (java)
      Tasks: 43 (limit: 1741)
     Memory: 531.9M (peak: 559.4M)
        CPU: 24.686s
     CGroup: /system.slice/jenkins.service
             └─15546 /usr/bin/java -Djava.awt.headless=true -jar /usr/share/java/jenkins.war --webroot=/var/cache/jenkins/war --httpPort=8080
```

#### 5. Verify Jenkins Web Interface & Group Membership
```bash
curl -s -I http://localhost:8080/login | head -n 1 && groups ubuntu
```

```text
HTTP/1.1 200 OK
ubuntu : ubuntu adm cdrom sudo dip lxd docker
```
