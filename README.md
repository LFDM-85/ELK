# Elastic Stack (ELK) on Docker

[![Elastic Stack version](https://img.shields.io/badge/Elastic%20Stack-9.2.2-00bfb3?style=flat&logo=elastic-stack)](https://www.elastic.co/blog/category/releases)
[![Build Status](https://github.com/deviantony/docker-elk/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/deviantony/docker-elk/actions/workflows/ci.yml?query=branch%3Amain)

## 📖 About This Project

This repository is based on the excellent [deviantony/docker-elk](https://github.com/deviantony/docker-elk) project, providing a complete implementation of the [Elastic stack][elk-stack] using Docker and Docker Compose.

The stack enables you to analyze any dataset using Elasticsearch's powerful search and aggregation capabilities combined with Kibana's visualization features.

### 🎯 Project Philosophy

This setup aims to make the Elastic stack as accessible as possible for development and testing. It is **not designed for production deployment**, but rather serves as a flexible template for exploration and experimentation.

The configuration is intentionally minimal and unopinionated, prioritizing clear documentation over complex automation. The initial setup requires no external dependencies and uses minimal scripting.

### 🧩 Core Components

Built on [official Docker images][elastic-docker] from Elastic:

- **[Elasticsearch](https://github.com/elastic/elasticsearch/tree/main/distribution/docker)** - Distributed search and analytics engine
- **[Logstash](https://github.com/elastic/logstash/tree/main/docker)** - Server-side data processing pipeline
- **[Kibana](https://github.com/elastic/kibana/tree/main/src/dev/build/tasks/os_packages/docker_generator)** - Data visualization and exploration tool

### 📦 Available Extensions

The stack includes several optional extensions in the `extensions/` directory:

- **Curator** - Elasticsearch index lifecycle management
- **Filebeat** - Lightweight shipper for log files
- **Fleet Server** - Centralized agent management
- **APM Server** - Application performance monitoring
- **Heartbeat** - Uptime monitoring
- **Metricbeat** - System and service metrics collector

### 🔐 Licensing Note

> [!IMPORTANT]
> [Platinum][subscriptions] features are enabled by default for a **30-day trial period**. After the trial expires, you automatically retain access to all free features included in the Open Basic license, with no data loss or manual intervention required. See [How to disable paid features](#how-to-disable-paid-features) to opt out of this behavior.

### 🌐 Alternative Stack Variants

- [`tls`](https://github.com/deviantony/docker-elk/tree/tls) - TLS encryption enabled in Elasticsearch, Kibana (optional), and Fleet

---

## 🚀 Quick Start

### Initial Setup

```sh
# 1. Clone the repository
git clone https://github.com/LFDM-85/ELK.git
cd ELK

# 2. Run the setup service (initializes users and roles)
docker compose up setup

# 3. (Optional but recommended) Generate Kibana encryption keys
docker compose up kibana-genkeys
# Copy the output to kibana/config/kibana.yml

# 4. Start the stack
docker compose up
```

After about a minute, access Kibana at <http://localhost:5601>

**Default credentials:**
- **User:** `elastic`
- **Password:** `changeme`

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://github.com/user-attachments/assets/6f67cbc0-ddee-44bf-8f4d-7fd2d70f5217">
  <img alt="Animated demo" src="https://github.com/user-attachments/assets/501a340a-e6df-4934-90a2-6152b462c14a">
</picture>

> [!NOTE]
> Run services in detached mode by adding the `-d` flag: `docker compose up -d`

---

## 📋 Table of Contents

1. [Requirements](#requirements)
2. [Installation & Setup](#installation--setup)
3. [Security Configuration](#security-configuration)
4. [Data Ingestion](#data-ingestion)
5. [Configuration](#configuration)
6. [Extensions](#extensions)
7. [Performance Tuning](#performance-tuning)
8. [Maintenance](#maintenance)
9. [Troubleshooting](#troubleshooting)

---

## 📦 Requirements

### System Requirements

- **Docker Engine** version 18.06.0 or newer
- **Docker Compose** version 2.0.0 or newer
- **RAM:** Minimum 1.5 GB (4 GB+ recommended)

> [!NOTE]
> On Linux, ensure your user has the [required permissions][linux-postinstall] to interact with the Docker daemon.

### Exposed Ports

By default, the stack exposes the following ports:

| Port  | Service                     |
|-------|-----------------------------|
| 5044  | Logstash Beats input        |
| 50000 | Logstash TCP input          |
| 9600  | Logstash monitoring API     |
| 9200  | Elasticsearch HTTP          |
| 9300  | Elasticsearch TCP transport |
| 5601  | Kibana                      |
| 8220  | Fleet Server (optional)     |
| 8200  | APM Server (optional)       |

### Platform-Specific Notes

#### Windows

If using legacy Hyper-V mode in Docker Desktop for Windows, enable [File Sharing][desktop-filesharing] for the `C:` drive.

#### macOS

Docker Desktop for Mac allows mounting files from `/Users/`, `/Volume/`, `/private/`, `/tmp`, and `/var/folders` only. Clone the repository in one of these locations or configure additional paths in [Docker Desktop settings][desktop-filesharing].

> [!WARNING]
> Elasticsearch's [bootstrap checks][bootstrap-checks] are disabled to simplify development setup. For production deployments, follow the [Important System Configuration][es-sys-config] guide.

---

## ⚙️ Installation & Setup

### Step 1: Initialize the Stack

> [!WARNING]
> Rebuild stack images with `docker compose build` after switching branches or updating component versions.

```sh
# Clone the repository
git clone https://github.com/LFDM-85/ELK.git
cd ELK

# Initialize Elasticsearch users and roles
docker compose up setup
```

### Step 2: Generate Kibana Encryption Keys (Recommended)

```sh
docker compose up kibana-genkeys
```

Copy the generated keys to `kibana/config/kibana.yml`.

### Step 3: Start the Stack

```sh
docker compose up
```

Wait approximately one minute for Kibana to initialize, then access the web UI at <http://localhost:5601>.

### Step 4: Initial Login

Use the default credentials:
- **Username:** `elastic`
- **Password:** `changeme`

> [!NOTE]
> The `elastic`, `logstash_internal`, and `kibana_system` users are initialized with passwords from the `.env` file (default: `changeme`). Change these passwords immediately for security.

---

## 🔒 Security Configuration

### Changing Default Passwords

> [!WARNING]
> The default `changeme` password is **insecure**. Reset all passwords immediately.

#### 1. Generate Secure Passwords

```sh
# Reset elastic user password
docker compose exec elasticsearch bin/elasticsearch-reset-password --batch --user elastic

# Reset logstash_internal user password
docker compose exec elasticsearch bin/elasticsearch-reset-password --batch --user logstash_internal

# Reset kibana_system user password
docker compose exec elasticsearch bin/elasticsearch-reset-password --batch --user kibana_system
```

**Save these generated passwords securely.**

#### 2. Update Configuration Files

Update the `.env` file with the new passwords:

```env
ELASTIC_PASSWORD=<generated_elastic_password>
LOGSTASH_INTERNAL_PASSWORD=<generated_logstash_password>
KIBANA_SYSTEM_PASSWORD=<generated_kibana_password>
```

These passwords are referenced in:
- `logstash/pipeline/logstash.conf` (logstash_internal)
- `kibana/config/kibana.yml` (kibana_system)

#### 3. Restart Services

```sh
docker compose up -d logstash kibana
```

### Resetting Passwords via API

If unable to use Kibana, reset passwords using the Elasticsearch API:

```sh
curl -XPOST -D- 'http://localhost:9200/_security/user/elastic/_password' \
    -H 'Content-Type: application/json' \
    -u elastic:<current_password> \
    -d '{"password" : "<new_password>"}'
```

> [!NOTE]
> Learn more about securing the Elastic Stack at [Secure the Elastic Stack][sec-cluster].
> 
> To disable authentication entirely, refer to [Security settings in Elasticsearch][es-security].

---

## 📊 Data Ingestion

### Using Logstash TCP Input

The default configuration accepts data on TCP port 50000. Send log data using `netcat`:

```sh
# Determine your nc version first
nc -h

# Then use the appropriate command:
cat /path/to/logfile.log | nc -q0 localhost 50000          # BSD
cat /path/to/logfile.log | nc -c localhost 50000           # GNU
cat /path/to/logfile.log | nc --send-only localhost 50000  # nmap
```

### Using Sample Data

Load sample datasets directly from Kibana:
1. Navigate to <http://localhost:5601>
2. Go to **Home** → **Add data**
3. Select a sample dataset (e.g., Sample web logs, Sample flight data)

---

## 🛠️ Configuration

> [!IMPORTANT]
> Configuration changes require service restarts. The stack does not support dynamic reloading.

### Elasticsearch Configuration

**Primary configuration file:** [`elasticsearch/config/elasticsearch.yml`][config-es]

**Environment variable overrides** (in `docker-compose.yml`):

```yml
elasticsearch:
  environment:
    network.host: _non_loopback_
    cluster.name: my-cluster
```

**Documentation:** [Install Elasticsearch with Docker][es-docker]

### Kibana Configuration

**Primary configuration file:** [`kibana/config/kibana.yml`][config-kbn]

**Environment variable overrides:**

```yml
kibana:
  environment:
    SERVER_NAME: kibana.example.org
```

**Documentation:** [Install Kibana with Docker][kbn-docker]

### Logstash Configuration

**Primary configuration file:** [`logstash/config/logstash.yml`][config-ls]

**Pipeline configuration:** [`logstash/pipeline/logstash.conf`](logstash/pipeline/logstash.conf)

**Environment variable overrides:**

```yml
logstash:
  environment:
    LOG_LEVEL: debug
```

**Documentation:** [Configuring Logstash for Docker][ls-docker]

### Disabling Paid Features

To revert to a basic license before trial expiration:

**Option 1: Via Kibana UI**
- Navigate to **Management** → **Stack Management** → **License Management**
- Click **Revert to Basic**

**Option 2: Via API**
```sh
curl -XPOST 'http://localhost:9200/_license/start_basic?acknowledge=true' \
    -u elastic:<password>
```

> [!NOTE]
> Changing `xpack.license.self_generated.type` from `trial` to `basic` in configuration only works **before initial setup**. After trial activation, use one of the methods above.

### Scaling Elasticsearch

See the Wiki guide: [Scaling out Elasticsearch](https://github.com/deviantony/docker-elk/wiki/Elasticsearch-cluster)

### Version Selection

**Current branch:** Tracks Elastic Stack 9.x

To use a different version:
1. Update `ELASTIC_VERSION` in the [`.env`](.env) file
2. Rebuild images: `docker compose build`

**Available version branches:**
- [`release-8.x`](https://github.com/deviantony/docker-elk/tree/release-8.x) - 8.x series
- [`release-7.x`](https://github.com/deviantony/docker-elk/tree/release-7.x) - 7.x series (End-of-Life)
- [`release-6.x`](https://github.com/deviantony/docker-elk/tree/release-6.x) - 6.x series (End-of-Life)
- [`release-5.x`](https://github.com/deviantony/docker-elk/tree/release-5.x) - 5.x series (End-of-Life)

> [!IMPORTANT]
> Always review [official upgrade instructions][upgrade] before upgrading component versions.

---

## 🔌 Extensions

### Available Extensions

Several optional integrations are provided in the [`extensions/`](extensions/) directory:

- **Curator** - Index lifecycle management and curation
- **Filebeat** - Log file shipping
- **Fleet Server** - Elastic Agent management
- **APM Server** - Application performance monitoring
- **Heartbeat** - Uptime and availability monitoring
- **Metricbeat** - Infrastructure metrics collection

### Enabling Extensions

Each extension includes its own documentation in the respective subdirectory. Some may require configuration changes to core components.

### Adding Plugins

To add plugins to any component:

1. Add a `RUN` statement to the component's `Dockerfile`:
   ```dockerfile
   RUN logstash-plugin install logstash-filter-json
   ```

2. Update the service configuration with plugin-specific settings

3. Rebuild the image:
   ```sh
   docker compose build <service_name>
   ```

---

## ⚡ Performance Tuning

### JVM Memory Configuration

Control memory allocation using environment variables:

| Service       | Environment Variable |
|---------------|---------------------|
| Elasticsearch | `ES_JAVA_OPTS`      |
| Logstash      | `LS_JAVA_OPTS`      |

**Default allocations** (in `docker-compose.yml`):
- Elasticsearch: 4 GB heap
- Logstash: 256 MB heap

**Example - Increase Logstash memory:**

```yml
logstash:
  environment:
    LS_JAVA_OPTS: -Xms1g -Xmx1g
```

**Default behavior when not set:**
- Elasticsearch: [Automatically determined heap size][es-heap]
- Logstash: Fixed 1 GB heap

### JMX Remote Monitoring

Enable JMX for monitoring and management:

```yml
logstash:
  environment:
    LS_JAVA_OPTS: >-
      -Dcom.sun.management.jmxremote
      -Dcom.sun.management.jmxremote.ssl=false
      -Dcom.sun.management.jmxremote.authenticate=false
      -Dcom.sun.management.jmxremote.port=18080
      -Dcom.sun.management.jmxremote.rmi.port=18080
      -Djava.rmi.server.hostname=DOCKER_HOST_IP
      -Dcom.sun.management.jmxremote.local.only=false
```

Replace `DOCKER_HOST_IP` with your Docker host's IP address.

---

## 🧹 Maintenance

### Re-running Setup

To reinitialize users with passwords from `.env`:

```sh
docker compose up setup
```

**Example output:**
```
 ⠿ Container docker-elk-elasticsearch-1  Running
 ⠿ Container docker-elk-setup-1          Created
Attaching to docker-elk-setup-1
...
docker-elk-setup-1  | [+] User 'monitoring_internal'
docker-elk-setup-1  |    ⠿ User does not exist, creating
docker-elk-setup-1  | [+] User 'beats_system'
docker-elk-setup-1  |    ⠿ User exists, setting password
docker-elk-setup-1 exited with code 0
```

### Complete Cleanup

To stop all services and remove persisted data:

```sh
docker compose --profile=setup down -v
```

> [!WARNING]
> This command permanently deletes all Elasticsearch data stored in volumes.

---

## 🐛 Troubleshooting

### Common Issues

**Issue: "max virtual memory areas vm.max_map_count [65530] is too low"**
```sh
# Linux/macOS
sudo sysctl -w vm.max_map_count=262144

# Windows (WSL2)
wsl -d docker-desktop
sysctl -w vm.max_map_count=262144
```

**Issue: Services fail to start due to memory**
- Increase Docker memory allocation in Docker Desktop settings
- Reduce JVM heap sizes in `docker-compose.yml`

**Issue: Kibana shows "Unable to connect to Elasticsearch"**
- Check Elasticsearch logs: `docker compose logs elasticsearch`
- Verify passwords in `.env` match those set in Elasticsearch
- Wait 1-2 minutes for Elasticsearch to fully initialize

### Viewing Logs

```sh
# View all logs
docker compose logs

# Follow logs for specific service
docker compose logs -f elasticsearch

# View last 100 lines
docker compose logs --tail=100
```

---

## 🔗 Additional Resources

### Documentation Links

- [Elastic Stack Documentation](https://www.elastic.co/guide/index.html)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Original Repository Wiki](https://github.com/deviantony/docker-elk/wiki)

### Community Resources

- [External Applications Guide](https://github.com/deviantony/docker-elk/wiki/External-applications)
- [Popular Integrations](https://github.com/deviantony/docker-elk/wiki/Popular-integrations)

---

## 📄 License

This project follows the licensing of the [original docker-elk repository](https://github.com/deviantony/docker-elk).

The Elastic Stack components are licensed under the [Elastic License](https://www.elastic.co/licensing/elastic-license).

---

[elk-stack]: https://www.elastic.co/elastic-stack/
[elastic-docker]: https://www.docker.elastic.co/
[subscriptions]: https://www.elastic.co/subscriptions
[es-security]: https://www.elastic.co/docs/reference/elasticsearch/configuration-reference/security-settings
[license-settings]: https://www.elastic.co/docs/reference/elasticsearch/configuration-reference/license-settings
[license-mngmt]: https://www.elastic.co/docs/deploy-manage/license/manage-your-license-in-self-managed-cluster
[license-apis]: https://www.elastic.co/docs/api/doc/elasticsearch/group/endpoint-license

[docker-install]: https://docs.docker.com/get-started/get-docker/
[compose-install]: https://docs.docker.com/compose/install/
[linux-postinstall]: https://docs.docker.com/engine/install/linux-postinstall/
[desktop-filesharing]: https://docs.docker.com/desktop/settings-and-maintenance/settings/#file-sharing

[bootstrap-checks]: https://www.elastic.co/docs/deploy-manage/deploy/self-managed/bootstrap-checks
[es-sys-config]: https://www.elastic.co/docs/deploy-manage/deploy/self-managed/important-system-configuration
[es-heap]: https://www.elastic.co/docs/deploy-manage/deploy/self-managed/important-settings-configuration#heap-size-settings

[builtin-users]: https://www.elastic.co/docs/deploy-manage/users-roles/cluster-or-deployment-auth/built-in-users
[ls-monitoring]: https://www.elastic.co/docs/reference/logstash/monitoring-with-metricbeat
[sec-cluster]: https://www.elastic.co/docs/deploy-manage/security#cluster-or-deployment-security-features

[config-es]: ./elasticsearch/config/elasticsearch.yml
[config-kbn]: ./kibana/config/kibana.yml
[config-ls]: ./logstash/config/logstash.yml

[es-docker]: https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-elasticsearch-with-docker
[kbn-docker]: https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-kibana-with-docker
[ls-docker]: https://www.elastic.co/docs/reference/logstash/docker-config

[upgrade]: https://www.elastic.co/docs/deploy-manage/upgrade/deployment-or-cluster/self-managed