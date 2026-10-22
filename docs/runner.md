# Forgejo Runner Setup

Forgejo runner is used to run CI/CD pipelines for forgejo server. For security reasons, it is running in a separate vm on proxmox with limited access to this server.

## Setup

Run the following commands on the runner vm to install docker and sysbox. Update the SYSBOX_VER variable to the latest version of sysbox before running the commands.

```sh
apt update && apt upgrade -y

apt install qemu-guest-agent -y
systemctl enable --now qemu-guest-agent

curl -fsSL https://get.docker.com | sudo sh
sudo systemctl enable --now docker

# check newest version of sysbox here: https://github.com/nestybox/sysbox/releases
SYSBOX_VER=0.7.1
wget https://github.com/nestybox/sysbox/releases/download/v${SYSBOX_VER}/sysbox-ce_${SYSBOX_VER}.linux_amd64.deb
sha256sum sysbox-ce_${SYSBOX_VER}.linux_amd64.deb # Verify hash

apt install jq ./sysbox-ce_${SYSBOX_VER}.linux_amd64.deb -y
systemctl status sysbox -n20

apt install gpg htop -y
```

Make sysbox-runc the default runtime for docker by editing `/etc/docker/daemon.json` [reference](https://github.com/nestybox/sysbox/blob/master/docs/user-guide/install-package.md#configuring-dockers-default-runtime-to-sysbox):

```json
{
  "default-runtime": "sysbox-runc",
  "runtimes": {
    "sysbox-runc": {
      "path": "/usr/bin/sysbox-runc"
    }
  }
}
```

Restart docker to apply the changes:

```sh
sudo systemctl restart docker
```

Create runner user:

```sh
useradd --create-home runner
usermod -aG docker runner
```

Add Forgejo runner install/update script `/usr/local/sbin/update-runner` [docs](https://forgejo.org/docs/v16.0/admin/actions/installation/binary/#downloading-and-installing-the-binary):

```sh
#!/bin/bash
set -euo pipefail

if [[ $EUID -ne 0 ]]; then
  echo "This script must be run as root" >&2
  exit 1
fi

ARCH=$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/')
RUNNER_VERSION=$(curl -fsS https://data.forgejo.org/api/v1/repos/forgejo/runner/releases/latest | jq -r .name | cut -c 2-)
FORGEJO_URL="https://code.forgejo.org/forgejo/runner/releases/download/v${RUNNER_VERSION}/forgejo-runner-${RUNNER_VERSION}-linux-${ARCH}"

if [[ -x /usr/local/bin/forgejo-runner ]]; then
  CURRENT_VERSION=$(/usr/local/bin/forgejo-runner --version | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -n1 || true)
  if [[ "$CURRENT_VERSION" == "$RUNNER_VERSION" ]]; then
    echo "✓ forgejo-runner ${CURRENT_VERSION} is already up to date"
    exit 0
  fi
  echo "Updating forgejo-runner ${CURRENT_VERSION} -> ${RUNNER_VERSION}"
fi

TMP_DIR=$(mktemp -d)
trap 'rm -rf "$TMP_DIR"' EXIT
cd "$TMP_DIR"

wget -O forgejo-runner "${FORGEJO_URL}" || curl -fLo forgejo-runner "${FORGEJO_URL}"
wget -O forgejo-runner.asc "${FORGEJO_URL}.asc" || curl -fLo forgejo-runner.asc "${FORGEJO_URL}.asc"
gpg --keyserver hkps://keys.openpgp.org --recv EB114F5E6C0DC2BCDD183550A4B61A2DC5923710

if gpg --verify forgejo-runner.asc forgejo-runner; then
  echo "✓ Verified"
else
  echo "✗ Signature verification failed, not installing" >&2
  exit 1
fi

install -m 755 forgejo-runner /usr/local/bin/forgejo-runner
forgejo-runner --version

if systemctl cat forgejo-runner.service &>/dev/null; then
  systemctl restart forgejo-runner.service
  echo "✓ Restarted forgejo-runner.service"
fi
```

Generate config as runner user. Modify the config and add the runner token from the Forgejo server:

```sh
sudo -iu runner sh -c 'forgejo-runner generate-config > ~/runner-config.yml'
```

Create the systemd service `/etc/systemd/system/forgejo-runner.service` [docs](https://forgejo.org/docs/v16.0/admin/actions/installation/binary/#running-as-a-systemd-service):

```ini
[Unit]
Description=Forgejo Runner
Documentation=https://forgejo.org/docs/latest/admin/actions/
After=docker.service

[Service]
ExecStart=/usr/local/bin/forgejo-runner daemon -c /home/runner/runner-config.yml
ExecReload=/bin/kill -s HUP $MAINPID

User=runner
WorkingDirectory=/home/runner
Restart=on-failure
TimeoutStopSec=infinity
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now forgejo-runner.service
systemctl status forgejo-runner.service -n20
```
