# Lab 27: Securing Sensitive Data with Ansible Vault

### 1) Lab Overview & Objectives
This lab secures sensitive credentials (database passwords) using **Ansible Vault** and automates the configuration of a **MySQL** database server using a dedicated Ansible role:
1. **Infrastructure**:
   - **Control Node (`ansible-master`)**: AWS EC2 instance running Ansible Core.
   - **Managed Node (`ansible-slave`)**: AWS EC2 instance (`172.31.0.30`).
2. **Key Objectives**:
   - Create a modular MySQL role (`mysql/tasks/main.yml`) to:
     - Install `mysql-server` and `python3-mysqldb` (required by Ansible MySQL modules).
     - Start and enable the MySQL service.
     - Create database `ivolve`.
     - Create user `ivolveuser` with password authenticated via `caching_sha2_password`.
     - Grant all privileges on `ivolve.*` to `ivolveuser`.
   - Encrypt sensitive variables (such as `database_password`) using `ansible-vault` inside `secrets.yaml`.
   - Execute the playbook using `--ask-vault-pass`.
   - Verify MySQL connectivity and privileges directly on the managed node using `ivolveuser`.

---

### 2) SSH into the Control Node
Connect to the control node:

```bash
ssh -i ~/.ssh/k3s.pem ubuntu@35.180.121.225
```

---

### 3) MySQL Role Implementation
The role layout is located in [`ansible/lab27/mysql/`](file:///home/marceleno/ivolve-labs/ansible/lab27/mysql):

#### Role Tasks (`ansible/lab27/mysql/tasks/main.yml`)
```yaml
---
# tasks file for mysql
- name: Install MySQL
  apt:
    name:
      - mysql-server
      - python3-mysqldb
    state: present

- name: Start MySQL service
  service:
    name: mysql
    state: started

- name: Create MySQL user
  mysql_user:
    name: ivolveuser
    plugin_auth_string: "{{ database_password }}"
    plugin: caching_sha2_password
    host: "localhost"
    state: present
    login_unix_socket: /run/mysqld/mysqld.sock

- name: Create MySQL database 'ivolve'
  mysql_db:
    name: ivolve
    state: present

- name: Grant privileges to 'ivolveuser' user for 'ivolve' database
  mysql_user:
    name: ivolveuser
    priv: "ivolve.*:ALL"
    host: "localhost"
    state: present
```

---

### 4) Securing Sensitive Data with Ansible Vault
Create and encrypt the secret variables file:

```bash
ansible-vault create secrets.yaml
```

Unencrypted content structure:
```yaml
database_password: "YOUR_SECURE_PASSWORD"
```

To view or edit the encrypted vault file:

```bash
ansible-vault view secrets.yaml
```

```bash
ansible-vault edit secrets.yaml
```

Encrypted file structure ([`ansible/lab27/secrets.yaml`](file:///home/marceleno/ivolve-labs/ansible/lab27/secrets.yaml)):
```text
$ANSIBLE_VAULT;1.1;AES256
31326463386462653736633639373963326462393365363937633435386562333561336637346630
...
```

---

### 5) Master Playbook Implementation
Located at [`ansible/lab27/mysql-playbook.yaml`](file:///home/marceleno/ivolve-labs/ansible/lab27/mysql-playbook.yaml):

```yaml
- name: configure MySQL DB
  hosts: slaves
  become: true
  vars_files:
    - secrets.yaml
  roles:
    - mysql
```

Inventory configuration ([`ansible/lab27/inventory`](file:///home/marceleno/ivolve-labs/ansible/lab27/inventory)):
```ini
[slaves]
172.31.0.30 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa
```

---

### 6) Playbook Execution
Execute the playbook providing the vault password prompt:

```bash
ansible-playbook mysql-playbook.yaml -i inventory --ask-vault-pass
```

```text
Vault password: 

PLAY [configure MySQL DB] ******************************************************

TASK [Gathering Facts] *********************************************************
ok: [172.31.0.30]

TASK [mysql : Install MySQL] ***************************************************
ok: [172.31.0.30]

TASK [mysql : Start MySQL service] *********************************************
ok: [172.31.0.30]

TASK [mysql : Create MySQL user] ***********************************************
ok: [172.31.0.30]

TASK [mysql : Create MySQL database 'ivolve'] **********************************
ok: [172.31.0.30]

TASK [mysql : Grant privileges to 'ivolveuser' user for 'ivolve' database] *****
ok: [172.31.0.30]

PLAY RECAP *********************************************************************
172.31.0.30                : ok=6    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

---

### 7) Verification on Managed Node (`ansible-slave`)
Connect to the managed node:

```bash
ssh -i ~/.ssh/k3s.pem ubuntu@15.237.179.70
```

#### 1. Validate Database Access with Created User
Connect to MySQL using `ivolveuser` and list accessible databases:

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

#### 2. Validate User Privileges & Database Operations
Verify that `ivolveuser` can create tables, insert records, and query inside the `ivolve` database:

```bash
mysql -u ivolveuser -p -e 'USE ivolve; CREATE TABLE test_table (id INT PRIMARY KEY, name VARCHAR(50)); INSERT INTO test_table VALUES (1, "test"); SELECT * FROM test_table; DROP TABLE test_table;'
```

```text
Enter password: 
id	name
1	test
```
