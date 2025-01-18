# RKE2 Ansible Role

This Ansible role automates the installation of **RKE2** (Rancher Kubernetes Engine 2) on provisioned nodes and can optionally install additional tools such as Longhorn, ArgoCD, Autoscaler, and Monitoring.

## Requirements

- Ansible 2.9 or higher
- Ubuntu 20.04 or other supported Linux distributions on the nodes
- SSH access to the nodes
- Set the required variables in your `inventory.ini` or through the `vars` section in your playbook

## Role Variables

This section outlines the variables available for configuration in this role.

### RKE2 Installation

- **uninstall_rke2**  
  Set to `true` to uninstall RKE2 from the nodes. Default is `false`.

  ```yaml
  uninstall_rke2: false
  ```

- **uninstall_rke2_on_worker**  
  Set to `true` to uninstall RKE2 from worker nodes only. Default is `false`.

  ```yaml
  uninstall_rke2_on_worker: false
  ```

- **rke2_token**  
  Token used to join additional nodes to the RKE2 cluster. This can be generated during the initial setup of the RKE2 server node.

  ```yaml
  rke2_token: "your-token-here"
  ```

- **rke2_version**  
  The version of RKE2 to install. Default is the latest stable version.

  ```yaml
  rke2_version: "v1.22.3+rke2r1"
  ```

- **rke2_server_ip**  
  The IP address of the RKE2 server node. Required if configuring the role for master/slave nodes.

  ```yaml
  rke2_server_ip: "192.168.1.100"
  ```

### Optional Add-Ons

- **install_longhorn**  
  Set to `true` to install Longhorn for persistent storage in your RKE2 cluster. Default is `false`.

  ```yaml
  install_longhorn: false
  ```

- **install_monitoring**  
  Set to `true` to install monitoring tools (e.g., Prometheus and Grafana) in the cluster. Default is `false`.

  ```yaml
  install_monitoring: false
  ```

- **monitoring_slack_url**  
  URL for Slack webhook to send monitoring alerts to a Slack channel.

  ```yaml
  monitoring_slack_url: "https://hooks.slack.com/services/..."
  ```

- **monitoring_slack_channel**  
  The Slack channel to receive monitoring alerts.

  ```yaml
  monitoring_slack_channel: "#monitoring-alerts"
  ```

### ArgoCD Installation

- **install_argocd**  
  Set to `true` to install ArgoCD for continuous delivery in your cluster. Default is `false`.

  ```yaml
  install_argocd: false
  ```

- **argocd_domain**  
  The domain name for accessing the ArgoCD web interface. This is required if you plan to use ArgoCD.

  ```yaml
  argocd_domain: "argocd.example.com"
  ```

- **install_argocd_bootstraper**  
  Set to `true` to install ArgoCD bootstrapping tools. Default is `false`.

  ```yaml
  install_argocd_bootstraper: false
  ```

- **argocd_bootstrap_repo_url**  
  URL of the Git repository for ArgoCD bootstrapping. Set this if using ArgoCD with a repository for configuration management.

  ```yaml
  argocd_bootstrap_repo_url: "https://github.com/your/repo"
  ```

- **argocd_bootstrap_repo_path**  
  Path within the repository for the ArgoCD bootstrap configuration. Default is `bootstrap`.

  ```yaml
  argocd_bootstrap_repo_path: "bootstrap"
  ```

- **argocd_bootstrap_repo_branch**  
  Branch of the Git repository to use for ArgoCD bootstrapping. Default is `HEAD`.

  ```yaml
  argocd_bootstrap_repo_branch: "HEAD"
  ```

- **argocd_bootstrap_repo_username**  
  Username for accessing the ArgoCD bootstrap repository (if private). Leave empty for public repositories.

  ```yaml
  argocd_bootstrap_repo_username: "your-username"
  ```

- **argocd_bootstrap_repo_access_token**  
  Access token for the ArgoCD bootstrap repository (if private). Leave empty for public repositories.

  ```yaml
  argocd_bootstrap_repo_access_token: "your-token"
  ```

### Autoscaler Installation

- **install_autoscaller**  
  Set to `true` to install the Kubernetes Autoscaler. Default is `false`.

  ```yaml
  install_autoscaller: false
  ```

## Example Playbook

```yaml
---
- hosts: all
  become: true
  vars:
    rke2_token: "your-token-here"
    rke2_version: "v1.22.3+rke2r1"
    uninstall_rke2: false
    install_longhorn: true
    install_monitoring: true
    install_argocd: true
    argocd_domain: "argocd.example.com"
    install_autoscaller: true
  roles:
    - rke2-ansible
```

## Usage

### Clone the Repository

First, clone this repository to your local machine:

```bash
git clone https://github.com/ali3bdalla/rke2-ansible.git
cd rke2-ansible
```

## sample of inventory.yaml

```
cluster:
  children:
    - controller
    - worker
controller:
  hosts:
    controller_1:
      ansible_host: 10.100.70.101
      ansible_user: appdepl
    controller_2:
      ansible_host: 10.100.70.102
      ansible_user: appdepl
    controller_3:
      ansible_host: 10.100.70.103
      ansible_user: appdepl
lightweight:
  hosts:
    rke2-worker-11:
      ansible_host: 10.100.70.110
      ansible_user: appdepl
    rke2-worker-12:
      ansible_host: 10.100.70.111
      ansible_user: appdepl
    rke2-worker-13:
      ansible_host: 10.100.70.112
      ansible_user: appdepl
statefull:
  hosts:
    rke2-worker-21:
      ansible_host: 10.100.70.140
      ansible_user: appdepl
    rke2-worker-22:
      ansible_host: 10.100.70.141
      ansible_user: appdepl
    rke2-worker-23:
      ansible_host: 10.100.70.142
      ansible_user: appdepl
    rke2-worker-31:
      ansible_host: 10.100.70.160
      ansible_user: appdepl
    rke2-worker-32:
      ansible_host: 10.100.70.161
      ansible_user: appdepl
worker:
  children:
    - lightweight
    - statefull
```

### sample playbook.yml

```
- name: "Generate RKE2 token and store it in file"
  hosts: localhost # You can run this locally on your controller machine
  tasks:
    - name: "Check if .rke-config/token exists"
      stat:
        path: ".rke-config/token"
      register: token_file
    - name: "Generate RKE2 token if file does not exist"
      shell: "openssl rand -hex 32"
      register: rke2_token
      when: not token_file.stat.exists

    - name: "Create .rke-config directory if not exists"
      file:
        path: ".rke-config"
        state: directory
      when: not token_file.stat.exists

    - name: "Write token to file"
      copy:
        content: "{{ rke2_token.stdout }}"
        dest: ".rke-config/token"
      when: not token_file.stat.exists
- name: "Install RKE2 on controllers"
  hosts: controller
  become: true
  become_user: root
  vars:
    generated_token: "{{ lookup('file', '.rke-config/token') }}"
  roles:
    - role: ali3bdalla.rke2
      vars:
        rke2_token: "{{ generated_token }}"

- name: "Install RKE2 on workers"
  hosts: worker
  become: true
  become_user: root
  vars:
    generated_token: "{{ lookup('file', '.rke-config/token') }}"
  roles:
    - role: ali3bdalla.rke2
      vars:
        rke2_token: "{{ generated_token }}"
```

# Services can be enabled

### Nas NFS Storage class

see https://github.com/ali3bdalla/k8s-truenas

```
install_nfs: false
nfs_server_protocol: "http"
nfs_server_host: ""
nfs_server_port: ""
nfs_server_api_key: ""
nfs_server_volume_path: ""
nfs_server_snapshot_path: ""

```
