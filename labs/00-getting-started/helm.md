# Installing Helm on Linux

Helm is the package manager for Kubernetes. It helps you install, upgrade, and manage Kubernetes applications easily.

---

## 1. Install via Official Script (Recommended, Easiest)

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

This script:
- Detects your OS and architecture
- Downloads the latest Helm release
- Installs `helm` to `/usr/local/bin`

---

## 2. Manual Installation

### Step 1: Download latest release
```bash
wget https://get.helm.sh/helm-v3.16.2-linux-amd64.tar.gz
```
*(Replace `v3.16.2` with the latest version from [Helm releases](https://github.com/helm/helm/releases)).*

### Step 2: Extract the archive
```bash
tar -zxvf helm-v3.16.2-linux-amd64.tar.gz
```

### Step 3: Move the binary to PATH
```bash
sudo mv linux-amd64/helm /usr/local/bin/helm
```

### Step 4: Verify installation
```bash
helm version
```

---

## 3. Install via Package Manager

### On Debian/Ubuntu
```bash
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### On Fedora/CentOS/RHEL
```bash
sudo dnf install helm
```
or
```bash
sudo yum install helm
```

---

## 4. Quick Test

Add a repo and search for a chart:
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm search repo nginx
```

If you see results, Helm is installed successfully.

---

## 5. Uninstall Helm (Optional)

If you need to remove Helm:
```bash
sudo rm /usr/local/bin/helm
```

---

✅ You now have Helm installed and ready to use on Linux.
