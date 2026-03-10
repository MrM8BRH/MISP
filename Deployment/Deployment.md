<details>
<summary><b>Minimum System Requirements (Production)</b></summary>

Recommended baseline for **production environments**:
| Component      | Recommended                         |
| -------------- | ----------------------------------- |
| CPU            | 8+ vCPUs                            |
| RAM            | 32–128 GB                           |
| Disk           | 300–500 GB SSD                      |
| OS			       | Debian / RHEL / Rocky Linux  		   |

**Partitioning**:

| Mount Point  | Filesystem 				| Size              | Device Type            |
|-------------|-----------------------------|-------------------|------------------------|
| swap        | swap       					| 16 GB            | LVM					 |
| /tmp        | ext4       					| 16 GB            | LVM				     |
| /           | ext4       					| 20 GB            | LVM                    |
| /boot       | ext4       					| 1 GB             | Standard partition	 |
| /boot/efi   | EFI System Partition        | 1 GB             | Standard partition	 |
| biosboot    | BIOS Boot			        | 1 GB             | Standard partition	 |
| /var        | ext4       					| Remaining storage | LVM                    |

</details>

<details>
<summary><b>Preparing a System Before MISP Installation</b></summary>

<details>
<summary><b>Change hostname</b></summary>

```
hostnamectl set-hostname MISP
```
</details>

<details>
<summary><b>Change Timezone</b></summary>

```
timedatectl
timedatectl set-timezone Asia/Jerusalem
```
</details>

<details>
<summary><b>Disable SELinux</b></summary>

Set it to `permissive` or `disabled`:
```
# Check the current status and mode of SELinux.
sestatus

# Opens the SELinux configuration file using the nano text editor.
nano /etc/selinux/config

# A configuration option that can be set in the SELinux configuration file to disable SELinux on the system,
# preventing it from enforcing security policies.
SELINUX=disabled
```
Apply immediately:
```
sudo setenforce 0
```
</details>

<details>
<summary><b>Update the system & Install additional tools</b></summary>

RHEL family
```
yum update -y
yum install -y dnf
dnf install -y net-tools nano bind-utils chkconfig wget net-tools cloud-utils-growpart
```
Debian family
```
apt update -y
apt full-upgrade -y
apt install -y net-tools nano wget net-tools cloud-guest-utils
```
</details>

<details>
<summary><b>Disable Firewall</b></summary>

RHEL family
```
systemctl stop firewalld
systemctl disable firewalld
```
Debian family
```
systemctl stop ufw
systemctl disable ufw
```
</details>


<details>
<summary><b>Disable Transparent Huge Pages (THP)</b></summary>

```
nano /etc/systemd/system/disable-thp.service
```
```
[Unit]
Description=Disable Transparent Huge Pages (THP)

[Service]
Type=simple
ExecStart=/bin/sh -c "echo 'never' > /sys/kernel/mm/transparent_hugepage/enabled && echo 'never' > /sys/kernel/mm/transparent_hugepage/defrag"

[Install]
WantedBy=multi-user.target
```

```
systemctl daemon-reload
systemctl start disable-thp
systemctl enable disable-thp
```
</details>

<details>
<summary><b>System Tuning (Required)</b></summary>

Enable Memory Overcommit for Redis
- Open `/etc/sysctl.conf` for editing.
- Add `vm.overcommit_memory = 1` to the end of the file.
- Run `sudo sysctl -p`.
</details>


```diff
- After completing the above, restart the system
reboot
```
</details>

<details>
<summary><b>Docker Installation</b></summary>

#### Install Docker Engine on RHEL
```
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
#### Add the current user to the 'docker' group.
```
sudo usermod -aG docker $USER
```
#### Verify Docker Compose Version
```
docker compose version
```
#### Start & Enable Docker Service
```
sudo systemctl start docker.socket
sudo systemctl enable docker.socket
```

#### Install Docker Engine on Debian
[Link](https://docs.docker.com/engine/install/debian/)

#### Install Docker Engine on Ubuntu
[Link](https://docs.docker.com/engine/install/ubuntu/)
</details>

<details>
<summary><b>MISP Installation & Configuration</b></summary>

#### Download MISP Docker Compose Files
```
mkdir /opt/misp && cd /opt/misp
wget https://raw.githubusercontent.com/MISP/misp-docker/refs/heads/master/docker-compose.yml
wget https://raw.githubusercontent.com/MISP/misp-docker/refs/heads/master/template.env
```
#### Configure Environment Variables
```
cp template.env .env
nano .env
```
Change the following
```
ADMIN_EMAIL
ADMIN_ORG
ADMIN_PASSWORD
GPG_PASSPHRASE
ENCRYPTION_KEY
BASE_URL
# Optional and used by the mail sub-system
  SMARTHOST_ADDRESS
  SMARTHOST_PORT
  SMARTHOST_USER
  SMARTHOST_PASSWORD
# Performance Tuning
  ### PHP
    PHP_MEMORY_LIMIT
    PHP_MAX_EXECUTION_TIME
    PHP_UPLOAD_MAX_FILESIZE
    PHP_POST_MAX_SIZE
    PHP_MAX_INPUT_TIME
    PHP_MAX_FILE_UPLOADS
  ### MariaSQL/MySQL (InnoDB)
    INNODB_BUFFER_POOL_SIZE
    INNODB_CHANGE_BUFFERING
    INNODB_IO_CAPACITY
    INNODB_IO_CAPACITY_MAX
    INNODB_LOG_FILE_SIZE
    INNODB_READ_IO_THREADS
    INNODB_STATS_PERSISTENT
    INNODB_WRITE_IO_THREADS
```
#### Notes

> **Important:** In the table below, several settings are formatted in **bold**. It is strongly recommended to override these values at a minimum before deployment.

> **Important:** Any passwords used **must not contain** the following characters:  
> `\` (backslash) or `+` (plus).  
> Including these characters may cause the container to fail during startup.

> **Security Recommendation:** Ensure that all passwords are **strong and unique**.  
> It is recommended to use a **cryptographically secure password generator**.

The following command generates a 64-character hexadecimal password suitable for secure configurations.

```
openssl rand -hex 32
```

#### Start MISP
```
docker compose up -d
```

> Disabling `MISP.showCorrelationsOnIndex` fixed the login delay issue.
</details>

<details>
<summary><b>Scheduled Updates</b></summary>

crontab -e
```
# Update MISP taxonomies (classification tags such as TLP, threat categories, etc.)
0 1 * * * /usr/bin/curl -XPOST --insecure --header "Authorization: YOUR_API_KEY" --header "Accept: application/json" --header "Content-Type: application/json" https://YOUR_MISP_ADDRESS/taxonomies/update

# Update warning lists (known benign domains, IP ranges, and false-positive indicators)
10 1 * * * /usr/bin/curl -XPOST --insecure --header "Authorization: YOUR_API_KEY" --header "Accept: application/json" --header "Content-Type: application/json" https://YOUR_MISP_ADDRESS/warninglists/update

# Update notice lists (security advisories and contextual security notices)
20 1 * * * /usr/bin/curl -XPOST --insecure --header "Authorization: YOUR_API_KEY" --header "Accept: application/json" --header "Content-Type: application/json" https://YOUR_MISP_ADDRESS/noticelists/update

# Update galaxies (threat actors, malware families, campaigns, and ATT&CK mappings)
30 1 * * * /usr/bin/curl -XPOST --insecure --header "Authorization: YOUR_API_KEY" --header "Accept: application/json" --header "Content-Type: application/json" https://YOUR_MISP_ADDRESS/galaxies/update

# Update object templates (data structures used for structured threat intelligence objects)
40 1 * * * /usr/bin/curl -XPOST --insecure --header "Authorization: YOUR_API_KEY" --header "Accept: application/json" --header "Content-Type: application/json" https://YOUR_MISP_ADDRESS/objectTemplates/update

# Sync all configured MISP feeds daily (threat intelligence indicators)
0 2 * * * /usr/bin/curl -XPOST --insecure --header "Authorization: YOUR_API_KEY" --header "Accept: application/json" --header "Content-Type: application/json" https://YOUR_MISP_ADDRESS/feeds/fetchFromAllFeeds
```
</details>

<details>
<summary><b>Upgrading the MISP Instance</b></summary>

```
# Pull any new/updated images from the DockerHub repository
docker compose pull

# Tear down the current containers, and re-deploy them using the latest images that have been pulled.
docker compose up -d --force-recreate
```
</details>

<details>
<summary><b>Validation & Troubleshooting</b></summary>

- Verify Running Containers
```
docker ps -a
```
- View Logs
```
docker logs container_name
```
- Logs Path:
```
/opt/misp/logs/opt/misp/logs
```
</details>
