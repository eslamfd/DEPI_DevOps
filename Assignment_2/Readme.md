
---

# Assignment 02

This project covers the following tasks:

* Test with **SonarQube**
* Push images to **Nexus**
* Deploy application at `domain.com`
* Integrate **Prometheus** and **Grafana** for monitoring
* Write an **Ansible** playbook to install Docker and run a Dockerfile

---

## Progress

* **SonarQube**: Installed and functional. However, running tests against Spring PetClinic is producing errors.
* **Nexus**: Container is running, but login attempts return *401 Unauthorized*.
* **Deployment**: Still under investigation.
* **Prometheus & Grafana**: Docker Compose file created, containers are running successfully.
* **Ansible**: Playbook implemented. Some issues remain with the final task.

---

## SonarQube

1. Pulled the official **community** Docker image.

   > Note: `lts-community` image is not supported at runtime.

2. Run the container:

   * Default user: `admin`
   * Default password: `admin` (prompted to change on first login)

3. Steps:

   * Create a new project.
   * Configure global settings.
   * Generate a project token.
   * Run Maven with:

   ```bash
   mvn clean verify sonar:sonar \
     -Dsonar.projectKey=projectname \
     -Dsonar.host.url=http://localhost:9000 \
     -Dsonar.login=<token>
   ```

4. View results at [http://localhost:9000](http://localhost:9000).

---

## Nexus

**Official Docker Command**:

```bash
docker run -d -p 8081:8081 --name nexus sonatype/nexus3
```

**Docker Compose Setup**:

```yaml
version: "3.9"

services:
  nexus:
    image: sonatype/nexus3:latest
    container_name: nexus
    ports:
      - "8082:8081"   # Nexus UI
      - "8083:8083"   # Optional (raw repos)
    volumes:
      - nexus-data:/nexus-data

volumes:
  nexus-data:
```

**Image Push Process**:

```bash
# Log in
docker login localhost:8083

# Tag the image
docker tag petclinic:latest localhost:8083/petclinic:latest

# Push the image
docker push localhost:8083/petclinic:latest
```

> **Issue**: Login attempt results in `401 Unauthorized`.

---

## Prometheus & Grafana

**Docker Compose**:

```yaml
version: '3'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=200h'
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
```

**Reference Article**: [Monitoring FastAPI with Grafana + Prometheus](https://levelup.gitconnected.com/monitoring-fastapi-with-grafana-prometheus-a-5-minute-guide-658280c7f358)

---

## Ansible

The Ansible playbook automates Docker installation across multiple operating systems and builds a Docker image from a Dockerfile.

### Modules Used

* **command**: Run shell commands remotely.
* **debug**: Print output messages.
* **assert**: Validate system requirements (RAM/CPU).
* **apt / yum / win\_chocolatey**: Install packages depending on OS (Ubuntu, CentOS, Windows).
* **apt\_key / apt\_repository**: Add Docker’s GPG key and repository for apt.
* **service**: Manage Docker daemon state (start/enable).
* **community.docker.docker\_image**: Build images from Dockerfiles.

**Summary**:

* `command` and `debug` = diagnostic helpers.
* `assert` = system checks.
* Package managers (`apt`, `yum`, `win_chocolatey`) = install Docker.
* Repository modules = enable official Docker repos.
* `service` = ensure Docker runs.
* `community.docker.docker_image` = build project image.

---

# Ansible Configuration (`ansible.cfg`)

This configuration defines defaults, privilege escalation, and SSH connection behavior for Ansible runs.

---

## `[defaults]`

* **inventory = ./hosts**

  * Specifies the inventory file location (`./hosts`).

* **remote\_user = ansible**

  * Default user for connecting to managed hosts.

* **host\_key\_checking = False**

  * Disables host key verification.
  * Useful for lab/testing environments; not recommended in production.

* **retry\_files\_enabled = False**

  * Prevents creation of retry files (`*.retry`).

* **interpreter\_python = auto**

  * Automatically detects which Python interpreter to use on the remote host.

* **timeout = 30**

  * SSH connection timeout in seconds.

* **forks = 10**

  * Maximum number of parallel processes (default concurrency level).

* **gathering = smart**

  * Optimizes fact gathering (caches facts when possible).

* **stdout\_callback = yaml**

  * Formats playbook output in **YAML** style for better readability.

* **deprecation\_warnings = False**

  * Suppresses deprecation warnings during execution.

---

## `[privilege_escalation]`

* **become = True**

  * Enables privilege escalation (similar to `sudo`).

* **become\_method = sudo**

  * Uses `sudo` as the escalation method.

* **become\_ask\_pass = False**

  * Disables interactive password prompts (assumes key-based authentication).

---

## `[ssh_connection]`

* **ssh\_args = -o ControlMaster=auto -o ControlPersist=60s**

  * Enables SSH multiplexing for faster connections.
  * Reuses SSH sessions for 60 seconds before closing.

* **pipelining = True**

  * Reduces the number of SSH operations per task (improves performance).

---

This configuration provides:

* **Performance optimizations** (pipelining, forks, SSH multiplexing)
* **Cleaner output** (YAML formatting, no retry files, no deprecation spam)
* **Simplified privilege management** (sudo enabled by default)

---


