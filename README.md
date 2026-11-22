# ERPNext Xinyu Ops
This repository contains the configuration files for deploying ERPNext with custom settings for Xinyu operations.

# ECS Setup

## Install Docker

## Install Docker Compose

## Install MariaDB

# Build Docker Image

```bash
docker build \
  --build-arg=FRAPPE_BRANCH=version-16 \
  --secret=id=apps_json,src=./images/apps.json \
  --file=images/Containerfile \ 
  --tag=erpnext-xinyu:0.1 .
```
We can specify the `FRAPPE_PATH` and `FRAPPE_BRANCH` build arguments if necessary.

# Start Services

```bash
docker compose up -d 
```

# Create sites and install Apps

```bash
docker compose exec backend \
  bench new-site erp \
  --mariadb-user-host-login-scope=% \
  --db-root-password 'db password' \
  --admin-password 'admin password in UI' \
  --install-app erpnext \
  --install-app erpnext_china \
  --install-app erpnext_xinyu
```
