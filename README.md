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
This file must be at `roles/requirements.yml` — not the repo root.

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

## Part 6 — Gather Required AWS Values Manually

> **Important:** The following values must be collected manually from the AWS
> account before editing any playbook files. Dynamic VPC and subnet lookup
> is not reliable with the collection version bundled in AWX — hardcoding
> is the correct approach here.

### 6a — Amazon Linux 2023 AMI ID

AMI IDs differ per region. Run this on the Mac terminal:

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

### 6b — VPC ID

> **Important:** Do not assume a default VPC exists — it may have been
> deleted. List all available VPCs and pick the correct one explicitly.

```bash
aws ec2 describe-vpcs \
  --region YOUR_REGION \
  --profile awx-automation \
  --query "Vpcs[*].{ID:VpcId,Name:Tags[?Key=='Name']|[0].Value,CIDR:CidrBlock}" \
  --output table
```

### 6c — Subnet ID

> **Important:** The subnet must have `MapPublicIpOnLaunch = True`. A private
> subnet will cause the configure phase to fail — AWX running inside Minikube
> on the Mac cannot reach a private IP address.

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=YOUR_VPC_ID" \
  --region YOUR_REGION \
  --profile awx-automation \
  --query "Subnets[*].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,Public:MapPublicIpOnLaunch}" \
  --output table
```

Pick a subnet where `Public = True`.

### 6d — Security Group Check

If `awx-learning-sg` already exists from a previous brick, verify it is in
the same VPC. AWS rejects duplicate names within the same VPC.

```bash
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=awx-learning-sg" \
  --region YOUR_REGION \
  --profile awx-automation \
  --query "SecurityGroups[*].{ID:GroupId,VPC:VpcId,Name:GroupName}" \
  --output table
```

| Result | Action |
|---|---|
| Empty | Nothing — the playbook creates it |
| Found in same VPC | Nothing — the playbook reconciles it |
| Found in a different VPC | Change `security_group_name` to `awx-brick5-sg` in the playbook |

---

## Part 7 — provision_ec2.yml

> **Important notes:**
> - Replace all placeholder values (`YOUR_REGION`, `YOUR_AMI_ID`,
>   `YOUR_VPC_ID`, `YOUR_SUBNET_ID`, `ACCOUNT_ID`) with values from Part 6
> - `ssh_public_key` is intentionally a placeholder — injected via AWX
>   Extra Variables at runtime (see Part 12)
> - `module_defaults` group syntax and `assume_role` as a module parameter
>   are not supported in the AWX bundled collection version — role assumption
>   is handled via an explicit `sts_assume_role` task, and temporary
>   credentials are passed to every subsequent task individually

```yaml
---
- name: Provision EC2 instance
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    aws_region: YOUR_REGION
    instance_type: t2.micro
    ami_id: YOUR_AMI_ID
    key_pair_name: awx-brick5-key
    security_group_name: awx-learning-sg
    instance_name: awx-brick5-ec2
    assume_role_arn: "arn:aws:iam::ACCOUNT_ID:role/awx-dynamic-inventory-role"
    ssh_public_key: "SET_VIA_AWX_EXTRA_VARS"
    vpc_id: "YOUR_VPC_ID"        # set manually — see Part 6b
    subnet_id: "YOUR_SUBNET_ID"  # set manually — must be public — see Part 6c

  tasks:
    - name: Assume IAM role and get temporary credentials
      amazon.aws.sts_assume_role:
        role_arn: "{{ assume_role_arn }}"
        role_session_name: "awx-provision-session"
        region: "{{ aws_region }}"
      register: assumed_role

    - name: Set temporary credential facts
      ansible.builtin.set_fact:
        tmp_key:    "{{ assumed_role.sts_creds.access_key }}"
        tmp_secret: "{{ assumed_role.sts_creds.secret_key }}"
        tmp_token:  "{{ assumed_role.sts_creds.session_token }}"

    - name: Ensure SSH key pair exists in AWS
      amazon.aws.ec2_key:
        name: "{{ key_pair_name }}"
        key_material: "{{ ssh_public_key }}"
        region: "{{ aws_region }}"
        access_key: "{{ tmp_key }}"
        secret_key: "{{ tmp_secret }}"
        session_token: "{{ tmp_token }}"
        state: present

    - name: Ensure security group exists
      amazon.aws.ec2_security_group:
        name: "{{ security_group_name }}"
        description: AWX learning security group
        region: "{{ aws_region }}"
        access_key: "{{ tmp_key }}"
        secret_key: "{{ tmp_secret }}"
        session_token: "{{ tmp_token }}"
        vpc_id: "{{ vpc_id }}"
        rules:
          - proto: tcp
            ports: [22]
            cidr_ip: 0.0.0.0/0
          - proto: tcp
            ports: [80]
            cidr_ip: 0.0.0.0/0
        state: present

    - name: Launch EC2 instance
      amazon.aws.ec2_instance:
        name: "{{ instance_name }}"
        region: "{{ aws_region }}"
        access_key: "{{ tmp_key }}"
        secret_key: "{{ tmp_secret }}"
        session_token: "{{ tmp_token }}"
        instance_type: "{{ instance_type }}"
        image_id: "{{ ami_id }}"
        key_name: "{{ key_pair_name }}"
        vpc_subnet_id: "{{ subnet_id }}"
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

> **Important:** The `system_baseline_packages` list defined here overrides
> `roles/system_baseline/defaults/main.yml` entirely due to variable priority.
> Do not include `curl` — Amazon Linux 2023 ships with `curl-minimal` which
> conflicts with the full `curl` package and causes a dnf dependency error.

```yaml
---
- name: Configure EC2 instances
  hosts: aws_ec2
  become: true
  gather_facts: true

  vars:
    system_timezone: "Europe/Paris"
    system_baseline_packages:   # curl deliberately excluded — conflicts with curl-minimal
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

## Part 9 — system_baseline defaults

> Keep defaults consistent with `configure.yml` — `curl` excluded from both.

### roles/system_baseline/defaults/main.yml

```yaml
---
system_timezone: "Europe/Paris"

system_baseline_packages:   # curl excluded — conflicts with curl-minimal on Amazon Linux 2023
  - wget
  - vim
  - sysstat
  - unzip

system_update_packages: false
```

---

## Part 10 — site.yml

```yaml
---
# Phase 1 — Provision infrastructure
- import_playbook: provision_ec2.yml

# Phase 2 — Configure provisioned instances
- import_playbook: configure.yml
```

---

## Part 11 — Push to GitHub

```bash
git add .
git commit -m "Brick 5: amazon.aws provisioning + geerlingguy.nginx Galaxy role"
git push origin main
```

---

## Part 12 — AWX Configuration

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
   - Source variables: same as `inventories/aws/aws_ec2.yml` above
3. Save — **do not sync yet**, the instance does not exist until the provision job runs

### Job Template: Provision Only

| Field | Value |
|---|---|
| Name | `Brick5 - Provision EC2` |
| Inventory | `Brick5 AWS Dynamic` |
| Project | `Brick 05 - Collections` |
| Playbook | `provision_ec2.yml` |
| Credentials | `AWS IAM - awx-automation` |
| Privilege Escalation | ❌ Not needed — runs on localhost |

**Extra Variables** — required, the SSH public key cannot be read from the
Mac filesystem inside the AWX Minikube container:

```yaml
---
ssh_public_key: "PASTE_FULL_PUBLIC_KEY_HERE"
```

Get the value with: `cat ~/.ssh/awx_learning_key.pub`

### Job Template: Full Site

| Field | Value |
|---|---|
| Name | `Brick5 - Provision + Configure` |
| Inventory | `Brick5 AWS Dynamic` |
| Project | `Brick 05 - Collections` |
| Playbook | `site.yml` |
| Credentials | `AWS IAM - awx-automation` + `AWX Learning SSH Key` |
| Privilege Escalation | ✅ Enabled |

**Extra Variables** — same `ssh_public_key` entry as above.

> Two credentials are required — AWS for the provision phase, SSH for the
> configure phase. AWX supports multiple credentials per Job Template.

---

## Part 13 — Launch and Verify

### Recommended: Run in Two Steps

**Step 1 — Provision:**
Templates → `Brick5 - Provision EC2` → 🚀

After completion, manually sync the inventory:
**Inventories → Brick5 AWS Dynamic → Sources → 🔄 sync**

Confirm the instance appears in the **Hosts tab** before proceeding.

**Step 2 — Full run:**
Templates → `Brick5 - Provision + Configure` → 🚀

### Expected Output (second run — fully idempotent)

```
PLAY [Provision EC2 instance] ******************************************

TASK [Assume IAM role and get temporary credentials]
changed: [localhost]

TASK [Ensure SSH key pair exists in AWS]
ok: [localhost]

TASK [Launch EC2 instance]
ok: [localhost]

TASK [Wait for SSH to become available]
ok: [localhost]

PLAY [Configure EC2 instances] *****************************************

TASK [system_baseline : Set hostname]
ok: [x.x.x.x]

TASK [system_baseline : Install baseline packages (Amazon Linux / RedHat)]
ok: [x.x.x.x]

TASK [geerlingguy.nginx : Install nginx]
ok: [x.x.x.x]

TASK [Deploy custom index page]
ok: [x.x.x.x]

TASK [Confirm nginx is responding]
ok: [x.x.x.x]

PLAY RECAP *************************************************************
localhost : ok=7  changed=0  unreachable=0  failed=0
x.x.x.x  : ok=9  changed=0  unreachable=0  failed=0
```

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
| `lookup('file') — file not found` | AWX runs in a Minikube container — Mac filesystem not accessible | Pass `ssh_public_key` via AWX Extra Variables |
| `module_defaults group not supported` | Older collection version in AWX | Remove `module_defaults` — use explicit credentials per task |
| `Unsupported parameter: assume_role` | Collection version does not support it as module param | Use `sts_assume_role` task, pass `tmp_key/tmp_secret/tmp_token` per task |
| `list object has no element 0` on VPC lookup | No default VPC — filter returned empty | Hardcode `vpc_id` and `subnet_id` — gather with CLI (Part 6) |
| `curl conflicts with curl-minimal` | Amazon Linux 2023 ships `curl-minimal` | Remove `curl` from `system_baseline_packages` in both `configure.yml` and `defaults/main.yml` |
| Phase 2 finds no hosts | Inventory not synced after provision | Manually sync inventory source after provision job completes |
| `geerlingguy.nginx` not found | Wrong requirements file location | File must be at `roles/requirements.yml` — not repo root |
| Security group VPC mismatch | `awx-learning-sg` exists in a different VPC | Change `security_group_name` to `awx-brick5-sg` |

---

## Known Limitations in This Setup

| Limitation | Reason | Workaround |
|---|---|---|
| `vpc_id` and `subnet_id` hardcoded | No default VPC — dynamic lookup returns empty | Gather manually via CLI before editing the playbook (Part 6) |
| `ssh_public_key` via Extra Variables | AWX Minikube container cannot access Mac filesystem | Set in AWX Job Template Extra Variables |
| Explicit credentials on every task | Older `amazon.aws` collection — `module_defaults` group not supported | Accepted limitation for this AWX version |
| Ansible version warning | AWX execution environment uses Ansible 2.15 | Informational only — does not block execution |

---

## Cleanup

```bash
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Brick,Values=brick-05" \
  --query "Reservations[*].Instances[*].InstanceId" \
  --output text \
  --region YOUR_REGION \
  --profile awx-automation)

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

## Key Concepts Introduced

| Concept | What it means |
|---|---|
| Galaxy role | Pre-built community-maintained role pulled from galaxy.ansible.com |
| `roles/requirements.yml` | Tells AWX which Galaxy roles to install before running jobs |
| `collections/requirements.yml` | Tells AWX which collections to install before running jobs |
| `sts_assume_role` task | Explicitly assumes an IAM role and captures short-lived credentials |
| `import_playbook` | Chains multiple playbooks into one execution |
| `post_tasks` | Tasks that run after all roles in a play have completed |
| Localhost provisioning | Play running against `localhost` to call cloud APIs — no SSH |
| Variable priority | Playbook `vars` block always overrides `role/defaults/main.yml` |

---

## What is Next — Brick 06

Brick 06 introduces Molecule — a testing framework for Ansible roles. The
`system_baseline` role gets unit-tested locally using Docker containers as
fake hosts, with a full test → converge → verify → destroy cycle running
on the Mac before anything is pushed to AWX.