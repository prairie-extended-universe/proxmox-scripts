# Proxmox LXC Template

This script takes a base Debian image and applies some sensible defaults, and installs Docker and Tailscale.

Currently tested on the `debian-13-standard_13.6-1_amd64` image from Proxmox.

# Initial template setup

```sh
apt update
apt upgrade -y

# https://docs.docker.com/engine/install/debian/#install-using-the-repository
apt install ca-certificates curl -y
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

apt update
apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

mkdir /opt/docker-configs

# https://tailscale.com/docs/install/linux
curl -fsSL https://tailscale.com/install.sh | sh

# https://tailscale.com/docs/concepts/userspace-networking
sed -i 's/.*FLAGS.*/FLAGS="--tun=userspace-networking"/' /etc/default/tailscaled

# https://serverfault.com/questions/84521/automate-dpkg-reconfigure-tzdata/84528#84528
ln -fs /usr/share/zoneinfo/America/Detroit /etc/localtime
dpkg-reconfigure -f noninteractive tzdata
```

# Individual post-clone setup

Be sure to select "full clone".

```sh
passwd
tailscale up

# https://superuser.com/questions/1156344/ddg#1156345
rm /etc/ssh/ssh_host_*
ssh-keygen -A
```

```sh
cd /opt/docker-configs
mkdir example
cd example
vim compose.yaml
docker compose up
```
