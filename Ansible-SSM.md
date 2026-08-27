# Ansible → AWS Systems Manager (SSM) → EC2

Yes. If the requirement is **10 EC2 instances, deploy Nginx on all of them using Ansible, but absolutely no SSH**, the clean AWS-native approach is:

**Ansible → AWS Systems Manager (SSM) → EC2**

You don't need port 22 open, SSH keys, or a bastion host.

### Architecture

```text
                 Your Laptop / Ansible Controller
                            |
                            | HTTPS :443
                            v
                  AWS Systems Manager
                            |
          +-----------------+-----------------+
          |        |        |        |         |
         EC2-1   EC2-2    EC2-3    ...      EC2-10
          |        |        |                 |
       SSM Agent / SSM Agent / SSM Agent ... SSM Agent
          |
       Install Nginx
```

The important point is that **Ansible itself normally uses SSH**, but we're going to configure Ansible to use the **AWS SSM connection plugin** instead.

---

# 1. Prerequisites on the 10 EC2 instances

Assume:

* 10 EC2 instances
* Amazon Linux 2023 or Ubuntu
* All instances are in your VPC
* No SSH access
* You want to install Nginx

Each EC2 needs the **SSM Agent**.

For Amazon Linux 2023, SSM Agent is normally already installed.

Check from AWS Systems Manager:

**AWS Console → Systems Manager → Fleet Manager**

You should see all 10 instances as managed nodes.

---

# 2. Give EC2 permission to use SSM

Create an IAM role for the EC2 instances.

Attach:

```text
AmazonSSMManagedInstanceCore
```

For example:

```text
Role:
    EC2-SSM-Role

Policy:
    AmazonSSMManagedInstanceCore
```

Attach this IAM role to all 10 EC2 instances.

The important permission is not an Ansible permission.

It is the **EC2 instance's permission to communicate with SSM**.

---

# 3. Network connectivity

This is a very important interview point.

You don't need:

```text
Internet → EC2:22
```

You need the EC2 instances to communicate with AWS Systems Manager.

They need outbound HTTPS connectivity to the required AWS endpoints.

You can achieve this through:

### Option A — NAT Gateway

```text
Private EC2
    |
    v
NAT Gateway
    |
    v
Internet
    |
    v
AWS SSM
```

### Option B — VPC Endpoints

For a completely private architecture, create VPC interface endpoints for:

```text
com.amazonaws.<region>.ssm
com.amazonaws.<region>.ssmmessages
com.amazonaws.<region>.ec2messages
```

Depending on the SSM/agent architecture and AWS region, the endpoint requirements can vary, so verify the current AWS SSM networking requirements for your region.

Security group should allow outbound HTTPS:

```text
EC2 SG
Outbound:
TCP 443
```

---

# 4. Install Ansible on your controller

Your laptop or an EC2 instance can act as the Ansible controller.

For Ubuntu:

```bash
sudo apt update
sudo apt install ansible -y
```

Check:

```bash
ansible --version
```

You should get something like:

```text
ansible [core ...]
```

---

# 5. Install the AWS SSM Ansible connection plugin

The easiest approach is to install the Ansible collection that provides the SSM connection functionality.

For example:

```bash
ansible-galaxy collection install amazon.aws
```

You also need the AWS SDK:

```bash
pip install boto3 botocore
```

Verify:

```bash
ansible-galaxy collection list
```

You should see:

```text
amazon.aws
```

---

# 6. Configure AWS credentials

Your Ansible controller needs permission to interact with AWS.

For a local machine, preferably configure an AWS profile:

```bash
aws configure
```

Enter:

```text
AWS Access Key ID
AWS Secret Access Key
Default region
Output format
```

Better in production:

```text
AWS IAM Identity Center
```

or an IAM role if your Ansible controller itself runs on AWS.

You can verify:

```bash
aws sts get-caller-identity
```

---

# 7. Create the Ansible project

Create:

```text
ansible-nginx/
├── inventory/
│   └── hosts.ini
├── group_vars/
│   └── all.yml
├── playbook.yml
└── ansible.cfg
```

---

# 8. Inventory — identify the 10 EC2 instances

Here's where it gets interesting.

Because we don't want SSH, we don't need to put:

```text
ansible_host=10.0.1.10
```

and we don't need SSH configuration.

We can identify the instances through AWS.

For a simple demo, you can create an inventory containing the EC2 instance IDs:

```ini
[webservers]
i-0123456789abcdef0
i-0123456789abcdef1
i-0123456789abcdef2
i-0123456789abcdef3
i-0123456789abcdef4
i-0123456789abcdef5
i-0123456789abcdef6
i-0123456789abcdef7
i-0123456789abcdef8
i-0123456789abcdef9
```

But there is an even better production approach:

**Don't manually maintain 10 instance IDs.**

Use the AWS dynamic inventory plugin.

---

# 9. Dynamic inventory

Create:

```text
inventory/aws_ec2.yml
```

Example:

```yaml
plugin: amazon.aws.aws_ec2

regions:
  - ap-south-1

filters:
  tag:Environment: production
  instance-state-name: running

keyed_groups:
  - key: tags.Role
    prefix: role
```

Now if your EC2 instances have:

```text
Environment = production
Role = web
```

Ansible automatically discovers them.

You don't have to maintain:

```text
EC2-1
EC2-2
...
EC2-10
```

manually.

---

# 10. Configure Ansible to use SSM instead of SSH

This is the critical part.

In your `ansible.cfg`:

```ini
[defaults]
inventory = inventory/aws_ec2.yml
host_key_checking = False
interpreter_python = auto_silent
```

Then your inventory variables need to specify the SSM connection.

Conceptually:

```yaml
ansible_connection: aws_ssm
```

Instead of:

```yaml
ansible_connection: ssh
```

That's the key difference.

---

# 11. How SSM actually works

Normally:

```text
Ansible
   |
   | SSH :22
   v
EC2
```

But in your requirement:

```text
Ansible
   |
   | AWS API
   v
SSM
   |
   | SSM channel
   v
EC2
```

So:

```text
NO SSH
NO PORT 22
NO SSH KEY
NO BASTION
```

SSM Agent running on the EC2 receives the command and executes it locally.

---

# 12. Test connectivity before installing Nginx

This is something I'd strongly recommend.

Don't immediately run the Nginx playbook.

First test:

```bash
ansible all -m ping
```

With the SSM connection configured correctly, you should get:

```text
i-0123456789abcdef0 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

i-0123456789abcdef1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

...
```

All 10 should return:

```text
SUCCESS
```

If one doesn't, troubleshoot SSM before touching Nginx.

---

# 13. Create the Nginx playbook

Now create:

```text
playbook.yml
```

For Amazon Linux:

```yaml
---
- name: Deploy Nginx
  hosts: all
  become: true

  tasks:

    - name: Install Nginx
      ansible.builtin.dnf:
        name: nginx
        state: present

    - name: Enable and start Nginx
      ansible.builtin.systemd:
        name: nginx
        state: started
        enabled: true

    - name: Verify Nginx
      ansible.builtin.uri:
        url: http://localhost
        status_code: 200
```

Run:

```bash
ansible-playbook playbook.yml
```

Ansible will execute the tasks on all 10 EC2 instances.

---

# 14. If you're using Ubuntu

The package installation changes.

```yaml
---
- name: Deploy Nginx
  hosts: all
  become: true

  tasks:

    - name: Install Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Enable and start Nginx
      ansible.builtin.systemd:
        name: nginx
        state: started
        enabled: true

    - name: Verify Nginx
      ansible.builtin.uri:
        url: http://localhost
        status_code: 200
```

---

# 15. Better approach — handle Amazon Linux + Ubuntu

If your 10 EC2s aren't necessarily the same OS, don't hardcode `apt` or `dnf`.

You can use:

```yaml
---
- name: Deploy Nginx
  hosts: all
  become: true

  tasks:

    - name: Install Nginx
      ansible.builtin.package:
        name: nginx
        state: present

    - name: Enable and start Nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

    - name: Verify Nginx
      ansible.builtin.uri:
        url: http://localhost
        status_code: 200
```

This is cleaner because Ansible uses the appropriate package manager for the OS.

---

# 16. Make the deployment idempotent

This is another important Ansible concept.

If you execute:

```bash
ansible-playbook playbook.yml
```

again, Ansible shouldn't reinstall/restart everything unnecessarily.

That's why we use:

```yaml
state: present
```

and:

```yaml
state: started
enabled: true
```

instead of shell commands like:

```yaml
shell: yum install nginx
```

or:

```yaml
shell: systemctl start nginx
```

Ansible understands the desired state.

---

# 17. Verify all 10 machines

You can run:

```bash
ansible all -m shell -a "systemctl is-active nginx"
```

Expected:

```text
i-001 | CHANGED | rc=0 >>
active

i-002 | CHANGED | rc=0 >>
active

i-003 | CHANGED | rc=0 >>
active

...
```

Or:

```bash
ansible all -m uri -a "url=http://localhost"
```

You should receive HTTP 200 responses.

---

# 18. What happens internally?

Suppose you run:

```bash
ansible-playbook playbook.yml
```

### Step 1

Ansible discovers:

```text
10 EC2 instances
```

through AWS dynamic inventory.

### Step 2

Ansible sees:

```yaml
ansible_connection: aws_ssm
```

Therefore it does **not** try:

```text
SSH → EC2:22
```

### Step 3

Ansible communicates with AWS/SSM.

### Step 4

SSM identifies:

```text
i-001
i-002
...
i-010
```

### Step 5

SSM Agent on each EC2 receives the commands.

### Step 6

The commands execute locally:

```text
dnf install nginx
systemctl enable nginx
systemctl start nginx
```

### Step 7

Results return through SSM.

So the communication path is:

```text
Ansible Controller
       |
       | AWS API
       v
AWS Systems Manager
       |
       +---------> EC2-1 → SSM Agent → Nginx
       |
       +---------> EC2-2 → SSM Agent → Nginx
       |
       +---------> EC2-3 → SSM Agent → Nginx
       |
       ...
       |
       +---------> EC2-10 → SSM Agent → Nginx
```

---

# 19. Production architecture I'd recommend

For a real environment, I'd structure it like this:

```text
                         AWS
                          |
                +---------+---------+
                |                   |
          Systems Manager       EC2 API
                |
                |
        +-------+-------+
        |               |
      SSM Agent       SSM Agent
        |               |
      EC2-1           EC2-10
        |               |
      Nginx           Nginx
```

And:

```text
Ansible Controller
        |
        | HTTPS
        v
AWS APIs
        |
        v
Systems Manager
        |
        v
Private EC2
```

With:

* **No SSH**
* **No port 22**
* **No SSH keys**
* **No bastion**
* **IAM-based authentication**
* **SSM Agent**
* **VPC endpoints or NAT**
* **Ansible**
* **Dynamic EC2 inventory**

---

## One important interview clarification

If an interviewer says:

> "Deploy Nginx on 10 EC2 using Ansible without SSH."

I'd answer:

> **"Ansible traditionally uses SSH for Linux hosts, but since SSH is prohibited, I would use AWS Systems Manager as the transport. I'd install/enable SSM Agent on the EC2 instances, attach an IAM role with AmazonSSMManagedInstanceCore, provide the instances with outbound HTTPS connectivity to SSM—preferably through VPC endpoints for private instances—and configure Ansible's AWS SSM connection plugin. I'd use the amazon.aws EC2 dynamic inventory to discover the 10 instances and then run an idempotent Nginx playbook across them."**

That's a **very strong Senior DevOps answer** because it addresses not only Ansible but also **IAM, networking, SSM, dynamic inventory, security, and idempotency**.
