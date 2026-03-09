# AWX Brick 05 — Galaxy Roles & Collections

> **Goal:** Stop writing everything from scratch. Use `geerlingguy.nginx` from
> Ansible Galaxy to replace the custom nginx role, and use the `amazon.aws`
> collection to provision EC2 instances directly from a playbook — no AWS
> console required.

---

## What Changes from Brick 04

| Area | Brick 04 | Brick 05 |
|---|---|---|
| nginx role | Custom — written from scratch | `geerlingguy.nginx` from Galaxy |
| EC2 provisioning | Manual via AWS console | `amazon.aws.ec2_instance` module |
| Role source | Local `roles/` directory | Galaxy + local |
| Collection source | `collections/requirements.yml` | Same — extended |
| Playbook structure | Single `site.yml` | `provision_ec2.yml` + `configure.yml` imported by `site.yml` |
| IAM role permissions | EC2 read only | EC2 full access (write needed to create instances) |

---

## Repository Structure

```
awx-brick-05-collections/
├── ansible.cfg
├── site.yml                      # imports both playbooks in sequence
├── provision_ec2.yml             # phase 1 — create EC2 via amazon.aws
├── configure.yml                 # phase 2 — system_baseline + geerlingguy.nginx
├── collections/
│   └── requirements.yml          # amazon.aws + community.general
├── roles/
│   ├── requirements.yml          # geerlingguy.nginx from Galaxy
│   └── system_baseline/          # carried over from Brick 4
│       ├── tasks/main.yml
│       ├── handlers/main.yml
│       ├── defaults/main.yml
│       └── meta/main.yml
├── inventories/
│   └── aws/
│       └── aws_ec2.yml
└── group_vars/
    └── aws_ec2.yml
```

---

## Part 1 — Create the Repo

```bash
cd ~
git clone https://github.com/USERNAME/awx-brick-05-collections.git
cd awx-brick-05-collections

mkdir -p roles/system_baseline/{tasks,handlers,defaults,meta}
mkdir -p group_vars inventories/aws collections

# Copy system_baseline from Brick 4
cp -r ~/awx-brick-04-roles/roles/system_baseline roles/
```

---

## Part 2 — ansible.cfg

```ini
[defaults]
roles_path       = ./roles:~/.ansible/roles
collections_path = ./collections
```

---

## Part 3 — Requirements Files

### collections/requirements.yml

AWX installs these collections before running any job.

```yaml
---
collections:
  - name: amazon.aws
    version: ">=6.0.0"
  - name: community.general
    version: ">=7.0.0"
```

### roles/requirements.yml

AWX installs Galaxy roles listed here automatically during project sync.

```yaml
---
roles:
  - name: geerlingguy.nginx
    version: "3.2.0"
```

> `geerlingguy.nginx` is one of the most downloaded Galaxy roles. It supports
> Amazon Linux and is actively maintained — a standard choice in production.

---

## Part 4 — Inventory and group_vars

### inventories/aws/aws_ec2.yml

```yaml
---
plugin: amazon.aws.aws_ec2

regions:
  - YOUR_REGION

assume_role_arn: "arn:aws:iam::ACCOUNT_ID:role/awx-dynamic-inventory-role"

filters:
  tag:Environment: awx-learning
  tag:Brick: brick-05
  instance-state-name: running

hostnames:
  - ip-address

keyed_groups:
  - key: tags.Environment
    prefix: env
    separator: "_"

compose:
  ansible_host: public_ip_address
  ansible_user: "'ec2-user'"
```

> `tag:Brick: brick-05` scopes the dynamic inventory to only Brick 5
> instances, keeping each brick cleanly separated.

### group_vars/aws_ec2.yml

```yaml
---
ansible_user: ec2-user
ansible_ssh_private_key_file: ~/.ssh/awx_learning_key
```

---

## Part 5 — IAM Role Permissions Update

The `amazon.aws` collection needs EC2 write permissions to create instances.
The current role only has read access from Brick 3. Update it.

1. **AWS Console → IAM → Roles → awx-dynamic-inventory-role → Permissions → Add permissions → Attach policies**
2. Attach: `AmazonEC2FullAccess`
3. Attach: `AmazonVPCReadOnlyAccess`

---

## Part 6 — Find the Amazon Linux 2023 AMI ID

Each AWS region has different AMI IDs. Run this on the Mac terminal to get
the latest one for the target region:

```bash
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023*" \
             "Name=architecture,Values=x86_64" \
             "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text \
  --region YOUR_REGION \
  --profile awx-automation
```

Copy the output — it looks like `ami-0abcdef1234567890`. This value goes into
`provision_ec2.yml`.

---

## Part 7 — provision_ec2.yml

This playbook runs against `localhost`. It calls the AWS API from the AWX
execution environment to create the EC2 instance, then waits until port 22
is reachable before finishing.

```yaml
---
- name: Provision EC2 instance
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    aws_region: YOUR_REGION
    instance_type: t2.micro
    ami_id: YOUR_AMAZON_LINUX_2023_AMI_ID
    key_pair_name: awx-brick5-key
    security_group_name: awx-learning-sg
    instance_name: awx-brick5-ec2
    ssh_public_key: "{{ lookup('file', '~/.ssh/awx_learning_key.pub') }}"

  tasks:
    - name: Ensure SSH key pair exists in AWS
      amazon.aws.ec2_key:
        name: "{{ key_pair_name }}"
        key_material: "{{ ssh_public_key }}"
        region: "{{ aws_region }}"
        state: present

    - name: Get default VPC
      amazon.aws.ec2_vpc_net_info:
        region: "{{ aws_region }}"
        filters:
          isDefault: "true"
      register: vpc_info

    - name: Get default subnet
      amazon.aws.ec2_vpc_subnet_info:
        region: "{{ aws_region }}"
        filters:
          vpc-id: "{{ vpc_info.vpcs[0].vpc_id }}"
          defaultForAz: "true"
      register: subnet_info

    - name: Ensure security group exists
      amazon.aws.ec2_security_group:
        name: "{{ security_group_name }}"
        description: AWX learning security group
        region: "{{ aws_region }}"
        vpc_id: "{{ vpc_info.vpcs[0].vpc_id }}"
        rules:
          - proto: tcp
            ports: [22]
            cidr_ip: 0.0.0.0/0
          - proto: tcp
            ports: [80]
            cidr_ip: 0.0.0.0/0
        state: present
      register: sg_info

    - name: Launch EC2 instance
      amazon.aws.ec2_instance:
        name: "{{ instance_name }}"
        region: "{{ aws_region }}"
        instance_type: "{{ instance_type }}"
        image_id: "{{ ami_id }}"
        key_name: "{{ key_pair_name }}"
        vpc_subnet_id: "{{ subnet_info.subnets[0].subnet_id }}"
        network:
          assign_public_ip: true
        security_groups:
          - "{{ security_group_name }}"
        tags:
          Environment: awx-learning
          Brick: brick-05
          Name: "{{ instance_name }}"
        state: running
        wait: true
      register: ec2_info

    - name: Show provisioned instance details
      ansible.builtin.debug:
        msg:
          - "Instance ID : {{ ec2_info.instances[0].instance_id }}"
          - "Public IP   : {{ ec2_info.instances[0].public_ip_address }}"

    - name: Wait for SSH to become available
      ansible.builtin.wait_for:
        host: "{{ ec2_info.instances[0].public_ip_address }}"
        port: 22
        delay: 15
        timeout: 180
        state: started
```

---

## Part 8 — configure.yml

`post_tasks` run after all roles complete — useful for tasks that depend on
what the roles set up.

```yaml
---
- name: Configure EC2 instances
  hosts: aws_ec2
  become: true
  gather_facts: true

  vars:
    system_timezone: "Europe/Paris"
    system_baseline_packages:
      - curl
      - wget
      - vim
      - sysstat
      - unzip

    # geerlingguy.nginx variables
    nginx_listen_port: 80
    nginx_vhosts:
      - listen: "80 default_server"
        server_name: "_"
        root: /usr/share/nginx/html
        index: index.html
        extra_parameters: |
          location / {
            try_files $uri $uri/ =404;
          }
    nginx_remove_default_vhost: false

  roles:
    - role: system_baseline
    - role: geerlingguy.nginx

  post_tasks:
    - name: Deploy custom index page
      ansible.builtin.copy:
        dest: /usr/share/nginx/html/index.html
        content: |
          <h1>Hello from AWX Brick 5</h1>
          <p>Host: {{ inventory_hostname }}</p>
          <p>OS: {{ ansible_distribution }} {{ ansible_distribution_version }}</p>
          <p>Provisioned and configured entirely by Ansible</p>
        mode: '0644'

    - name: Confirm nginx is responding
      ansible.builtin.uri:
        url: "http://{{ ansible_host }}"
        status_code: 200
      delegate_to: localhost
      become: false
```

---

## Part 9 — site.yml

```yaml
---
# Phase 1 — Provision infrastructure
- import_playbook: provision_ec2.yml

# Phase 2 — Configure provisioned instances
- import_playbook: configure.yml
```

---

## Part 10 — Push to GitHub

```bash
git add .
git commit -m "Brick 5: amazon.aws provisioning + geerlingguy.nginx Galaxy role"
git push origin main
```

---

## Part 11 — AWX Configuration

### Create Project

| Field | Value |
|---|---|
| Name | `Brick 05 - Collections` |
| Source Control Type | `Git` |
| URL | `https://github.com/USERNAME/awx-brick-05-collections.git` |
| Credential | `GitHub PAT` |
| Branch | `main` |

Save → 🔄 sync → wait for green.

### Create Inventory

1. **Inventories → Add → Add Inventory**
   - Name: `Brick5 AWS Dynamic` → Save
2. **Sources tab → Add**
   - Name: `EC2 Dynamic Source`
   - Source: `Amazon EC2`
   - Credential: `AWS IAM - awx-automation`
   - Enable: **Overwrite** ✅ **Update on launch** ✅
   - Source variables:

```yaml
---
plugin: amazon.aws.aws_ec2
regions:
  - YOUR_REGION
assume_role_arn: "arn:aws:iam::ACCOUNT_ID:role/awx-dynamic-inventory-role"
filters:
  tag:Environment: awx-learning
  tag:Brick: brick-05
  instance-state-name: running
hostnames:
  - ip-address
keyed_groups:
  - key: tags.Environment
    prefix: env
    separator: "_"
compose:
  ansible_host: public_ip_address
  ansible_user: "'ec2-user'"
```

3. Save — **do not sync yet**, the instance does not exist until the provision job runs.

### Job Template: Provision Only

| Field | Value |
|---|---|
| Name | `Brick5 - Provision EC2` |
| Inventory | `Brick5 AWS Dynamic` |
| Project | `Brick 05 - Collections` |
| Playbook | `provision_ec2.yml` |
| Credentials | `AWS IAM - awx-automation` |
| Privilege Escalation | ❌ Not needed — runs on localhost |

### Job Template: Full Site (Provision + Configure)

| Field | Value |
|---|---|
| Name | `Brick5 - Provision + Configure` |
| Inventory | `Brick5 AWS Dynamic` |
| Project | `Brick 05 - Collections` |
| Playbook | `site.yml` |
| Credentials | `AWS IAM - awx-automation` + `AWX Learning SSH Key` |
| Privilege Escalation | ✅ Enabled |

> Two credentials are attached — AWS for the provision phase, SSH for the
> configure phase. AWX supports multiple credentials per Job Template.

---

## Part 12 — Launch and Verify

### Recommended: Run in Two Steps (easier to debug)

**Step 1 — Provision:**
Templates → `Brick5 - Provision EC2` → 🚀

Wait for the job to complete, then check:
- AWX → **Inventories → Brick5 AWS Dynamic → 🔄 sync**
- The instance should appear in the Hosts tab

**Step 2 — Full run:**
Templates → `Brick5 - Provision + Configure` → 🚀

### Expected Output

```
PLAY [Provision EC2 instance] ******************************************

TASK [Ensure SSH key pair exists in AWS]
ok: [localhost]

TASK [Launch EC2 instance]
changed: [localhost]

TASK [Wait for SSH to become available]
ok: [localhost]

PLAY [Configure EC2 instances] *****************************************

TASK [system_baseline : Set hostname]
changed: [x.x.x.x]

TASK [geerlingguy.nginx : Install nginx]
changed: [x.x.x.x]

TASK [Deploy custom index page]
changed: [x.x.x.x]

TASK [Confirm nginx is responding]
ok: [x.x.x.x]

PLAY RECAP *************************************************************
localhost : ok=5  changed=1  unreachable=0  failed=0
x.x.x.x  : ok=9  changed=6  unreachable=0  failed=0
```

### Idempotency Check

Run `site.yml` a second time — the provision phase reports `ok` (instance
already exists), the configure phase reports `ok` (nothing to change).

### Browser Check

```
http://EC2_PUBLIC_IP
```

Expected:

```
Hello from AWX Brick 5
Host: x.x.x.x
OS: Amazon Linux 2023
Provisioned and configured entirely by Ansible
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `ec2_key` module not found | `amazon.aws` collection not installed | Verify `collections/requirements.yml` exists and re-sync project |
| Wrong AMI ID | AMI IDs differ per region | Run the `aws ec2 describe-images` command in Part 6 for the exact region |
| Phase 2 finds no hosts | Instance just created, inventory not yet synced | Run provision job separately first, sync inventory manually, then run configure |
| `geerlingguy.nginx` not found | AWX looks for `roles/requirements.yml` specifically | Confirm file is at `roles/requirements.yml` not repo root, re-sync |
| SSH times out in `wait_for` | Security group not applied or no internet gateway | Increase `timeout` to `300`, verify security group has port 22 from `0.0.0.0/0` |
| Permission denied creating EC2 | IAM role still has read-only permissions | Attach `AmazonEC2FullAccess` to `awx-dynamic-inventory-role` (Part 5) |

---

## Cleanup

```bash
# Get the instance ID
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Brick,Values=brick-05" \
  --query "Reservations[*].Instances[*].InstanceId" \
  --output text \
  --region YOUR_REGION \
  --profile awx-automation)

# Terminate it
aws ec2 terminate-instances \
  --instance-ids $INSTANCE_ID \
  --region YOUR_REGION \
  --profile awx-automation
```

---

## AWX Objects Created

| Type | Name |
|---|---|
| Project | `Brick 05 - Collections` |
| Inventory | `Brick5 AWS Dynamic` |
| Template | `Brick5 - Provision EC2` |
| Template | `Brick5 - Provision + Configure` |

---

## Key Concepts Introduced in Brick 05

| Concept | What it means |
|---|---|
| Galaxy role | A pre-built, community-maintained role pulled from galaxy.ansible.com |
| `roles/requirements.yml` | Tells AWX which Galaxy roles to install before running jobs |
| `collections/requirements.yml` | Tells AWX which collections to install before running jobs |
| `import_playbook` | Chains multiple playbooks into one execution |
| `post_tasks` | Tasks that run after all roles in a play have completed |
| Localhost provisioning | Running a play against `localhost` to call cloud APIs rather than SSH into a server |

---

## What is Next — Brick 06

Brick 06 introduces Molecule — a testing framework for Ansible roles. The
`system_baseline` and `nginx` roles get unit-tested locally using Docker
containers as fake hosts, with a full test → converge → verify → destroy cycle
running on the Mac before anything is pushed to AWX.