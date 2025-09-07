# Spring PetClinic Follow-up

---

### Added Features

* **Static Analysis:** SonarQube
* **Artifact Storage:** Nexus
* **Monitoring:** Prometheus & Grafana

---

## 3. SonarQube

1. Pull and run the **community edition** Docker image.

2. Access SonarQube: [http://localhost:9000](http://localhost:9000)

   * Default user: `admin`
   * Default password: `admin` (change on first login)

3. Workflow:

   * Create a new project
   * Configure global settings
   * Generate project token
   * Run analysis with Maven:

   ```bash
   mvn clean verify sonar:sonar \
     -Dsonar.projectKey=projectname \
     -Dsonar.host.url=http://localhost:9000 \
     -Dsonar.login=<token>
   ```


---

## 4. Nexus

### Quick Start

Run Nexus via Docker:

```bash
docker run -d -p 8081:8081 --name nexus sonatype/nexus3
```

### Docker Compose (seperate file)

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

### Push Images

```bash
docker login localhost:8083
docker tag petclinic:latest localhost:8083/repository/project/petclinic:latest
docker push localhost:8083/repository/project/petclinic:latest
```

---

## 5. Monitoring with Prometheus & Grafana

### **Prometheus service**

* **Purpose**: Time-series database and monitoring system.
* **Image**: Uses the official Prometheus image.
* **Port**: Exposes `9090` so you can open the Prometheus UI at `http://localhost:9090`.
* **Volume**: Mounts local `prometheus.yml` into `/etc/prometheus/prometheus.yml`. That file tells Prometheus what to scrape
* **Networks**: On the shared `backend` network so it can reach both app containers.
* **depends\_on**: Starts only after the Postgres and MySQL app containers are up, since those are what it needs to monitor.
* **Persistence**:

  * `prometheus_data` → `/prometheus` (ensures TSDB survives restarts).
* **Dependencies**: waits for database exporters to be available.

In short: Prometheus is pulling metrics from PetClinic apps and storing them in its own time-series database.

#### Postgres Exporter

* **Image**: `prometheuscommunity/postgres-exporter:latest`
* **Purpose**: Exposes PostgreSQL metrics for Prometheus.
* **Config**: Uses the DSN `postgresql://petclinic:petclinic@postgres:5432/petclinic?sslmode=disable`.

#### MySQL Exporter

* **Image**: `prom/mysqld-exporter:latest`
* **Purpose**: Exposes MySQL metrics for Prometheus.
* **Config**: Uses the DSN `petclinic:petclinic@(mysql:3306)/`.

---

### **Grafana service**

* **Purpose**: Visualization and dashboards for metrics.
* **Image**: Official Grafana image. `grafana/grafana:latest`
* **Port**: Exposes `3000` for Grafana’s web UI (`http://localhost:3000`).
* **Environment**: Sets up the default admin credentials and disables random user sign-ups.
* **Volumes**:

  * `grafana_data` →  → `/var/lib/grafana` : persists dashboards/config so you don’t lose them when the container restarts.
  * `provisioning/datasources` → automatically pre-configures Prometheus as a data source.
* **depends\_on**: Waits for Prometheus to be running before it starts, because it needs that data source.

In short: Grafana reads from Prometheus and builds dashboards with all the JVM/HTTP/DB metrics that Spring Boot exposes.

## Volumes

* `pg_data`: PostgreSQL data
* `mysql_data`: MySQL data
* `sonarqube_data`: SonarQube data
* `sonarqube_extensions`: SonarQube plugins/extensions
* `grafana_data`: Grafana state (dashboards, users, settings)
* `prometheus_data`: Prometheus TSDB storage

---

# Prometheus & Spring Boot Configuration

This repo contains two important configuration files:

## 1. `prometheus.yml`

Defines how Prometheus discovers and scrapes metrics:

* **Global settings**

  * Scrape interval: every `15s`
  * Rule evaluation interval: every `15s`

* **Jobs**

  * `spring-petclinic`: Scrapes the Spring Boot apps at `/actuator/prometheus` on ports `8080` (Postgres and MySQL flavors).
  * `prometheus`: Prometheus scrapes itself for internal health.
  * `postgres_exporter`: Collects PostgreSQL DB metrics at `9187`.
  * `mysqld_exporter`: Collects MySQL DB metrics at `9104`.

This file lives inside the **Prometheus container**, mounted at `/etc/prometheus/prometheus.yml`.

---

## 2. `application.yaml` (Spring Boot app)

Tells Spring Boot which Actuator endpoints to expose so Prometheus has something to scrape:

* Exposed endpoints: `/actuator/health`, `/actuator/info`, `/actuator/prometheus`, `/actuator/metrics`
* Explicitly enables the Prometheus endpoint.
* Groups all actuator routes under `/actuator/*`.
* Reminder: secure these endpoints in production (auth, TLS, firewall).

This file belongs in the **Spring PetClinic applications**.

---

## Big Picture

1. **Spring PetClinic** apps expose metrics via `/actuator/prometheus`.
2. **Database exporters** (`postgres_exporter`, `mysqld_exporter`) expose DB metrics on their ports.
3. **Prometheus** reads `prometheus.yml` and scrapes:

   * PetClinic app metrics
   * Database metrics
   * Its own status
4. **Grafana** queries Prometheus and visualizes everything as dashboards and alerts.
5. **SonarQube** runs separately for code quality analysis (not scraped by Prometheus, but part of the full stack).

So flow is: **App → Prometheus → Grafana → Charts**


---

## References

* *The Power of Spring Boot Actuator and Prometheus in Enhancing Service Monitoring* — Batuhan Orhon on Medium. A nice walkthrough explaining how Spring Boot, Actuator, and Prometheus play together, complete with `/actuator/prometheus` magic. ([Medium][1])

* *Monitoring Made Simple: Empowering Spring Boot Applications with Prometheus and Grafana* — Simform Engineering on Medium. Covers configuring Micrometer, Actuator, Prometheus, Grafana using Docker Compose—you know, the practical stuff. ([Medium][2])

* *Monitoring Spring Boot Applications Using Actuator Metrics* — Stackademic on Medium. Handy guide to Actuator endpoints and scraping metrics via Prometheus. ([Stackademic][3])

* *Set Up Prometheus and Spring Boot Metrics* — Official Spring Boot documentation. Describes the `/actuator/prometheus` endpoint and how to expose it properly. ([docs.spring.io][4])

[1]: https://batuhanorhon.medium.com/spring-boot-actuator-and-prometheus-with-an-example-application-90c88f7cc923?utm_source=chatgpt.com "The Power of Spring Boot Actuator and Prometheus in ..."
[2]: https://medium.com/simform-engineering/revolutionize-monitoring-empowering-spring-boot-applications-with-prometheus-and-grafana-e99c5c7248cf?utm_source=chatgpt.com "Set Up Prometheus and Grafana for Spring Boot Monitoring"
[3]: https://blog.stackademic.com/monitoring-spring-boot-applications-using-actuator-metrics-4e5db6fa2d4f?utm_source=chatgpt.com "Monitoring Spring Boot Applications Using Actuator Metrics"
[4]: https://docs.spring.io/spring-boot/api/rest/actuator/prometheus.html?utm_source=chatgpt.com "Prometheus (prometheus) :: Spring Boot"

* *Setting Up a Complete Devops Pipeline with Jenkins, Kubernetes, SonarQube, Nexus, Trivy, Prometheus & Grafana with mail notification* — An end-to-end guide covering SonarQube, Nexus, and monitoring with Prometheus and Grafana. ([Medium][1])

* *DevOps Project : Building a Full-Stack Board Game* — A DevOps playground that includes SonarQube, Nexus, Prometheus, and Grafana monitoring as part of a full CI/CD pipeline. ([Medium][2])

* *Code Quality Observability with Sonarqube API and Grafana Stack in Kubernetes* — Shows how to pull SonarQube metrics into Grafana dashboards, using Kubernetes and APIs. ([Medium][3])

* **SonarQube Official Doc – Setting up monitoring with Prometheus** — Technical documentation explaining how SonarQube integrates with Prometheus via a PodMonitor in Kubernetes. ([SonarQube Documentation][4])

* **Nexus Monitoring with Prometheus** — Nexus Repository can expose Prometheus-formatted metrics at `/service/metrics/prometheus`. Includes a sample `prometheus.yml` snippet. ([Sonatype Help][5])

* *Installing a Grafana Dashboard for Nexus* — Practical steps to enable Prometheus metrics in Nexus and install a Grafana dashboard to monitor it. ([devsecblueprint.com][6])

* **Grafana Official Docs – Get started with Grafana and Prometheus** — The definitive guide to connecting Prometheus as a data source and building dashboards in Grafana. ([Grafana Labs][7])

* *Grafana support for Prometheus* — Solid background on how Grafana integrates with Prometheus for visualization and alerting. ([Prometheus][8])

* *Monitoring FastAPI with Grafana + Prometheus: A 5-Minute Guide* — Quick Medium tutorial on wiring Prometheus and Grafana together for application monitoring (concepts apply beyond FastAPI). ([medium.com](https://levelup.gitconnected.com/monitoring-fastapi-with-grafana-prometheus-a-5-minute-guide-658280c7f358?utm_source=chatgpt.com))

---

[1]: https://medium.com/%40robin5002234/setting-up-a-complete-devops-pipeline-with-jenkins-kubernetes-sonarqube-nexus-trivy-prometheus-a3d1df3cda3d?utm_source=chatgpt.com "Setting Up a Complete Devops Pipeline with Jenkins ..."
[2]: https://medium.com/%40abrahimcse/devops-project-building-a-full-stack-board-game-b48d9ab6187b?utm_source=chatgpt.com "DevOps Project : Building a Full-Stack Board Game"
[3]: https://medium.com/%40hergi2004/code-quality-observability-with-sonarqube-api-and-grafana-stack-in-kubernetes-502d8560b765?utm_source=chatgpt.com "Code Quality observability with Sonarqube API and ..."
[4]: https://docs.sonarsource.com/sonarqube-server/10.8/setup-and-upgrade/deploy-on-kubernetes/set-up-monitoring/prometheus/?utm_source=chatgpt.com "Setting up the monitoring with the Prometheus server"
[5]: https://help.sonatype.com/en/prometheus.html?utm_source=chatgpt.com "Prometheus"
[6]: https://devsecblueprint.com/projects/devsecops-home-lab/testing-results/install-nexus-grafana-dashboard?utm_source=chatgpt.com "Installing a Grafana Dashboard for Nexus | DSB"
[7]: https://grafana.com/docs/grafana/latest/getting-started/get-started-grafana-prometheus/?utm_source=chatgpt.com "Get started with Grafana and Prometheus"
[8]: https://prometheus.io/docs/visualization/grafana/?utm_source=chatgpt.com "Grafana support for Prometheus"

