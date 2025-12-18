# SUSE-AI Node Setup via Ansible

This project automates the setup of a high-availability RKE2 cluster with Rancher on SUSE-based systems, providing a reliable foundation for deploying SUSE AI applications. It streamlines installation through containerized Ansible playbooks and, when enabled, adds GPU support by installing the NVIDIA driver and GPU Operator.

*Note:* The initial implementation is designed to work with the default configuration options for both RKE2 and Rancher.


## Prerequisites

- Docker or Podman
- SSH key-based access to all target nodes
- Target hosts are SUSE based OS
- Proper DNS setup (e.g. `rancher.example.com`)
- Target hosts must fulfill prerequisites at https://docs.rke2.io/install/quickstart#prerequisites
- Target hosts with python 3.11+ version. Verify that Python 3 points to version 3.11 or higher. Run ```python3 --version```
- A valid registration key for the SUSE OS enterprise distribution, which can be obtained with your SUSE subscription.

## Components

- Ansible playbooks for:
  - Optional installation of nvidia driver packages
  - RKE2 HA server installation
  - RKE2 agent node installation
  - Optional deployment of Rancher
  - Optional deployment of nvidia gpu-operator
- Roles for idempotent configuration
- A Dockerfile to run the playbooks in a container

## Inventory Example

Identify the servers and agents that must belong to the same RKE2 cluster.

This is an example of inventory.ini file with 3 RKE2 Servers and 2 RKE2 Agents.
```bash
#inventory.ini.example
[rke2_servers]
rke2_server1 ansible_host=192.168.1.10
rke2_server2 ansible_host=192.168.1.11
rke2_server3 ansible_host=192.168.1.12

[rke2_agents]
rke2_agent1 ansible_host=192.168.1.20
rke2_agent2 ansible_host=192.168.1.21

[all:vars]
ansible_user=<SSH_USER>
```

This is an example of inventory.ini file with 1 RKE2 server.

```bash
#inventory.ini.onenode.example
[rke2_servers]
rke2_server1 ansible_host=192.168.1.10

[all:vars]
ansible_user=<SSH_USER>
```

This is an example of inventory.ini file with target host being the localhost.

```bash
##inventory.ini.local.example
[rke2_servers]
rke2_server1 ansible_host=localhost

[all:vars]
ansible_user=<SSH_USER>
```

## Notes

- Mount your SSH keys under `~/.ssh` to enable access to target nodes.
- The load balancer rke2.lb_address provided in the extra_vars.yml must route port 9345 and 443 to the RKE2 server nodes.
- Playbooks handle the following known issues of RKE2:
    * https://docs.rke2.io/known_issues#firewalld-conflicts-with-default-networking by disabling the firewalld.
    * https://docs.rke2.io/known_issues#networkmanager ensuring that NetworkManager is configured to ignore CNI-managed interfaces if it's enabled.



## Usage

### 1. Build the Docker Image from the source

```bash
docker build -t suse-ai-node-ansible-runner -f Dockerfile.local .
```

If you want to use the newer buildx:
```bash
docker buildx build -t suse-ai-node-ansible-runner -f Dockerfile.local --load .
```

### 2. Create inventory.ini file 
```bash
cp inventory.ini.example inventory.ini
```
Update the ansible host and user entries in inventory.ini

### 3. Create extra_vars.yml 
```bash
cp extra_vars.yml.example extra_vars.yml
```
Configure entries in extra_vars.yml accordingly.

### 4. Run the site.yml playbook

At a high level, this playbook verifies that the target hosts are supported systems and registers them with the SCC if they are not already registered. It installs required packages and, when enabled, the NVIDIA drivers. NVIDIA G06 drivers are installed on servers with NVIDIA GPUs and are supported on Turing and newer architectures. Finally, the playbook reboots the target hosts and then run some checks and installs rke2 servers, rke2 agents, rancher and gpu-operator.

```bash
docker run --rm \
  -v ~/.ssh/id_rsa:/root/.ssh/id_rsa:ro \
  -v ./inventory.ini:/workspace/inventory.ini \
  -v ./extra_vars.yml:/workspace/extra_vars.yml \
  suse-ai-node-ansible-runner \
  ansible-playbook -i inventory.ini playbooks/site.yml -e "@extra_vars.yml"
```

If your target ansible_host is a *localhost*:

```bash
docker run --rm \
  --network host \
  -v ~/.ssh/id_rsa:/root/.ssh/id_rsa:ro \
  -v ./inventory.ini:/workspace/inventory.ini \
  -v ./extra_vars.yml:/workspace/extra_vars.yml \
  suse-ai-node-ansible-runner \
  ansible-playbook -i inventory.ini playbooks/stage1.yml -e "@extra_vars.yml"

docker run --rm \
  -v ~/.ssh/id_rsa:/root/.ssh/id_rsa:ro \
  -v ./inventory.ini:/workspace/inventory.ini \
  -v ./extra_vars.yml:/workspace/extra_vars.yml \
  suse-ai-node-ansible-runner \
  ansible-playbook -i inventory.ini playbooks/stage2.yml -e "@extra_vars.yml"
```
Note: NVIDIA drivers are not installed when localhost is the target. Recommended for localhost is to manually install the drivers.

### 6. Troubleshooting

#### 6a. Failed to connect to the host via ssh

confirm key permissions (~/.ssh 700, private key 600).

verify public key is in ~/.ssh/authorized_keys of the remote user.

run ssh -v user@host to debug connection/auth issues.


### 7. Additional information

#### 7a. Rancher Prime
By default, we use the community helm chart to deploy rancher. In order to use Rancher Prime, please add the following entries in the *rancher:* section of extra_vars.yml:
```yaml
use_prime: true
prime_helm_repo_url: "<rancher-prime-helm-chart-repo-url>"
```
To learn more about the Rancher Prime Helm chart repository URL, see the [Prime-only documentation](https://scc.suse.com/rancher-docs/rancherprime/latest/en/reference-guide.html#chart-repo-url) using your [SUSE Customer Center (SCC)](https://scc.suse.com/) credentials to log in.


### 8. Validations

Successfully tested against following target host OS versions:

| Arch                   | Distro                 | Version              | Succesfully Validated (1.0.0 release)             |
| ---------------------  | ---------------------- | -------------------- | --------------------------------------------------|
| x86_64                 | sle-micro              | 6.0                  |  yes                                              |
| x86_64                 | sle-micro              | 6.1                  |  yes                                              |
| arm64                  | sle-micro              | 6.0                  |  yes                                              |
| arm64                  | sle-micro              | 6.1                  |  yes                                              |
| x86_64                 | sles                   | 15-sp6               |  yes                                              |
| arm64                  | sles                   | 15-sp6               |  yes                                              |
| x86_64                 | sles                   | 15-sp7               |  yes                                              |
| arm64                  | sles                   | 15-sp7               |  yes                                              |
| x86_64                 | leap                   | 15.6                 |  yes                                              |
