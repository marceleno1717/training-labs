# Lab 24: Initial Ansible Configuration and Ad-Hoc Execution

### 1) Lab Overview & Objectives
This lab covers the setup and initial ad-hoc administration of an Ansible control node connected to a managed worker node in AWS EC2 (`eu-west-3` region):
1. **Control Node (`ansible-master`)**:
   - Public IP: `13.37.227.35` | Private IP: `172.31.10.22`
2. **Managed Node (`ansible-slave`)**:
   - Public IP: `52.47.108.81` | Private IP: `172.31.0.30`
3. **Tasks**:
   - Install and configure Ansible on the control node.
   - Generate an SSH keypair on the control node.
   - Distribute the public key to the managed node (`ssh-copy-id`).
   - Create an inventory file referencing the managed node.
   - Execute ad-hoc Ansible commands to verify connectivity (`ping`) and install/verify Docker on the managed node.

---

### 2) Install and Configure Ansible on Control Node (`ansible-master`)
Install `pipx` and install Ansible on the control node:

```bash
sudo apt update && sudo apt install -y pipx
```

```bash
pipx install --include-deps ansible
```

Ensure `~/.local/bin` is in the user's `PATH`:

```bash
pipx ensurepath
```

Verify Ansible version on the control node:

```bash
ansible --version
```

```text
ansible [core 2.21.6]
  config file = None
  configured module search path = ['/home/ubuntu/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /home/ubuntu/.local/share/pipx/venvs/ansible/lib/python3.14/site-packages/ansible
  ansible collection location = /home/ubuntu/.ansible/collections:/usr/share/ansible/collections
  executable location = /home/ubuntu/.local/bin/ansible
  python version = 3.14.4 (main, Aug 20 2026, 10:41:58) [GCC 15.2.0] (/home/ubuntu/.local/share/pipx/venvs/ansible/bin/python)
  jinja version = 3.1.6
  pyyaml version = 6.0.3 (with libyaml v0.2.5)
```

---

### 3) SSH Key Generation & Public Key Distribution
Generate a 4096-bit RSA keypair on the control node:

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

Transfer the public key to the managed node (`ansible-slave` at `172.31.0.30`):

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@172.31.0.30
```

Verify passwordless SSH connection:

```bash
ssh -i ~/.ssh/id_rsa ubuntu@172.31.0.30 "hostname -I; whoami"
```

```text
172.31.0.30
ubuntu
```

---

### 4) Inventory File Configuration
Create an inventory file [`ansible/lab24/inventory`](file:///home/marceleno/ivolve-labs/ansible/lab24/inventory) to define the managed node group:

```ini
[slaves]
172.31.0.30 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa
```

Verify connectivity using the `ping` module:

```bash
ansible -i inventory slaves -m ping
```

```text
172.31.0.30 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.14"
    },
    "changed": false,
    "ping": "pong"
}
```

---

### 5) Ad-Hoc Execution: Install & Verify Docker
Execute ad-hoc commands with root privileges (`become`) on the managed node:

#### 1. Install Docker Prerequisites
```bash
ansible -i inventory slaves -m apt -a "name=ca-certificates,curl,gnupg state=present update_cache=yes" -b
```

#### 2. Install Docker Packages
```bash
ansible -i inventory slaves -m apt -a "name=docker.io state=present update_cache=yes" -b
```

#### 3. Ensure Docker Service is Started & Enabled
```bash
ansible -i inventory slaves -m systemd -a "name=docker state=started enabled=yes" -b
```

#### 4. Verify Docker Version via Ad-Hoc Command
```bash
ansible -i inventory slaves -m command -a "docker --version" -b
```

```text
172.31.0.30 | CHANGED | rc=0 >>
Docker version 29.9.3, build 5457199
```

#### 5. Verify Docker Service Status via Ad-Hoc Command
```bash
ansible -i inventory slaves -m command -a "systemctl is-active docker" -b
```

```text
172.31.0.30 | CHANGED | rc=0 >>
active
```
