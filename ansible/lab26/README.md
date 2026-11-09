# Lab 26: Structured Configuration Management with Ansible Roles

### 1) Lab Overview & Objectives
This lab restructures monolithic Ansible playbooks into modular, reusable, and decoupled **Ansible Roles** for installing and configuring **Docker**, **Kubectl**, and **Jenkins**:
1. **Infrastructure**:
   - **Control Node (`ansible-master`)**: AWS EC2 instance (`13.37.227.35` / `172.31.10.22`) executing the role-based playbook.
   - **Managed Node (`ansible-slave`)**: AWS EC2 instance (`52.47.108.81` / `172.31.0.30`) target of the roles.
2. **Key Objectives**:
   - Initialize separate role directory trees for `docker`, `kubectl`, and `jenkins` (`ansible-galaxy init`).
   - Migrate tasks into individual role task definitions (`tasks/main.yml`).
   - Create a clean top-level playbook (`tools_playbook.yaml`) executing the roles in sequence.
   - Run the role-based playbook against the inventory and verify status on the managed node.

---

### 2) Role Directory Structure
The modular layout in [`ansible/lab26/`](file:///home/marceleno/ivolve-labs/ansible/lab26/):

```text
lab26/
├── inventory
├── tools_playbook.yaml
├── docker/
│   └── tasks/
│       └── main.yml
├── kubectl/
│   └── tasks/
│       └── main.yml
└── jenkins/
    └── tasks/
        └── main.yml
```

Initialize each role using `ansible-galaxy`:

```bash
ansible-galaxy init docker && ansible-galaxy init kubectl && ansible-galaxy init jenkins
```

---

### 3) Role Task Definitions

#### 1. Docker Role (`ansible/lab26/docker/tasks/main.yml`)
```yaml
---
# tasks file for docker
- name: Install prerequisites
  apt:
    name:
      - ca-certificates
      - curl
      - gnupg
      - openjdk-21-jre
    state: present
    update_cache: true

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
```

#### 2. Kubectl Role (`ansible/lab26/kubectl/tasks/main.yml`)
```yaml
---
# tasks file for kubectl
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
```

#### 3. Jenkins Role (`ansible/lab26/jenkins/tasks/main.yml`)
```yaml
---
# tasks file for jenkins
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

### 4) Master Playbook Implementation (`tools_playbook.yaml`)
Located at [`ansible/lab26/tools_playbook.yaml`](file:///home/marceleno/ivolve-labs/ansible/lab26/tools_playbook.yaml):

```yaml
---
- name: Install Docker, Kubectl and Jenkins
  hosts: slaves
  become: true
  roles:
    - docker
    - kubectl
    - jenkins
```

---

### 5) Playbook Execution
Run the role-based playbook from `ansible-master`:

```bash
ansible-playbook -i inventory tools_playbook.yaml
```

```text
PLAY [Install Docker, Kubectl and Jenkins] *************************************

TASK [Gathering Facts] *********************************************************
ok: [172.31.0.30]

TASK [docker : Install prerequisites] ******************************************
ok: [172.31.0.30]

TASK [docker : Add Docker GPG key] *********************************************
ok: [172.31.0.30]

TASK [docker : Add Docker repository] ******************************************
ok: [172.31.0.30]

TASK [docker : Install Docker] *************************************************
ok: [172.31.0.30]

TASK [docker : Start and enable Docker] ****************************************
ok: [172.31.0.30]

TASK [docker : Add ubuntu user to Docker group] ********************************
ok: [172.31.0.30]

TASK [kubectl : Add Kubernetes GPG key] ****************************************
ok: [172.31.0.30]

TASK [kubectl : Add Kubernetes repository] *************************************
ok: [172.31.0.30]

TASK [kubectl : Install Kubectl] ***********************************************
ok: [172.31.0.30]

TASK [jenkins : Add Jenkins GPG key] *******************************************
ok: [172.31.0.30]

TASK [jenkins : Add Jenkins repository] ****************************************
ok: [172.31.0.30]

TASK [jenkins : Install Jenkins] ***********************************************
ok: [172.31.0.30]

TASK [jenkins : Start and enable Jenkins] **************************************
ok: [172.31.0.30]

PLAY RECAP *********************************************************************
172.31.0.30                : ok=14   changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

---

### 6) Verification on Managed Node (`ansible-slave`)

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

#### 2. Verify Services Running
```bash
systemctl is-active docker && systemctl is-active jenkins
```

```text
active
active
```

#### 3. Test Web Access & User Groups
```bash
curl -s -I http://localhost:8080/login | head -n 1 && groups ubuntu
```

```text
HTTP/1.1 200 OK
ubuntu : ubuntu adm cdrom sudo dip lxd docker
```
