# Vprofile Project Setup Walkthrough
## Manual Provisioning on Local VMs (Windows/Mac Intel)

This is a corrected and updated walkthrough for setting up the Vprofile multi-tier web application on local VMs using Vagrant and VirtualBox. The original guide had several broken and outdated instructions — I hit 8 issues while following it, debugged every one, and documented the fixes here.

> Based on the Vprofile project by [Imran Teli (hkhcoder)](https://github.com/hkhcoder/vprofile-project) — clone the source code from his repo, then follow this guide to set it up correctly.

---


## 🧰 Stack

`Vagrant` · `VirtualBox` · `Nginx` · `Tomcat 10` · `Java 17` · `MariaDB` · `RabbitMQ` · `Memcached` · `CentOS Stream 9` · `Ubuntu Jammy`

---

## Architecture Overview

| VM | Hostname | IP | Role |
|----|----------|----|------|
| web01 | web01 | 192.168.56.11 | Nginx (reverse proxy) |
| app01 | app01 | 192.168.56.12 | Tomcat 10 (application server) |
| rmq01 | rmq01 | 192.168.56.13 | RabbitMQ (message broker) |
| mc01 | mc01 | 192.168.56.14 | Memcache (DB caching) |
| db01 | db01 | 192.168.56.15 | MySQL/MariaDB (database) |

---

## Prerequisites

- Oracle VM VirtualBox
- Vagrant
- Git Bash (Windows) or Terminal (Mac)

Install required Vagrant plugins:

```bash
vagrant plugin install vagrant-hostmanager
```

---

## Clone This Repo

Clone this repo to get the corrected Vagrantfile and this guide:

```bash
git clone https://github.com/Kanishkchahar/vprofile-setup-fixes.git
cd vprofile-setup-fixes
```

> The corrected `Vagrantfile` is included. Use it as-is — no changes needed.

> **Note:** The application source code is from [hkhcoder/vprofile-project](https://github.com/hkhcoder/vprofile-project). You will clone it later inside the VMs during the setup steps — you do not need to clone it now.

Then bring up the VMs:

```bash
vagrant up
```

> This may take a while. If it stops midway, run `vagrant up` again.

---

## Setup Order

Always provision services in this order:

1. MySQL (db01)
2. Memcache (mc01)
3. RabbitMQ (rmq01)
4. Tomcat (app01)
5. Nginx (web01)

---

## 1. MySQL Setup (db01)

```bash
vagrant ssh db01
sudo -i
```

```bash
dnf update -y
dnf install epel-release -y
dnf install git mariadb-server -y
systemctl start mariadb
systemctl enable mariadb
mysql_secure_installation
```

During secure installation use these answers:

| Prompt | Answer |
|--------|--------|
| Enter current password for root | Press **Enter** (no password yet) |
| Switch to unix_socket authentication | **Y** |
| Change the root password | **Y** → set to `admin123` |
| Remove anonymous users | **Y** |
| Disallow root login remotely | **n** |
| Remove test database and access to it | **Y** |
| Reload privilege tables now | **Y** |

> **Note:** Newer versions of MariaDB add two extra prompts at the top — "Switch to unix_socket authentication" and a separate "Change the root password" prompt — before the standard questions. Answer Y to both, setting the password to `admin123` when prompted.

```bash
mysql -u root -padmin123
```

```sql
create database accounts;
grant all privileges on accounts.* TO 'admin'@'localhost' identified by 'admin123';
grant all privileges on accounts.* TO 'admin'@'%' identified by 'admin123';
FLUSH PRIVILEGES;
exit;
```

Clone source and initialise the database:

```bash
cd /tmp/
git clone -b local https://github.com/hkhcoder/vprofile-project.git
cd vprofile-project
mysql -u root -padmin123 accounts < src/main/resources/db_backup.sql
systemctl restart mariadb
```

Open the firewall for port 3306:

```bash
systemctl start firewalld
systemctl enable firewalld
firewall-cmd --zone=public --add-port=3306/tcp --permanent
firewall-cmd --reload
systemctl restart mariadb
```

---

## 2. Memcache Setup (mc01)

```bash
vagrant ssh mc01
sudo -i
```

```bash
dnf update -y
dnf install epel-release -y
dnf install memcached -y
systemctl start memcached
systemctl enable memcached
sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
systemctl restart memcached
```

Open firewall ports:

```bash
systemctl start firewalld
systemctl enable firewalld
firewall-cmd --add-port=11211/tcp --permanent
firewall-cmd --add-port=11111/udp --permanent
firewall-cmd --reload
memcached -p 11211 -U 11111 -u memcached -d
```

> **Note:** The original guide uses `--runtime-to-permanent` which throws a **Permission denied** error (`Errno 13: /etc/sysconfig/network-scripts/ifcfg-eth1`). Use `--permanent` flag directly on each command instead, followed by `--reload`. This achieves the same result without the error.

---

## 3. RabbitMQ Setup (rmq01)

```bash
vagrant ssh rmq01
sudo -i
```

```bash
dnf update -y
dnf install epel-release -y
dnf install wget -y
dnf -y install centos-release-rabbitmq-38
dnf --enablerepo=centos-rabbitmq-38 -y install rabbitmq-server
systemctl enable --now rabbitmq-server
```

Create a user and configure access:

```bash
sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
sudo rabbitmqctl add_user test test
sudo rabbitmqctl set_user_tags test administrator
rabbitmqctl set_permissions -p / test ".*" ".*" ".*"
sudo systemctl restart rabbitmq-server
```

Open firewall port:

```bash
systemctl start firewalld
systemctl enable firewalld
firewall-cmd --add-port=5672/tcp --permanent
firewall-cmd --reload
sudo systemctl restart rabbitmq-server
sudo systemctl enable rabbitmq-server
```

> **Note:** Same as mc01 — `--runtime-to-permanent` throws `Permission denied (Errno 13)` on rmq01. Use `--permanent` directly on each command and follow with `--reload` instead.

---

## 4. Tomcat Setup (app01)

```bash
vagrant ssh app01
sudo -i
```

```bash
dnf update -y
dnf install epel-release -y
dnf -y install java-17-openjdk java-17-openjdk-devel
dnf install git unzip -y
```

### ✅ Install Tomcat 10 via /vagrant shared folder

> **Why Tomcat 10?** The vprofile WAR uses the `jakarta.servlet` namespace which requires **Tomcat 10+**. Tomcat 9 uses the old `javax.servlet` namespace and will fail with `ClassNotFoundException: jakarta.servlet.ServletContextListener`.

> ⚠️ `wget` from inside the VM will fail — `archive.apache.org` and `dlcdn.apache.org` are unreachable from the VM's network. That is why we use the `/vagrant` shared folder set up earlier.

```bash
# Copy from shared folder and extract
cp /vagrant/apache-tomcat-10.1.26.tar.gz /tmp/
cd /tmp
tar xzvf apache-tomcat-10.1.26.tar.gz
```

> **Note:** Replace `10.1.26` with whatever version you downloaded. Verify the exact name with `ls /tmp/ | grep tomcat`.

Create the tomcat user and install:

```bash
useradd --home-dir /usr/local/tomcat --shell /sbin/nologin tomcat
mkdir -p /usr/local/tomcat
cp -r /tmp/apache-tomcat-10.1.26/* /usr/local/tomcat/
chown -R tomcat:tomcat /usr/local/tomcat
```

### ✅ Correct Tomcat Service File

> **What was wrong in the original guide:**
> - `JAVA_HOME=/usr/lib/jvm/jre` does not exist on CentOS 9 with Java 17
> - `ExecStart=catalina.sh run` is foreground mode but `Type=forking` expects a background process

```bash
vi /etc/systemd/system/tomcat.service
```

```ini
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat
Environment=JAVA_HOME=/usr/lib/jvm/java-17-openjdk-17.0.18.0.8-2.el9.x86_64
Environment=CATALINA_PID=/var/tomcat/%i/run/tomcat.pid
Environment=CATALINA_HOME=/usr/local/tomcat
Environment=CATALINA_BASE=/usr/local/tomcat
WorkingDirectory=/usr/local/tomcat
ExecStart=/usr/local/tomcat/bin/startup.sh
ExecStop=/usr/local/tomcat/bin/shutdown.sh
SuccessExitStatus=143
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

> **Important:** The `JAVA_HOME` path must match your exact installed JDK version. Check it with:
> ```bash
> ls /usr/lib/jvm/
> ```
> Then update the `Environment=JAVA_HOME=...` line accordingly.

```bash
systemctl daemon-reload
systemctl start tomcat
systemctl enable tomcat
```

### ✅ Fix Firewall Blocking Port 8080

> CentOS 9 runs `firewalld` by default even if you never started it manually. This silently blocks port 8080, causing a "No route to host" error from web01. We open just the required port — consistent with how db01, mc01, and rmq01 are configured.

```bash
systemctl start firewalld
systemctl enable firewalld
firewall-cmd --zone=public --add-port=8080/tcp --permanent
firewall-cmd --reload
```

Verify Tomcat is listening:

```bash
ss -tlnp | grep 8080
```

---

## 4b. Maven Setup (app01)

> ⚠️ Same network restriction applies for Maven — `dlcdn.apache.org` is unreachable from inside the VM. Use the `/vagrant` shared folder.

```bash
cp /vagrant/apache-maven-3.9.16-bin.zip /tmp/
cd /tmp
unzip apache-maven-3.9.16-bin.zip
cp -r apache-maven-3.9.16 /usr/local/maven3.9
export MAVEN_OPTS="-Xmx512m"
```

> **Note:** Replace `3.9.16` with whatever version you downloaded. Verify with `ls /tmp/ | grep maven`.

Clone source and verify configuration:

```bash
cd /tmp
git clone -b local https://github.com/hkhcoder/vprofile-project.git
cd vprofile-project
vim src/main/resources/application.properties
```

Confirm these backend hostnames are set correctly:

| Setting | Value |
|---------|-------|
| `jdbc.url` | `jdbc:mysql://db01:3306/accounts` |
| `memcached.active.host` | `mc01` |
| `rabbitmq.address` | `rmq01` |

Build the project:

```bash
/usr/local/maven3.9/bin/mvn install
```

### Deploy WAR to Tomcat

```bash
systemctl stop tomcat
rm -rf /usr/local/tomcat/webapps/ROOT*
cp /tmp/vprofile-project/target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
chown -R tomcat:tomcat /usr/local/tomcat/webapps/
systemctl start tomcat
```

Watch deployment logs:

```bash
tail -f /usr/local/tomcat/logs/catalina.out
```

Wait for: `Deployment of web application archive [...] ROOT.war has finished`

---

## 5. Nginx Setup (web01)

```bash
vagrant ssh web01
sudo -i
```

```bash
apt update && apt upgrade -y
apt install nginx -y
```

Create the Nginx config:

```bash
vi /etc/nginx/sites-available/vproapp
```

```nginx
upstream vproapp {
    server app01:8080;
}

server {
    listen 80;
    location / {
        proxy_pass http://vproapp;
    }
}
```

Activate the config and restart Nginx:

```bash
rm -rf /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp
systemctl restart nginx
```

---


## Verification

Visit `http://192.168.56.11` in your browser. You should see the Vprofile login page.

Default credentials:
- **Username:** `admin_vp`
- **Password:** `admin_vp`