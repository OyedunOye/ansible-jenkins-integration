# Ansible Integration in Jenkins

This project shows how to run Ansible from a Jenkins pipeline without installing Ansible on the Jenkins server itself. Jenkins hands the playbook, its configuration and an SSH key to a dedicated **Ansible control node**, then tells that node to run `ansible-playbook` over SSH. The control node uses the **AWS EC2 dynamic inventory plugin** to discover which EC2 instances to configure, so no server IP addresses are hard coded anywhere.

The playbook installs and starts Docker and installs the Docker Compose plugin on every EC2 instance it finds.

## How it works

```
+-----------------+    1. scp ansible/* and EC2 key     +------------------------+
|                 | ----------------------------------> |                        |
|     Jenkins     |                                     |  Ansible control node  |
|                 |    2. ssh: ansible-playbook ...     |  (ANSIBLE_SERVER)      |
|                 | ----------------------------------> |                        |
+-----------------+                                     +------------------------+
                                                           |                 |
                                   3. query AWS API for    |                 | 4. SSH as ec2-user
                                      running instances    v                 v    and configure
                                                     +-----------+    +-----------------+
                                                     |  AWS EC2  |    |  EC2 instances  |
                                                     |    API    |    |  (Docker, etc.) |
                                                     +-----------+    +-----------------+
```

1. Jenkins copies the contents of [ansible/](ansible/) to `/root` on the control node, along with the private key that grants access to the EC2 instances.
2. Jenkins opens an SSH session to the control node and runs `ansible-playbook my-playbook.yaml`.
3. Ansible reads [ansible.cfg](ansible/ansible.cfg), which points at the dynamic inventory file. The `aws_ec2` plugin calls the AWS API and builds the host list at run time.
4. Ansible connects to each discovered instance as `ec2-user` using the copied key and applies the playbook.

## Repository layout

| Path | Purpose |
| --- | --- |
| [Jenkinsfile](Jenkinsfile) | Declarative pipeline with the two Ansible stages |
| [ansible/ansible.cfg](ansible/ansible.cfg) | Ansible settings: inventory file, enabled plugin, remote user, SSH key path |
| [ansible/inventory_aws_ec2.yaml](ansible/inventory_aws_ec2.yaml) | AWS EC2 dynamic inventory definition |
| [ansible/my-playbook.yaml](ansible/my-playbook.yaml) | Playbook that installs Docker and Docker Compose |
| [prepare-ansible-server.sh](prepare-ansible-server.sh) | One time setup script for the control node (Ansible and boto3) |
| `src/`, [pom.xml](pom.xml), [Dockerfile](Dockerfile) | Sample Java Maven application (not used by the Ansible stages) |

## The Jenkins pipeline

The pipeline defines the control node address once as an environment variable:

```groovy
environment {
    ANSIBLE_SERVER = "172.104.233.61"
}
```

Change this value if your control node lives somewhere else.

### Stage 1: copy files to ansible server

```groovy
sshagent (['ansible-server-key']) {
    sh "scp -o StrictHostKeyChecking=no ansible/* root@${ANSIBLE_SERVER}:/root"
    withCredentials([sshUserPrivateKey(credentialsId: 'ansible-configured-ec2-server-key', keyFileVariable: 'keyfile', usernameVariable: 'user')]) {
        sh 'scp $keyfile root@$ANSIBLE_SERVER:/root/ssh-key.pem'
    }
}
```

* `sshagent` loads the control node key so `scp` can authenticate as `root`.
* Everything in `ansible/` lands in `/root`, so `ansible.cfg` sits in the directory the playbook will be run from and is picked up automatically.
* The EC2 key is written to `/root/ssh-key.pem`, which matches `private_key_file = ~/ssh-key.pem` in `ansible.cfg`.
* The key is passed with single quotes so Groovy does not interpolate the secret into the command string; the shell expands it instead.

### Stage 2: execute ansible playbook

```groovy
def remote = [:]
remote.name = "my-ansible-server"
remote.host = env.ANSIBLE_SERVER
remote.allowAnyHosts = true

withCredentials([sshUserPrivateKey(credentialsId: 'ansible-server-key', keyFileVariable: 'keyfile', usernameVariable: 'user')]) {
    remote.user = user
    remote.identityFile = keyfile
    sshCommand remote: remote, command: "ansible-playbook my-playbook.yaml"
}
```

This stage uses the **SSH Pipeline Steps** plugin. It builds a `remote` object describing the control node and runs the playbook there with `sshCommand`. The Ansible output streams back into the Jenkins console log, and a failed playbook fails the build.

There is also a commented out line that can run the control node setup script remotely:

```groovy
// sshScript remote: remote, script: "prepare-ansible-server.sh"
```

Uncomment it to have Jenkins provision a fresh control node as part of the pipeline.

## Dynamic inventory

Instead of a static list of hosts, [inventory_aws_ec2.yaml](ansible/inventory_aws_ec2.yaml) uses the `aws_ec2` inventory plugin:

```yaml
plugin: aws_ec2
regions:
 - us-east-1

# filters:
#   tag:Name: dev*

keyed_groups:
  - key: tags
  - key: instance_type
    prefix: instance_type
```

What each part does:

* **`plugin: aws_ec2`** tells Ansible to build the inventory by querying the EC2 API. The file name must end in `aws_ec2.yaml` (or `aws_ec2.yml`) for the plugin to accept it.
* **`regions`** limits discovery to `us-east-1`. Add more regions to the list if your instances are spread out.
* **`filters`** (commented out) narrows discovery to matching instances, for example only those whose `Name` tag starts with `dev`. Without a filter, every instance in the region is targeted.
* **`keyed_groups`** creates groups on the fly from instance metadata. With the config above you get groups such as `tag_Name_dev_server` and `instance_type_t2_micro`, which you can use as `hosts:` targets in a playbook instead of `all`.

[ansible.cfg](ansible/ansible.cfg) wires it all together:

```ini
[defaults]
host_key_checking = False
inventory = inventory_aws_ec2.yaml
enable_plugins = aws_ec2
remote_user = ec2-user
private_key_file = ~/ssh-key.pem
```

Because the inventory is resolved when the playbook runs, new instances are configured on the next pipeline run with no changes to the repository, and terminated instances simply drop out.

To inspect what the plugin discovers, run these on the control node:

```bash
ansible-inventory -i inventory_aws_ec2.yaml --list
ansible-inventory -i inventory_aws_ec2.yaml --graph
```

## Setup

### 1. Prepare the Ansible control node

On an Ubuntu or Debian server, run [prepare-ansible-server.sh](prepare-ansible-server.sh) as root:

```bash
apt update
apt install ansible -y
apt install python3-boto3
```

`boto3` is the AWS SDK for Python and is required by the `aws_ec2` plugin.

The control node also needs AWS credentials with permission to describe EC2 instances (for example `ec2:DescribeInstances`). Configure them for the `root` user, either with `aws configure` or by creating `/root/.aws/credentials`:

```ini
[default]
aws_access_key_id = <your access key>
aws_secret_access_key = <your secret key>
```

### 2. Provision the EC2 instances

Create the target instances in `us-east-1` using an Amazon Linux AMI (the playbook uses `yum` and connects as `ec2-user`). Make sure their security group allows SSH from the control node, and keep the key pair `.pem` file for the next step.

### 3. Configure Jenkins

Install these plugins:

* **SSH Agent** (provides `sshagent`)
* **SSH Pipeline Steps** (provides `sshCommand` and `sshScript`)
* **Credentials Binding** (provides `withCredentials`, usually installed by default)

Add two credentials of type **SSH Username with private key**:

| Credential ID | Username | Private key |
| --- | --- | --- |
| `ansible-server-key` | `root` | Key that lets Jenkins SSH into the Ansible control node |
| `ansible-configured-ec2-server-key` | `ec2-user` | Key pair `.pem` for the EC2 instances |

### 4. Create the pipeline job

Create a Pipeline (or Multibranch Pipeline) job pointing at this repository, set `ANSIBLE_SERVER` in the [Jenkinsfile](Jenkinsfile) to your control node IP, and run the build.

## Verifying the result

After a successful run, SSH into any of the EC2 instances and check:

```bash
sudo systemctl status docker
docker compose version
```
