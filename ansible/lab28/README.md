# Lab 28: Automated Host Discovery with Ansible Dynamic Inventory

### 1) Lab Overview & Objectives
This lab automates cloud infrastructure management by replacing static inventory files with the official **AWS EC2 Dynamic Inventory Plugin** (`amazon.aws.aws_ec2`):
1. **Infrastructure & Tagging**:
   - Cloud Provider: **Amazon Web Services (AWS)**, region `eu-west-3` (Paris).
   - Target Managed EC2 tagged with `type: ivolve` (`ansible-slave` at `15.237.179.70`).
2. **Key Objectives**:
   - Configure Ansible with the `amazon.aws.aws_ec2` inventory plugin.
   - Define dynamic inventory query filters matching running instances with tag `type: ivolve`.
   - List and graph the dynamically discovered targets using `ansible-inventory`.
   - Execute an Ansible playbook against the dynamically discovered host group (`aws_ec2`).
   - Connect directly to the managed instance to verify the applied configuration.

---

### 2) SSH into the Control Node
Connect to the control node:

```bash
ssh -i ~/.ssh/k3s.pem ubuntu@35.180.121.225
```

---

### 3) Ansible Dynamic Inventory Configuration

#### Ansible Configuration (`ansible/lab28/ansible.cfg`)
Configures Ansible to enable the `aws_ec2` plugin, set the default private key, and reference the dynamic inventory file:

```ini
[defaults]
remote_user = ubuntu
private_key_file = ~/.ssh/id_rsa
host_key_checking = False
inventory = inventory.aws_ec2.yaml

[inventory]
enable_plugins = aws_ec2
```

#### Dynamic Inventory Definition (`ansible/lab28/inventory.aws_ec2.yaml`)
Queries AWS EC2 in `eu-west-3` filtering for running instances matching tag `type: ivolve`:

```yaml
plugin: aws_ec2
regions:
  - eu-west-3
filters:
  instance-state-name: running
  tag:type: ivolve
```

---

### 4) Host Discovery & Inventory Verification
Verify dynamic discovery of target EC2 instances without static IP management:

#### 1. Graph Discovered Hosts
```bash
ansible-inventory --graph
```

```text
@all:
  |--@ungrouped:
  |--@aws_ec2:
  |  |--ec2-15-237-179-70.eu-west-3.compute.amazonaws.com
```

#### 2. Inspect Discovered Host Metadata
```bash
ansible-inventory --list
```

```json
{
    "_meta": {
        "hostvars": {
            "ec2-15-237-179-70.eu-west-3.compute.amazonaws.com": {
                "instance_id": "i-03f43cab354ad77b8",
                "instance_type": "t3.small",
                "private_ip_address": "172.31.0.30",
                "public_ip_address": "15.237.179.70",
                "tags": {
                    "Name": "ansible-slave",
                    "type": "ivolve"
                }
            }
        }
    },
    "all": {
        "children": [
            "ungrouped",
            "aws_ec2"
        ]
    },
    "aws_ec2": {
        "hosts": [
            "ec2-15-237-179-70.eu-west-3.compute.amazonaws.com"
        ]
    }
}
```

#### 3. Test Connectivity via Dynamic Host Group
```bash
ansible aws_ec2 -m ping
```

```text
ec2-15-237-179-70.eu-west-3.compute.amazonaws.com | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.14"
    },
    "changed": false,
    "ping": "pong"
}
```

---

### 5) Run Playbook on Dynamically Discovered EC2
Execute the playbook targeting the `aws_ec2` dynamic group ([`ansible/lab28/mysql-playbook.yaml`](file:///home/marceleno/ivolve-labs/ansible/lab28/mysql-playbook.yaml)):

```yaml
- name: configure MySQL DB
  hosts: aws_ec2
  become: true
  vars_files:
    - secrets.yaml
  roles:
    - mysql
```

Run the playbook:

```bash
ansible-playbook mysql-playbook.yaml --ask-vault-pass
```

```text
Vault password: 

PLAY [configure MySQL DB] ******************************************************

TASK [Gathering Facts] *********************************************************
ok: [ec2-15-237-179-70.eu-west-3.compute.amazonaws.com]

TASK [mysql : Install MySQL] ***************************************************
ok: [ec2-15-237-179-70.eu-west-3.compute.amazonaws.com]

TASK [mysql : Start MySQL service] *********************************************
ok: [ec2-15-237-179-70.eu-west-3.compute.amazonaws.com]

TASK [mysql : Create MySQL user] ***********************************************
ok: [ec2-15-237-179-70.eu-west-3.compute.amazonaws.com]

TASK [mysql : Create MySQL database 'ivolve'] **********************************
ok: [ec2-15-237-179-70.eu-west-3.compute.amazonaws.com]

TASK [mysql : Grant privileges to 'ivolveuser' user for 'ivolve' database] *****
ok: [ec2-15-237-179-70.eu-west-3.compute.amazonaws.com]

PLAY RECAP *********************************************************************
ec2-15-237-179-70.eu-west-3.compute.amazonaws.com : ok=6    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

---

### 6) Verification on Managed Node (`ansible-slave`)
Connect to the discovered instance:

```bash
ssh -i ~/.ssh/k3s.pem ubuntu@15.237.179.70
```

#### 1. Verify Database Service
```bash
systemctl is-active mysql
```

```text
active
```

#### 2. Verify Database and User Privileges
```bash
mysql -u ivolveuser -p -e 'SHOW DATABASES;'
```

```text
Enter password: 
Database
information_schema
ivolve
performance_schema
```
