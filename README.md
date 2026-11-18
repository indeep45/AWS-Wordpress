# Deploying WordPress with Apache on AWS EC2 and MySQL on AWS RDS

A hands-on project that deploys a live **WordPress** website on **AWS** using:

- an **EC2** instance running **Apache HTTP Server (httpd)** and **PHP** as the web server, and
- a managed **MySQL database on Amazon RDS** to store all WordPress data.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Tech Stack and Configuration](#tech-stack-and-configuration)
- [Prerequisites](#prerequisites)
- [Project Process at a Glance](#project-process-at-a-glance)
- [Step 1: Launch an AWS EC2 Instance](#step-1-launch-an-aws-ec2-instance)
- [Step 2: Set Up a MySQL Database on RDS](#step-2-set-up-a-mysql-database-on-rds)
- [Step 3: Install and Configure Apache HTTPD](#step-3-install-and-configure-apache-httpd)
- [Step 4: Install WordPress on the EC2 Instance](#step-4-install-wordpress-on-the-ec2-instance)
- [Step 5: Run the WordPress Installation Wizard](#step-5-run-the-wordpress-installation-wizard)
- [Result](#result)
- [Installing Themes and Plugins](#installing-themes-and-plugins)
- [Security Best Practices](#security-best-practices)
- [Troubleshooting](#troubleshooting)
- [Cleanup](#cleanup)
- [Future Improvements](#future-improvements)
- [Acknowledgements](#acknowledgements)

---

## Project Overview

WordPress is a popular content management system (CMS) used by millions of websites. In this project, WordPress is deployed on AWS using a **two-tier architecture**:

| Tier | Service | Role |
|------|---------|------|
| Web tier | Amazon EC2 + Apache + PHP | Serves the WordPress site to visitors |
| Database tier | Amazon RDS (MySQL) | Stores posts, pages, users and settings |

Using RDS instead of installing a database on the EC2 instance means AWS manages the database (backups, patching, availability), and the web server can be rebuilt without losing site data.

## Architecture

```
                    Internet / Browser
                           |
                       HTTP :80
                           |
              +---------------------------+
              |  EC2 instance "WordPress" |
              |  Amazon Linux 2023        |
              |  Apache (httpd) + PHP     |
              |  WordPress in             |
              |  /var/www/html            |
              +-------------+-------------+
                            |
                      MySQL :3306
                            |
              +---------------------------+
              |  RDS MySQL (database-1)   |
              |  Database: wordpress      |
              +---------------------------+

        Region: Asia Pacific (Mumbai) ap-south-1, same VPC
```

## Tech Stack and Configuration

| Component | Details |
|-----------|---------|
| Cloud provider | AWS |
| Region | Asia Pacific (Mumbai) `ap-south-1` |
| EC2 instance name | `WordPress` |
| AMI | Amazon Linux 2023 |
| EC2 instance type | `t3.micro` |
| Web server | Apache HTTP Server (`httpd`) |
| Language | PHP with `php-mysqli` extension |
| CMS | WordPress (latest release) |
| Database | Amazon RDS, MySQL Community engine |
| RDS identifier | `database-1` |
| RDS class | `db.t4g.micro` (Free Tier template) |
| Database name | `wordpress` |

## Prerequisites

- An AWS account
- An EC2 key pair (`.pem`) for SSH access
- Basic Linux command line knowledge
- A web browser to complete the WordPress installer

## Project Process at a Glance

1. **Launch EC2**: create the `WordPress` instance with HTTP (port 80) open.
2. **Create RDS**: launch a MySQL database, note the endpoint, username and password.
3. **Create the `wordpress` database** from the EC2 instance using the MySQL client.
4. **Install Apache (httpd)** on EC2.
5. **Install PHP and WordPress**, then point `wp-config.php` at the RDS database.
6. **Start Apache** and finish the setup in the browser using the EC2 public IP.

---

## Step 1: Launch an AWS EC2 Instance

1. Log in to the **AWS Management Console**.
2. Go to the **EC2 Dashboard** and click **Launch instance**.
3. Set the instance name to **`WordPress`**.
4. Choose an **Amazon Machine Image (AMI)**: **Amazon Linux 2023 AMI**.
5. Choose an **instance type**. A small instance is enough for a small WordPress site (this project uses `t3.micro`).
6. Configure the key pair, network settings and storage. Create or select a **security group** that allows **HTTP (port 80)** inbound traffic (and SSH on port 22 so you can connect).
7. Review the configuration and click **Launch instance**.

**EC2 instance in the Running state**

![EC2 instance running](images/01-ec2-instance-running.png)

**Security group inbound rules (SSH 22 and HTTP 80)**

![EC2 security group](images/02-ec2-security-group-rules.png)

---

## Step 2: Set Up a MySQL Database on RDS

### 2.1 Create the RDS instance

1. Go to the **RDS Dashboard** in the AWS console.
2. Click **Create database** and select **Easy create**.
3. Choose the **MySQL** engine and the desired version.
4. In **Templates**, choose **Free tier**.
5. Configure the database settings: **DB instance identifier**, **instance size**, **master username** and **password**. Note down the **database endpoint, username and password**, because you will need them later.
6. Configure **connectivity**: choose the **VPC**, **subnet group** and **security group**. Allow the **MySQL port (3306)** in the security group, and use the **same security group** for both the EC2 instance and the MySQL database so they can talk to each other.
7. Connect the `WordPress` EC2 instance to the DB instance (RDS offers this as a connectivity option).
8. Review the settings and click **Create database**.

Wait until the RDS status shows **Available**.

![RDS available with connectivity details](images/03-rds-available-connectivity.png)

### 2.2 Install the MySQL client on EC2

Connect to the EC2 instance using SSH, then install a MySQL-compatible client so you can reach the RDS instance:

```bash
sudo yum install -y https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
sudo yum install -y mysql-community-client

# Run the following line if you get a GPG error, then run the previous line again
rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022

dnf install mariadb105-server
```

> On Amazon Linux 2023 the `mariadb105` package provides the `mysql` command-line client. This is the client shown in the screenshot below.

### 2.3 Create the `wordpress` database

Connect to RDS and create the database WordPress will use:

```bash
mysql -h <database_endpoint> -u <database_username> -p
```

```sql
CREATE DATABASE wordpress;
SHOW DATABASES;
```

**Connected to RDS from EC2 using the MySQL client**

![MySQL login and SHOW DATABASES](images/04-rds-mysql-login-show-databases.png)

---

## Step 3: Install and Configure Apache HTTPD

Connect to the EC2 instance using SSH, then update the packages and install Apache:

```bash
sudo yum update -y
sudo yum install httpd -y
```

---

## Step 4: Install WordPress on the EC2 Instance

### 4.1 Install PHP and the MySQL extension

```bash
sudo yum install php php-mysqli -y
```

### 4.2 Download and extract WordPress

```bash
cd /var/www/html
sudo wget https://wordpress.org/latest.tar.gz
sudo tar -xvf latest.tar.gz
sudo rm latest.tar.gz
sudo mv wordpress/* .
sudo rm -rf wordpress/
```

This downloads the latest WordPress package into Apache's web root, extracts it, moves the files up into `/var/www/html`, and cleans up the archive and the empty folder.

![WordPress files in /var/www/html](images/06-wordpress-files-var-www-html.png)

### 4.3 Configure WordPress to connect to RDS

```bash
cd /var/www/html
sudo mv wp-config-sample.php wp-config.php
sudo vim wp-config.php
```

In `wp-config.php`, update the database settings:

```php
define( 'DB_NAME',     'wordpress' );
define( 'DB_USER',     'your_database_username' );
define( 'DB_PASSWORD', 'your_database_password' );
define( 'DB_HOST',     'your_database_endpoint' );
```

![wp-config.php database settings (password hidden)](images/07-wp-config-db-settings.png)

> The password is hidden in the screenshot. **Never commit real credentials to Git.**

### 4.4 Start the Apache HTTPD server

```bash
sudo systemctl start httpd
sudo systemctl status httpd
```

The status should show **active (running)** and "listening on port 80".

To make Apache start automatically after a reboot:

```bash
sudo systemctl enable httpd
```

![Apache httpd active and running](images/05-apache-httpd-running.png)

> If Apache was already running before PHP was installed, restart it with `sudo systemctl restart httpd` so it picks up PHP.

---

## Step 5: Run the WordPress Installation Wizard

1. Open a web browser and enter your EC2 instance's **public IP** (or domain name):

   ```
   http://<EC2_PUBLIC_IP>/
   ```

2. Follow the WordPress installation wizard: enter the **site title**, **admin username**, **password** and **email**, then click **Install WordPress**.

   ![WordPress installation wizard](images/08-wordpress-install-wizard.png)

3. Log in at `http://<EC2_PUBLIC_IP>/wp-admin/`.

   ![WordPress login page](images/10-wordpress-login-page.png)

---

## Result

WordPress is installed and running on EC2, with all content stored in the RDS MySQL database. The admin dashboard is available at `http://<EC2_PUBLIC_IP>/wp-admin/`.

![WordPress admin dashboard](images/09-wordpress-admin-dashboard.png)

---

## Installing Themes and Plugins

To install and activate themes and plugins from the WordPress dashboard, give Apache ownership of the web root and tell WordPress to write files directly.

```bash
sudo chown -R apache:apache /var/www/html
```

Then add this line to `wp-config.php`:

```php
define('FS_METHOD', 'direct');
```

---

## Security Best Practices

- **Do not commit secrets.** Keep database and WordPress admin passwords out of Git. Use placeholders in documentation.
- **Restrict SSH (port 22)** to your own IP instead of `0.0.0.0/0`.
- **Limit MySQL (port 3306)** to the EC2 instance's security group only, and keep the database **not publicly accessible**.
- The site runs over **HTTP**. Add **HTTPS** before using it for real traffic.
- Use a **strong, unique admin password** and avoid a default username like `admin` on a production site.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Site does not load in the browser | Check the security group allows inbound HTTP (80) and that `httpd` is running (`systemctl status httpd`) |
| "Error establishing a database connection" | Check `DB_NAME`, `DB_USER`, `DB_PASSWORD` and `DB_HOST` in `wp-config.php`, and that the security group allows MySQL (3306) between EC2 and RDS |
| Cannot connect to RDS from EC2 | Confirm EC2 and RDS are in the same VPC and share (or allow) the security group; confirm the endpoint is correct |
| GPG error when installing the MySQL client | Run `rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022`, then retry the install |
| Browser shows PHP code or a blank page | Make sure `php` and `php-mysqli` are installed and restart Apache: `sudo systemctl restart httpd` |
| Cannot install themes or plugins | Run `sudo chown -R apache:apache /var/www/html` and add `define('FS_METHOD', 'direct');` to `wp-config.php` |
| Public IP changed after stopping/starting EC2 | The auto-assigned public IP is not permanent. Attach an **Elastic IP** to keep it fixed |

## Cleanup

To avoid AWS charges when you are done:

1. Terminate the EC2 instance.
2. Delete the RDS database (optionally take a final snapshot first).
3. Remove unused security groups and Elastic IPs.

## Future Improvements

- Attach an **Elastic IP** and use a custom domain with **Route 53**
- Enable **HTTPS** with an SSL certificate
- Add an **Application Load Balancer** and **Auto Scaling**
- Enable **RDS Multi-AZ** and automated backups
- Store media on **S3** and serve it through **CloudFront**
- Automate the deployment with **Terraform** or **CloudFormation**

## Acknowledgements

This project follows the guide *"Deploying WordPress with Apache on AWS EC2 and MySQL on AWS RDS"* by **Malthesh Achar**.

---

**Author:** Kumar
