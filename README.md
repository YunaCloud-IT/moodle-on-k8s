# Moodle on Kubernetes (YunaCloud-IT)

A production-grade, highly available Moodle distribution for Kubernetes.

This repository contains both the Source Code for our hardened, non-root Moodle Docker image and the Helm Chart required to deploy it scalably.

## 🌟 Key Features

- Secure by Default: Runs as a non-root user (UID 1001) with a read-only root filesystem.

- Horizontal Scaling: Supports HPA (Auto-scaling) out of the box with ReadWriteMany storage support.

- PostgreSQL Optimized: Pre-configured with the pgsql driver for enterprise performance.

- Background Processing: dedicated CronJob resource (concurrency managed) for Moodle tasks.

- Production Ready: Includes Ingress upload tuning (client_max_body_size), Redis session support, and graceful shutdowns.

## 📋 Prerequisites

Before installing, ensure your cluster has:

- Kubernetes 1.23+

- Helm 3.8+

- Storage Class: A storage class capable of ReadWriteMany (RWX) access mode (e.g., NFS, AWS EFS, Google Filestore, Azure Files).

- Database: An external PostgreSQL database (Recommended) or a PostgreSQL operator installed in the cluster.

## 🚀 Quick Start (Installation)

### 1. Add the Helm Repository

We host our charts via GitHub Pages. Add the repo to your local Helm:

```bash
helm repo add yunacloud https://yunacloud-it.github.io/moodle-on-k8s/
helm repo update
```

### 2. Create Database Secret

For security, we do not put database passwords in the Helm values. Create a secret manually:

```bash
kubectl create secret generic moodle-db-credentials \
  --from-literal=postgres-user='moodleuser' \
  --from-literal=postgres-password='YOUR_STRONG_PASSWORD'
```

### 3. Install the Chart

Deploy Moodle with the release name moodle-prod:

```bash
helm install moodle-prod yunacloud/moodle \
  --set ingress.hostname=moodle.your-domain.com \
  --set externalDatabase.host=postgres.your-db-host.com
```

## 📦 Architecture

This chart deploys the following Kubernetes resources:

1. Deployment: Stateless PHP-Apache pods serving the web traffic.

2. CronJob: A singleton pod running admin/cli/cron.php every minute.

3. Service: Exposes the pods on port 80 (mapping to container port 8080).

4. Ingress: Routes external HTTPS traffic to the Service.

5. PVC: A shared ReadWriteMany volume for /var/www/moodledata.

6. ConfigMap: Generates the config.php dynamically from environment variables.

## 🛠️ Development & Building

If you want to modify the Docker image or the Chart source code:

### Building the Image
The image is built automatically via GitHub Actions on push to main.

```bash
docker build -t your-repo/moodle:custom -f docker/Dockerfile .
```

### Packaging the Chart

To create a new release of the chart:

1. Bump the version in charts/moodle/Chart.yaml.

2. Push to main.

3. The CI/CD pipeline will automatically package and release it to the Helm Repo.

## 🤝 Contributing

1. Fork the project

2. Create your feature branch (git checkout -b feature/AmazingFeature)

3. Commit your changes (git commit -m 'Add some AmazingFeature')

4. Push to the branch (git push origin feature/AmazingFeature)

5. Open a Pull Request

## 📄 License

Distributed under the MIT License. See LICENSE for more information.

## Prometheus Metrics

1. Log in as Admin in Moodle

2. Go to Site administration > Plugins > Local plugins > Prometheus metrics.

3. Enable the plugin.

4. Security Token:

    - The plugin requires a token to scrape metrics.

    - Copy the System Token shown on that settings page.

    - Alternative: You can whitelist the IP range of your Prometheus server to avoid using tokens, but in Kubernetes, IP ranges change. Using the token is safer.
