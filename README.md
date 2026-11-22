# ERPNext Xinyu Ops
This repository contains the configuration files for deploying ERPNext with custom settings for Xinyu operations.

# Setup ECS

## Create Developer User
```bash
adduser developer
usermod -aG sudo developer
```
## Mount Data Disk
https://help.aliyun.com/zh/ecs/user-guide/initialize-a-data-disk-whose-size-does-not-exceed-2-tib-on-a-linux-instance
### Check Status
```bash
root@iZuf65aos2bxxnolrua06rZ:~# lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
NAME         SIZE FSTYPE MOUNTPOINTS
nvme1n1       80G        
nvme0n1       40G        
├─nvme0n1p1    1M        
├─nvme0n1p2  200M vfat   /boot/efi
└─nvme0n1p3 39.8G ext4   /
```
### Create GPT Partition
```bash
parted /dev/nvme1n1 --script mklabel gpt
parted /dev/nvme1n1 --script mkpart data ext4 0% 100%
```
### Create FS
```bash
mkfs -t ext4 /dev/nvme1n1
```
### Mount
```bash
mount /dev/nvme1n1 /opt
```
### Auto Mount
```bash
blkid /dev/nvme1n1
UUID=ee4a919e-9080-439c-9601-dc1306111154 /opt ext4 defaults,nofail 0 2
```
## Install Docker
### Install
- https://docs.docker.com/engine/install/ubuntu/
- Need to use aliyun source: https://help.aliyun.com/zh/ecs/user-guide/install-and-use-docker#8dca4cfa3dn0e
- Need to config docker hub mirror to bypass the wall
### Use Docker as a non-privileged user
```bash
su - developer
sudo usermod -aG docker $USER
exit
```
https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user

## Install Docker Compose
```bash
apt install docker-compose-plugin
```

## Install MariaDB
### Install
```bash
apt install mariadb-server mariadb-client galera-4
```
https://mariadb.com/docs/server/mariadb-quickstart-guides/installing-mariadb-server-guide
### Migrate MariaDB data to opt
```bash
systemctl stop mariadb.service
mkdir /opt/mariadb
chown mysql:mysql /opt/mariadb

rsync -avHAX --numeric-ids /var/lib/mariadb/ /opt/mariadb/
rsync -avHAX --numeric-ids /var/log/mysql/ /opt/log/mariadb/

vim /etc/mysql/mariadb.conf.d/50-server.cnf
change datadir

vim /etc/apparmor.d/mariadbd
/opt/mariadb/ r,
/opt/mariadb/** rwk,
apparmor_parser -r /etc/apparmor.d/mariadbd
systemctl start mariadb

confirm
mariadb -e "SELECT @@datadir;"
```
### Create a User for Xinyu
```bash
create user 'xinyu'@'%' identified by 'your password';
grant all privileges on *.* to 'xinyu'@'%' with grant option;
flush privileges;
```
### Custom Setting
```bash
bind-address            = 0.0.0.0
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
innodb_snapshot_isolation = OFF
```
# Build Docker Image

## Build in Local
```bash
docker build \
  --build-arg=FRAPPE_BRANCH=version-16 \
  --secret=id=apps_json,src=./images/apps.json \
  --file=./images/Containerfile \
  --tag=erpnext-xinyu:0.1 .
```
We can specify the `FRAPPE_PATH` and `FRAPPE_BRANCH` build arguments if necessary.
## Export
```bash
sudo docker save -o erpnext-xinyu.tar erpnext-xinyu:0.1
```
## Upload
```bash
rsync -avz --info=progress2 -e ssh erpnext-xinyu.tar developer@106.14.8.99:/opt/work/
```
## Import
```bash
sudo docker load -i erpnext-xinyu.tar
```

# Start Services
```bash
docker compose up -d 
```

# Create sites and install Apps

```bash
docker compose exec backend \
  bench new-site erp \
  --mariadb-user-host-login-scope=% \
  --db-root-username xinyu
  --db-root-password 'db password' \
  --admin-password 'admin password in UI' \
  --install-app erpnext \
  --install-app erpnext_china \
  --install-app erpnext_xinyu
```
