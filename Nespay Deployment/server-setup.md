# Nespay

```table-of-contents
```

## Intro

This document is assuming all of Nespay services and dependencies are deployed into one single baremetal server. For multi server deployment scenario it is recommended to separate the node for hosting dependencies, and the services itself. 

The Nespay Product consist of multiple service.

| Service Name       | Purpose                                                   | Access                |
| ------------------ | --------------------------------------------------------- | --------------------- |
| backend-service    | Serving REST API for Merchant and Admin functionality.    | `api.domain.tld`      |
| webhook-service    | Receiving Webhook from third-party services.              | `webhook.domain.tld`  |
| job-service        | Processing various background jobs, and async tasks.      | `job.domain.tld`      |
| admin-dashboard    | Frontend service to perform various Administrative tasks. | `desk.domain.tld`     |

# Deployment

The deployment will be using containerized approach, the application will run inside container to provide better isolation and separation for each deployed services.
## Operating System
- Recommendation: RHEL 9  Derivatives (Almalinux / Rocky Linux / Centos Stream 9) 
	- Current OS being used on Nespay Production: `Almalinux 9.6` on `x86_64` architecture

### Disk Partitioning

It is recommended to use 2 device for storage with following design:
- OS: for hosting Operating System and System-wide packages
- Data: for storing anything related with application, use **bind mount** for following mount path:
	- `/var/lib/docker` - storing docker related data: containers, volume, etc
	- `/var/lib/grafana` - storing grafana data: datasources, provisioning scripts, temp files, sessions, etc
	- `/var/lib/loki` - storing Loki's collected data
	- `/var/lib/pgsql` - storing PostgreSQL data

Disk Partitioning example:

```sh
NAME          MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
nvme1n1     259:1    0   100G  0 disk
└─nvme1n1p1 259:6    0   100G  0 part /mnt/data
																			/var/lib/docker
																			/var/lib/grafana
																			/var/lib/loki
																			/var/lib/pgsql
nvme0n1       259:1    0  20G  0 disk
├─nvme0n1p1   259:2    0  20G  0 part /
└─nvme0n1p128 259:3    0  10M  0 part /boot/efi
```

The partition strategy and mount paths is encouraged to be adjusted based on disk configuration on the machine itself.

## Dependencies

Below is a list of required service and dependencies that need to be installed / prepared to ensure Nespay Product can run and functions properly:

- Amazon Web Services:
	- AWS Cognito.
	- AWS SES: for email service.
	- AWS Nitro Enclave: Trusted Execution Environment for protecting Private Keys.
	- AWS KMS: Key Management Service for.
	- AWS Secret Manager: storing sensitive data.
- Nginx - v1.20, or newer
- PostgreSQL - v17.x
- Redis - v6.x
- Grafana Loki - latest
- Grafana Alloy - latest
- Grafana Dashboard - latest
- CryptoApi - https://cryptoapis.io/ - for Transaction monitoring.
- Privy.io - https://www.privy.io/ -  MPC Wallet Provider.
- Rain - https://www.rain.xyz/ - Card Provider.


## Installation

This part will guide you to install and configure the dependencies and applications.

### Dependencies

To run Nespay Product, some dependencies must be provided before we can run the product.

#### PostgreSQL 17

Make sure to disable Almalinux's built-in PostgreSQL module with this command before installing PostgreSQL:

```sh
sudo dnf module disable postgresql
```

To instal PostgreSQL 17, follows guide on this link: https://www.postgresql.org/download/linux/redhat/

Fill the form with following info:
- Select version: `17`
- Select platform: `Red Hat Enterprise, Rocky, AlmaLinux or Oracle version 9`
- Select architecture: `x86_64`

Proceed the instalation as stated, ensure 

For production level, it is recommended to use PostgreSQL cluster instead of single instance.

##### PostgreSQL Roles

Multiple roles are required so each service only has access to its own schemas and data, preventing cross-service interference, and limiting the blast radius of mistakes or compromises.

Role that need to be created:
- `nespay_user` - to be used by Nespay's services
- `grafana` - to be used by Grafana instance, this one can be skipped when Grafana dashboard is deployed using SQLite as database storage.

You may add other roles as needed, such as for monitoring, and database administration process. When possible, always avoid using `postgres` user directly.

#### Redis

Nespay support multiple kind of Redis deployment scenario, from Single Instance, Sentinel, or Cluster mode.

Follow guides below to install and configure single-instance Redis Server

DNF Repo location: `/etc/yum.repos.d/redis.repo`

```ini
[Redis]
name=Redis
baseurl=http://packages.redis.io/rpm/rockylinux9
enabled=1
gpgcheck=1
```

Install redis

```sh
curl -fsSL https://packages.redis.io/gpg > /tmp/redis.key
sudo rpm --import /tmp/redis.key
sudo dnf install redis
rm /tmp/redis.key
```

Ensure you enable authentication on the redis instance, and note the password because we will use it later. The simples way to enable authentication is using `requirepass` attribute on the `redis.conf` file.

Once configuration is finished, enable and start Redis instance using

```sh
sudo systemctl enable redis.service
sudo systemctl start redis.service
```

> reference: https://redis.io/docs/latest/operate/oss_and_stack/install/install-stack/rpm/

#### Grafana Stacks

Grafana stacks on Nespay ecosystem is used to store some logs from the backend services, the services is directly writing the logs using `alloy` as data receiver, then `alloy` will forward it to `loki` instance, which later can be viewed from `grafana-server`.

First, configure dnf repo for grafana package repository, create a new file at `/etc/yum.repos.d/grafana.repo` with following contents:

```ini
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
```

Then proceed to install `grafana`, `loki`, and `alloy` with following command:

```sh
sudo dnf install \
	grafana \
	loki \
	alloy
```

> reference: https://rpm.grafana.com/

##### Configuring Loki

Before configuring Loki, first create these following directories first:

```sh
mkdir -p \
  /var/lib/loki/chunks \
  /var/lib/loki/compactor \
  /var/lib/loki/rules \
  /var/lib/loki/rules-temp \
  /var/lib/loki/tsdb-cache \
  /var/lib/loki/tsdb-index \
  /var/lib/loki/wal
```

Configure loki using profided configs located in `./configs/loki/config.yml` as reference, the configuration file is located at `/etc/loki/config.yml`

Use command below to validate Loki config, when no output displayed on the screen that means all configuration items are valid.

```sh
/usr/bin/loki \
	-verify-config \
	-config.file /etc/loki/config.yml
```

Proceed to enable and start Loki service using command below:

```sh
sudo systemctl enable loki.service
sudo systemctl start loki.service
```

##### Configuring Grafana

Depends on your chosen Grafana storage option: you don’t need to modify the config file if you use the default SQLite, but switching to PostgreSQL requires updating the database settings. 

If you choose PostgreSQL, edit `/etc/grafana/grafana.ini` in the `[database]` section to match your connection details. See [this reference](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/#database) for configuration guidance.

Then proceed to enable and start Grafana service:

```sh
sudo systemctl enable grafana-server.service
sudo systemctl start grafana-server.service
```

Use command below to reset the Grafana Administrator password:

```sh
grafana-cli admin reset-admin-password <new-password>
```

##### Configuring Alloy

Alloy acts as the telemetry collector that receives logs from Nespay backend services via OTLP and forwards them to Loki.

The configuration file is located at `/etc/alloy/config.alloy`. Use the provided configuration from `./configs/alloy/config.alloy` as reference.

Key configuration points:
- **Docker Log Collection**: Automatically discovers and collects logs from Docker containers
- **Log Filtering**: Drops health check and debug logs to reduce noise
- **Log Processing**: Keeps ERROR, WARN, INFO, FATAL level logs
- **Loki Integration**: Forwards processed logs to Loki instance
- **Labels**: Tags logs with cluster and environment information

Example configuration structure:
```alloy
logging {
  level  = "info"
  format = "logfmt"
}

// Docker container discovery
discovery.docker "containers" {
  host = "unix:///var/run/docker.sock"
  refresh_interval = "5s"
}

// Collect and process Docker logs
loki.source.docker "docker_logs" {
  host       = "unix:///var/run/docker.sock"
  targets    = discovery.docker.containers.targets
  forward_to = [loki.process.docker_logs.receiver]
}

// Write to Loki
loki.write "loki" {
  endpoint {
    url = "http://localhost:3100/loki/api/v1/push"
  }
  external_labels = {
    cluster = "nespay"
    env     = "production"
  }
}
```

**Important**: Adjust the Loki URL if your Loki instance is on a different host.

Validate the configuration:
```sh
alloy fmt /etc/alloy/config.alloy
```

Enable and start Alloy service:
```sh
sudo systemctl enable alloy.service
sudo systemctl start alloy.service
```

Verify Alloy is running:
```sh
sudo systemctl status alloy.service
```

#### Nginx

Nginx serves as the reverse proxy for all Nespay services, providing SSL termination, load balancing, and request routing.

Install Nginx from EPEL repository:
```sh
sudo dnf install epel-release
sudo dnf install nginx
```

Basic reverse proxy configuration will be provided separately. Place vhost configuration files in `/etc/nginx/conf.d/`.

Enable and start Nginx:
```sh
sudo systemctl enable nginx.service
sudo systemctl start nginx.service
```

Verify Nginx configuration:
```sh
sudo nginx -t
```

Reload Nginx after configuration changes:
```sh
sudo systemctl reload nginx.service
```

#### Docker

Docker is required to run Nespay backend services in containers.

Install Docker Engine on AlmaLinux 9:

Add Docker repository:
```sh
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

Install Docker packages:
```sh
sudo dnf install \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

Enable and start Docker:
```sh
sudo systemctl enable docker.service
sudo systemctl start docker.service
```

Add your user to docker group (optional, for non-root Docker access):
```sh
sudo usermod -aG docker $USER
newgrp docker
```

Verify Docker installation:
```sh
docker --version
docker compose version
```

Test Docker:
```sh
docker run hello-world
```

#### PostgreSQL Database Creation

After PostgreSQL installation and role creation, create the Nespay database and install required extensions.

Switch to postgres user:
```sh
sudo -u postgres psql
```

Create database and configure:
```sql
-- Create Nespay database
CREATE DATABASE nespay OWNER nespay_user;

-- Connect to nespay database
\c nespay

-- Install required extension for full-text search
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Grant all privileges
GRANT ALL PRIVILEGES ON DATABASE nespay TO nespay_user;
GRANT ALL PRIVILEGES ON SCHEMA public TO nespay_user;

-- Verify extension installation
\dx

-- Exit
\q
```

Test connection with nespay_user:
```sh
psql -U nespay_user -d nespay -h localhost
```

Update PostgreSQL authentication if needed (`/var/lib/pgsql/17/data/pg_hba.conf`):
```
# Allow nespay_user to connect from localhost
host    nespay    nespay_user    127.0.0.1/32    scram-sha-256
```

Reload PostgreSQL configuration:
```sh
sudo systemctl reload postgresql-17.service
```

### Security & Firewall

Configure firewall to allow required services while blocking unnecessary access.

**Firewall Configuration Tips:**
- Open only necessary ports (HTTP/HTTPS, PostgreSQL, Redis)
- Restrict PostgreSQL, and Redis to localhost or specific IPs
- Use firewalld for AlmaLinux 9
- Consider SSH port changes and fail2ban for SSH protection
- Enable SELinux in enforcing mode with proper policies

**Important Ports:**
- 80/443: HTTP/HTTPS (Nginx)
- 5432: PostgreSQL (restrict to localhost or backend containers)
- 6379: Redis (restrict to localhost)
- 3100: Loki (restrict to localhost)
- 4318: Alloy OTLP endpoint (restrict to localhost)

**General Security Recommendations:**
- Keep system packages updated regularly
- Use strong passwords for all services
- Enable SSL/TLS for all external services
- Implement proper backup strategies
- Monitor logs for suspicious activity
- Use AWS Secrets Manager or similar for sensitive credentials
- Implement rate limiting on Nginx
- Configure proper file permissions
- Disable unnecessary services
- Regular security audits

### Monitoring & Alerting

#### Adding Loki as Datasource in Grafana

Access Grafana web interface at `http://your-server:3000` (default credentials: admin/admin).

Add Loki datasource:
1. Navigate to **Configuration** → **Data Sources**
2. Click **Add data source**
3. Select **Loki**
4. Configure:
   - **Name**: Loki
   - **URL**: `http://localhost:3100`
   - **Access**: Server (default)
5. Click **Save & Test**

View logs in Grafana:
- Navigate to **Explore** menu
- Select **Loki** datasource from dropdown
- Use Log browser to filter by labels (container, job, level)
- Use LogQL queries to search logs (e.g., `{container="nespay-backend"} |= "error"`)

The logs from all Nespay services are automatically collected and available in the Explore view with proper filtering and search capabilities.

### Backup & Recovery

**Database Backup Recommendations:**

Regular database backups are critical for disaster recovery and data protection. Consider implementing:

- **Automated backup mechanism**: Set up automated daily backups using cron jobs or backup tools
- **Backup retention policy**: Define how long backups should be kept (e.g., 30 days)
- **Off-site storage**: Store backups in a different location or cloud storage
- **Backup verification**: Regularly test backup restoration to ensure backups are valid
- **Point-in-time recovery**: Consider PostgreSQL WAL archiving for transaction-level recovery
- **Backup monitoring**: Set up alerts for backup failures

**What to Backup:**
- PostgreSQL database
- Environment configurations (.env files)
- Application configurations
- SSL certificates
- Grafana dashboards and datasources

**Backup Frequency:**
- Database: Daily at minimum, more frequently for critical data
- Configurations: After any changes
- Full system: Weekly or monthly

### System Maintenance

Regular system maintenance ensures optimal performance, security, and reliability.

**Log Management:**
- Configure log rotation for all services to prevent disk space exhaustion
- Monitor log file sizes regularly
- Archive old logs to separate storage
- Consider centralized logging retention policies in Loki

**System Updates:**
- Schedule regular system package updates
- Test updates in staging environment first
- Maintain update changelog for audit purposes
- Subscribe to security advisories for AlmaLinux

**Service Health Monitoring:**
- Regularly check service status of all components
- Monitor system resources (CPU, memory, disk, network)
- Set up alerts for service failures
- Review error logs periodically

**Performance Monitoring:**
- Track application performance metrics
- Monitor database query performance
- Check Redis hit rates and memory usage

**Certificate Management:**
- Track SSL certificate expiration dates
- Automate certificate renewal where possible
- Test certificate renewal process before expiration
- Maintain certificate inventory

**Database Maintenance:**
- Regular VACUUM operations on PostgreSQL
- Monitor and optimize slow queries
- Review and update database indexes
- Monitor database size and growth trends

**Security Maintenance:**
- Review and update firewall rules
- Audit user access and permissions
- Review security logs for anomalies
- Update passwords and credentials periodically
- Apply security patches promptly

### Testing & Validation

After completing the installation, perform these validation steps to ensure everything is properly configured.

**1. Service Connectivity**
- Verify all systemd services are running:
  ```sh
  systemctl status postgresql-17
  systemctl status redis
  systemctl status loki
  systemctl status alloy
  systemctl status grafana-server
  systemctl status nginx
  systemctl status docker
  ```
- Check service ports are listening:
  ```sh
  ss -tulpn | grep -E '(5432|6379|5672|3100|4318|3000|80|443)'
  ```

**2. Database Connection**
- Test PostgreSQL connection with nespay_user
- Verify pg_trgm extension is installed
- Run test queries to ensure database is accessible

**3. Log Collection Validation**
- Start a test container and verify logs appear in Loki
- Check Alloy is collecting Docker container logs
- View logs in Grafana Explore to confirm end-to-end pipeline

**4. Backend Services Health**
- Deploy Nespay backend services via docker-compose
- Check health endpoints:
  - `http://localhost:8000/api/v1/health` (Main API)
  - `http://localhost:8001/api/v1/health` (Webhook)
  - `http://localhost:8002/api/v1/health` (Jobs)
- Verify services can connect to PostgreSQL, and Redis.

**5. Nginx Reverse Proxy**
- Test Nginx configuration syntax
- Verify SSL certificates are properly configured
- Access services through Nginx reverse proxy
- Check Nginx access and error logs for issues

Successful validation of all these points confirms the infrastructure is ready for Nespay deployment. Refer to `DOCKER_DEPLOYMENT.md` for application deployment steps.

