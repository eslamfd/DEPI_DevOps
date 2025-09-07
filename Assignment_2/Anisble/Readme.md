
---

# Ansible Playbook: Docker Installation & Image Build

## Overview

This playbook ensures Docker is installed on target machines (Linux, Windows and CentOS), checks system requirements, and builds a Docker image from a local `Dockerfile`.

It covers:

* Hardware checks (RAM & CPU)
* Docker installation on **Ubuntu/Debian**, **CentOS/RedHat**, and **Windows**
* Building a test Docker image

---

## Project Structure

```
.
├── ansible.cfg        # Ansible configuration
├── hosts              # Inventory file (list of target hosts)
├── playbook.yml       # Main Ansible playbook
├── Dockerfile         # Dockerfile to build a test image
```

---

## Requirements

* Ansible installed on the control machine
* SSH access to target hosts (configured in `hosts`)
* Minimum system specs for targets:

  * **2 GB RAM**
  * **2 CPU cores**

---

## Usage

### 1. Update Inventory

Edit `hosts` with your servers:

```
[all]
server1 ansible_host=192.168.1.10
server2 ansible_host=192.168.1.11
```

### 2. Run Playbook

```bash
ansible-playbook -i hosts playbook.yml
```

### 3. Verify Results

```bash
docker --version
docker images
```

You should see Docker installed and an image named `my_test_image`.

---

## Playbook Modules Explained

Here’s what’s being used and why:

* **[command](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/command_module.html)**
  Runs commands like `docker --version` or `yum-config-manager`.

* **[debug](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/debug_module.html)**
  Prints useful info (like Docker version) during execution.

* **[assert](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/assert_module.html)**
  Enforces minimum hardware requirements before proceeding.

* **[apt](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html)**
  Manages packages on Debian/Ubuntu.

* **[apt\_key](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_key_module.html)**
  Adds Docker’s GPG key.

* **[apt\_repository](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_repository_module.html)**
  Configures Docker’s official repository.

* **[yum](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/yum_module.html)**
  Installs Docker on CentOS/RedHat.

* **[service](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/service_module.html)**
  Enables and starts Docker on RedHat-based systems.

* **[win\_chocolatey](https://docs.ansible.com/ansible/latest/collections/chocolatey/chocolatey/win_chocolatey_module.html)**
  Installs Docker Desktop on Windows via Chocolatey.

* **[community.docker.docker\_image](https://docs.ansible.com/ansible/latest/collections/community/docker/docker_image_module.html)**
  Builds the Docker image from the specified Dockerfile.

---

## Ansible Configuration Explained (`ansible.cfg`)

* **inventory = ./hosts**
  Default inventory file location.

* **remote\_user = ansible**
  Default SSH user for remote connections.

* **host\_key\_checking = False**
  Disables SSH host key verification (fewer prompts, but less secure).

* **retry\_files\_enabled = False**
  Disables `.retry` file creation.

* **interpreter\_python = auto**
  Lets Ansible auto-detect Python interpreter.

* **timeout = 30**
  SSH timeout in seconds.

* **forks = 10**
  Max number of parallel tasks.

* **stdout\_callback = yaml**
  Human-readable YAML output instead of JSON.

* **deprecation\_warnings = False**
  Suppresses noisy warnings.

* **become = True**
  Enables privilege escalation (sudo).

* **ssh\_args = -o ControlMaster=auto -o ControlPersist=60s**
  Reuses SSH connections for speed.

* **pipelining = True**
  Speeds up SSH execution by reducing overhead.

---

## Useful Links

* [Ansible Docs](https://docs.ansible.com/)
* [Playbook Best Practices](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_best_practices.html)
* [Ansible Configuration File Guide](https://docs.ansible.com/ansible/latest/reference_appendices/config.html)
* [Community Docker Collection](https://docs.ansible.com/ansible/latest/collections/community/docker/)

---
