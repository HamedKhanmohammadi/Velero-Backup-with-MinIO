
# Velero + MinIO + Kubernetes CSI Integration Guide

This guide walks you through setting up MinIO for Velero backups and integrating Velero with Kubernetes using CSI.

---

## 📦 Backup and Copy to Remote Storage (Simple MinIO)

> Source: https://github.com/farshadnick/kuber-in-action/tree/main/backup/velero

---

## 🖥️ Install MinIO Single-Node Instance

⚙️ **Run all these steps on the MinIO server** (can be the same as your Kubernetes master or a separate machine).

### Step 1: Create and Edit the `docker-compose.yml` File

Create the file:

```bash
vim docker-compose.yml
```

Paste the following content into the file:

```yaml
services:
  minio:
    image: quay.io/minio/minio
    container_name: minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: admin
      MINIO_ROOT_PASSWORD: password
    ports:
      - "9000:9000"  # S3 API
      - "9001:9001"  # Web UI
    volumes:
      - minio-data:/data

volumes:
  minio-data:
```

To save and exit Vim:
1. Press `Esc`
2. Type `:wq` and hit Enter

### Step 2: Start the Services with Docker Compose

```bash
docker compose up -d
```

### Step 3: Verify the Containers

```bash
docker ps
docker compose logs -f
```

### Step 4: Access MinIO

Open your browser and go to:

```
http://192.168.55.27:9001
```

> This is the MinIO **Web Console** (port `9001`).

Login credentials:

- **Username:** `admin`
- **Password:** `password`

> To change the login credentials, modify `MINIO_ROOT_USER` and `MINIO_ROOT_PASSWORD` in `docker-compose.yml`.

---

## 🧭 How to Create a MinIO Bucket for Velero Backups

### 1. Access the MinIO Web Console

Go to:

```
http://192.168.55.27:9001
```

### 2. Log In to MinIO

Use:

- **Username:** `admin`
- **Password:** `password`

> Use your custom credentials if changed.

### 3. Create the Backup Bucket

- Click the `+ Create Bucket` button
- Enter the bucket name exactly as:

```
velero-backups
```

- Leave all options as default
- Click `Create`

> This bucket will be used by Velero to store backup data.

### ✅ Optional: Confirm Access from Velero

```bash
kubectl get backupstoragelocations -n velero
```

Output should include:

```
NAME      PHASE       LAST VALIDATED   AGE     DEFAULT
default   Available   <time>           <age>   true
```

---

## ☁️ Install Velero

⚙️ **Run these steps on your Kubernetes control node (master).**

### Step 1: Create MinIO Bucket Credentials File

```bash
cat <<EOF > credentials-velero
[default]
aws_access_key_id=admin
aws_secret_access_key=password
EOF
```

### Step 2: Install Velero Binary

```bash
cd /opt/
wget https://github.com/vmware-tanzu/velero/releases/download/v1.16.1/velero-v1.16.1-linux-amd64.tar.gz
tar xvf velero-v1.16.1-linux-amd64.tar.gz
cd velero-v1.16.1-linux-amd64
cp velero /usr/local/bin/
cd ~
```

### Step 3: Install Velero with MinIO Backend

```bash
velero install     --provider aws     --plugins velero/velero-plugin-for-aws:v1.10.0     --bucket velero-backups     --secret-file ./credentials-velero     --backup-location-config region=minio,s3Url=http://192.168.55.27:9000,s3ForcePathStyle="true"     --use-volume-snapshots=false
```

Replace `192.168.55.27` with your actual MinIO server IP if different.

### Step 4: Check Velero Status

```bash
kubectl get backupstoragelocations -n velero
kubectl logs deployment/velero -n velero
kubectl get pods -n velero
```

---

## 🔄 Use Velero: Backup, Restore, and Schedule

### Step 1: Take a Backup

```bash
velero backup create BACKUP-NAME --include-namespaces '*' --wait
```
Example:
```bash
velero backup create my-backup --include-namespaces '*' --wait
```

### Step 2: Restore from a Backup

```bash
velero get backup
velero restore create --from-backup BACKUP-NAME
```

### Step 3: Schedule Backups

This creates a scheduled backup to run daily at 07:00 UTC:

```bash
velero schedule create backup --schedule "0 7 * * *"
```

---

✅ You now have a complete Velero + MinIO + CSI backup solution!
