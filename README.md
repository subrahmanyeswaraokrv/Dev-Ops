# Linux User Management using Ansible

## Overview
This document describes how to **create a Linux user (`venkata`) and grant sudo access**
on multiple servers using **Ansible**, executed from the **Ansible control server**.

The process is:
- Secure (hashed passwords)
- Idempotent (safe to re-run)
- Scalable (works for 1 or 100+ servers)

---

## Control Server Details
- Hostname: ansible
- Executed as: root
- Ansible Version: core 2.16.x
- Python Version: 3.10+

---

## Directory Structure

/home/venkata/ansible/
├── inventory.ini
└── add_user_venkata.yml

yaml
Copy code

---

## Inventory File

### Path
/home/venkata/ansible/inventory.ini

csharp
Copy code

### Content
```ini
[mongo_servers]
10.13.61.165
10.13.61.185
10.13.61.135
10.13.61.188
10.13.61.138
10.13.61.109
10.13.61.110
10.13.61.111
10.13.61.72
10.13.61.73
10.13.61.74
10.13.61.39
10.13.61.68
10.13.61.38
10.13.61.180
10.13.61.184
Connectivity Test (Mandatory)
Before running any playbook, verify SSH connectivity:

bash
Copy code
ansible -i inventory.ini mongo_servers -m ping -u touadmin --ask-pass
Expected output:

ini
Copy code
SUCCESS => pong
Remove or comment any unreachable hosts.

Password Hash Generation
Linux stores passwords as SHA-512 hashes.
Ansible requires hashed passwords (not plain text).

Command Used
bash
Copy code
python3 - <<EOF
import crypt
print(crypt.crypt("xuv7ooax5", crypt.mksalt(crypt.METHOD_SHA512)))
EOF
Example Output
shell
Copy code
$6$Ow9e8fkQEIDzbJ6h$pTdb5NG3clcMGP/4v0gJB.YSeWRqU4z3/dsjDsKG4j9p9eYlNhKuRGz21f5pK0XgPpWGs3gxpnllF995/dTgj1
⚠️ Never store plain-text passwords in playbooks.

Ansible Playbook
Path
swift
Copy code
/home/venkata/ansible/add_user_venkata.yml
Playbook Content
yaml
Copy code
---
- name: Create Linux user venkata and grant sudo access
  hosts: mongo_servers
  become: yes

  vars:
    username: venkata
    user_password: "$6$Ow9e8fkQEIDzbJ6h$pTdb5NG3clcMGP/4v0gJB.YSeWRqU4z3/dsjDsKG4j9p9eYlNhKuRGz21f5pK0XgPpWGs3gxpnllF995/dTgj1"

  tasks:
    - name: Ensure venkata user exists
      ansible.builtin.user:
        name: "{{ username }}"
        password: "{{ user_password }}"
        shell: /bin/bash
        create_home: yes
        state: present

    - name: Add venkata to sudo group
      ansible.builtin.user:
        name: "{{ username }}"
        groups: sudo
        append: yes
Execute the Playbook
First-Time Execution
bash
Copy code
ansible-playbook -i inventory.ini add_user_venkata.yml \
  -u touadmin --ask-pass --ask-become-pass
Prompts:

SSH password (touadmin)

sudo password (touadmin)

Expected Output
sql
Copy code
TASK [Ensure venkata user exists] → changed
PLAY RECAP → failed=0 unreachable=0
changed=1 means user was created and added to sudo group

Re-running the playbook will show changed=0

Verification
Check user exists
bash
Copy code
ansible -i inventory.ini mongo_servers -u touadmin -b \
  -m command -a "id venkata"
Check sudo group
bash
Copy code
ansible -i inventory.ini mongo_servers -u touadmin -b \
  -m command -a "groups venkata"
Expected:

bash
Copy code
venkata : venkata sudo
Manual Login Test
bash
Copy code
ssh venkata@10.13.61.165
Optional Enhancements
Force password change on first login
yaml
Copy code
- name: Force password change on first login
  command: chage -d 0 venkata
Passwordless sudo (Optional)
yaml
Copy code
- name: Allow venkata passwordless sudo
  copy:
    dest: /etc/sudoers.d/venkata
    content: "venkata ALL=(ALL) NOPASSWD:ALL\n"
    mode: '0440'
SSH Key-Based Authentication (Recommended)
bash
Copy code
ssh-keygen
ansible mongo_servers -i inventory.ini -u touadmin --ask-pass \
  -m authorized_key \
  -a "user=touadmin state=present key='{{ lookup('file', '/root/.ssh/id_rsa.pub') }}'"
Then run playbooks without passwords:

bash
Copy code
ansible-playbook -i inventory.ini add_user_venkata.yml -u touadmin --become
Rollback (Remove User)
yaml
Copy code
- name: Remove venkata user
  user:
    name: venkata
    state: absent
    remove: yes
Summary
Item	Status
User creation	Completed
Sudo access	Granted
Security	SHA-512 hashed password
Automation	Ansible
Re-run safe	Yes (Idempotent)
