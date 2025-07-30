# Production-Safe Server Environment Setup Guide for CentOS 7

**Author:** Abubakkar Khan Fazla Rabbi | System Administrator | Schertech |
**Date:** July 30, 2025

This document outlines the procedures for configuring a CentOS 7 server environment to support JSP and Java applications, ensuring compliance with production safety standards. The guide covers the installation and configuration of Java 1.8, Apache Tomcat 8 or later, MariaDB (MySQL-compatible), and the necessary environment variables. All steps prioritize security, reliability, and best practices for a production-ready environment.

---

## Prerequisites
- CentOS 7 installed (minimal or server edition).
- Root or sudo privileges.
- Internet access for package downloads.
- Basic familiarity with the Linux command line.

---

## Step 1: System Update
It is imperative to update the system to incorporate the latest security patches and package versions.

```bash
sudo yum update -y
sudo yum upgrade -y
```

If a kernel update is applied, reboot the system to ensure changes take effect:
```bash
sudo reboot
```

---

## Step 2: Install Java 1.8
Install OpenJDK 1.8 (Java 8), which includes `javac` for compiling Java code. OpenJDK is selected for its compatibility, support, and open-source nature.

1. Install OpenJDK 1.8:
```bash
sudo yum install -y java-1.8.0-openjdk java-1.8.0-openjdk-devel
```

2. Verify the Java installation:
```bash
java -version
```
Expected output:
```
openjdk version "1.8.0_XXX"
OpenJDK Runtime Environment (build 1.8.0_XXX-...)
OpenJDK 64-Bit Server VM (build 25.XXX-..., mixed mode)
```

3. Verify `javac` installation:
```bash
javac -version
```
Expected output:
```
javac 1.8.0_XXX
```

---

## Step 3: Configure JAVA_HOME Environment Variable
Set the `JAVA_HOME` environment variable system-wide to ensure consistent Java configuration across applications.

1. Determine the Java installation path:
```bash
sudo find /usr/lib/jvm -name 'java-1.8.0-openjdk*'
```
Example output:
```
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.XXX-...
```

2. Set `JAVA_HOME` in `/etc/environment`:
```bash
sudo nano /etc/environment
```
Add the following line (replace with the actual path):
```
JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.XXX-...
```

3. Apply the environment variable:
```bash
source /etc/environment
```

4. Verify `JAVA_HOME`:
```bash
echo $JAVA_HOME
```
Expected output:
```
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.XXX-...
```

5. Ensure `JAVA_HOME` persists across sessions by adding it to the shell profile:
```bash
sudo nano /etc/profile.d/java.sh
```
Add:
```bash
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.XXX-...
export PATH=$JAVA_HOME/bin:$PATH
```

6. Make the script executable and apply it:
```bash
sudo chmod +x /etc/profile.d/java.sh
source /etc/profile.d/java.sh
```

---

## Step 4: Install Apache Tomcat 8
Install and configure Apache Tomcat 8 to host JSP and Java applications. The version specified is the latest stable release at the time of writing; however, it is recommended to check for updates.

1. Create a dedicated user for Tomcat to enhance security by avoiding root privileges:
```bash
sudo useradd -r -m -s /bin/false tomcat
```

2. Download and extract Apache Tomcat 8:
```bash
sudo mkdir /opt/tomcat
cd /opt/tomcat
sudo wget https://archive.apache.org/dist/tomcat/tomcat-8/v8.5.82/bin/apache-tomcat-8.5.82.tar.gz
sudo tar -xvzf apache-tomcat-8.5.82.tar.gz
sudo mv apache-tomcat-8.5.82 tomcat8
sudo rm apache-tomcat-8.5.82.tar.gz
```

3. Set ownership and permissions for the Tomcat directory:
```bash
sudo chown -R tomcat:tomcat /opt/tomcat/tomcat8
sudo chmod -R 755 /opt/tomcat/tomcat8
```

4. Create a systemd service file for managing Tomcat:
```bash
sudo nano /etc/systemd/system/tomcat.service
```
Add the following configuration:
```tomcat
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target

[Service]
Type=forking
Environment="JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.XXX-..."
Environment="CATALINA_PID=/opt/tomcat/tomcat8/temp/tomcat.pid"
Environment="CATALINA_HOME=/opt/tomcat/tomcat8"
Environment="CATALINA_BASE=/opt/tomcat/tomcat8"
ExecStart=/opt/tomcat/tomcat8/bin/startup.sh
ExecStop=/opt/tomcat/tomcat8/bin/shutdown.sh
User=tomcat
Group=tomcat
UMask=0022
RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
```

5. Reload systemd and start Tomcat:
```bash
sudo systemctl daemon-reload
sudo systemctl start tomcat
sudo systemctl enable tomcat
```

6. Verify Tomcat is running:
```bash
sudo systemctl status tomcat
```
Access the Tomcat welcome page at `http://<server-ip>:8080` to confirm functionality.

7. Harden Tomcat for production:
   - Disable unnecessary default applications to reduce attack surface:
```bash
sudo mv /opt/tomcat/tomcat8/webapps/{docs,examples,host-manager,manager} /opt/tomcat/tomcat8/webapps-backup
```
   - Set secure permissions for configuration files:
```bash
sudo chmod 640 /opt/tomcat/tomcat8/conf/*
sudo chown tomcat:tomcat /opt/tomcat/tomcat8/conf/*
```

8. (Optional) Configure administrative access for Tomcat management:
```bash
sudo nano /opt/tomcat/tomcat8/conf/tomcat-users.xml
```
Add within `<tomcat-users>`:
```xml
<role సామాన్యం="manager-gui"/>
<role సామాన్యం="admin-gui"/>
<user username="admin" password="secure_password_here" roles="manager-gui,admin-gui"/>
```
Restart Tomcat to apply changes:
```bash
sudo systemctl restart tomcat
```

---

## Step 5: Install MariaDB
Install and configure MariaDB, a MySQL-compatible database, for application data storage.

1. Install MariaDB:
```bash
sudo yum install -y mariadb-server mariadb
```

2. Start and enable MariaDB to run at boot:
```bash
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

3. Secure the MariaDB installation:
```bash
sudo mysql_secure_installation
```
Follow the prompts to:
   - Set a strong root password.
   - Remove anonymous users.
   - Disallow remote root login.
   - Remove the test database.
   - Reload privilege tables.

4. Verify MariaDB functionality:
```bash
mysql -u root -p
```
Enter the root password to confirm access.

5. Create a dedicated database and user for the application:
```sql
CREATE DATABASE myappdb;
CREATE USER 'myappuser'@'localhost' IDENTIFIED BY 'secure_password_here';
GRANT ALL PRIVILEGES ON myappdb.* TO 'myappuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

## Step 6: Configure Firewall
Restrict network access to only essential ports to minimize security risks.

1. Install and enable firewalld:
```bash
sudo yum install -y firewalld
sudo systemctl start firewalld
sudo systemctl enable firewalld
```

2. Allow traffic on ports 8080 (Tomcat) and 3306 (MariaDB):
```bash
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --reload
```

3. Verify firewall rules:
```bash
sudo firewall-cmd --list-all
```

---

## Step 7: Monitoring and Maintenance
1. Monitor Tomcat logs for errors or issues:
```bash
sudo tail -f /opt/tomcat/tomcat8/logs/catalina.out
```
2. Monitor MariaDB logs:
```bash
sudo tail -f /var/log/mariadb/mariadb.log
```
3. Configure log rotation for Tomcat to manage log file sizes:
```bash
sudo nano /etc/logrotate.d/tomcat
```
Add:
```
/opt/tomcat/tomcat8/logs/*.log {
    daily
    rotate 14
    compress
    missingok
    notifempty
    create 640 tomcat tomcat
}
```

4. Regularly update the system to apply security patches:
```bash
sudo yum update -y
```

---

## Step 8: Security Best Practices
- Keep Java, Tomcat, and MariaDB updated to the latest stable versions.
- Use strong, unique passwords for all accounts.
- Restrict SSH access (e.g., use key-based authentication).
- Regularly back up the database:
```bash
mysqldump -u root -p myappdb > myappdb_backup.sql
```
- Enable SELinux in enforcing mode for enhanced security:
```bash
sudo setenforce 1
sudo nano /etc/selinux/config
```
Set `SELINUX=enforcing`.

---

## Troubleshooting
- If Tomcat fails to start, verify `JAVA_HOME` and directory permissions.
- If MariaDB access is denied, confirm user credentials and host permissions.
- Ensure firewall rules permit traffic on ports 8080 and 3306.


**Prepared by:**  
Abubakkar Khan  
System Administrator  
Schertech  

**Date:** July 30, 2025
