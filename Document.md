# SSO Integration with Keycloak and 389ds for Grafana, GitLab, and MinIO Access

## Table of Contents
- [Setup Overview](#setup-overview)
- [Prerequisites](#prerequisites)
- [Keycloak](#keycloak)
  - [System Requirements](#system-requirements)
  - [Step 1: Update System and Install Java](#step-1-update-system-and-install-java)
  - [Step 2: Set JAVA_HOME Environment Variable](#step-2-set-java_home-environment-variable)
  - [Step 3: Download Keycloak 26.1.0](#step-3-download-keycloak-2610)
  - [Step 4: Create a Systemd Service File](#step-4-create-a-systemd-service-file)
  - [Step 5: Set Execution Permission](#step-5-set-execution-permission)
  - [Step 6: Reload Systemd and Enable Keycloak Service](#step-6-reload-systemd-and-enable-keycloak-service)
  - [Step 7: Verify the Service Status](#step-7-verify-the-service-status)
- [Keycloak GUI setup](#keycloak-gui-setup)
  - [Create a realm](#create-a-realm)
  - [Create a client](#create-a-client)
- [Grafana](#grafana)
  - [Introduction to Grafana](#introduction-to-grafana)
  - [Steps to Run Grafana in a Podman Container](#steps-to-run-grafana-in-a-podman-container)
- [MinIO](#minio)
- [GitLab](#gitlab)
- [OAuth2 Proxy](#oauth2-proxy)
- [Nginx installation](#nginx-installation)
- [389ds](#389ds)

---

## Setup Overview
This setup enables Single Sign-On (SSO) using Keycloak and 389ds, allowing seamless access to Grafana, GitLab, and MinIO with a single login. The flow is as follows:

1. Access `grafana.keen.com` to sign in using Ankit's SSO credentials.
2. Authenticate with 389ds credentials on the Keycloak page.
3. Enter Grafana credentials to reach the Grafana home page.
4. On subsequent visits to `grafana.keen.com`, users are directly taken to the Grafana home page without needing to log in again.
5. Similarly, this SSO integration also works for GitLab and MinIO, where users authenticate once via Keycloak and 389ds to gain access to all three services (Grafana, GitLab, and MinIO) without re-entering credentials.

---

## Prerequisites

- Basic knowledge of Linux operating system administration.
- Access to a Linux environment where you can install and configure software.
- Basic knowledge of Podman.
- Administrative access to the LDAP server you intend to integrate with Keycloak.

Now that you're familiar with the components and prerequisites, let's dive into the setup process to configure Keycloak, Grafana, MinIO, GitLab, OAuth2 Proxy, and 389ds. Follow the step-by-step instructions provided in this guide to seamlessly integrate these technologies and enhance the security and scalability of your applications.

# Keycloak

Keycloak is an open-source identity and access management tool that provides authentication and authorization for applications. It allows users to log in using Single Sign-On (SSO) and integrates with identity providers such as LDAP.

> **Note:** You have to install Keycloak on the base system, not in the container.

## System Requirements
- A 64-bit operating system (Linux-based recommended).
- At least 2 GB of RAM (4 GB recommended for production).
- Java Development Kit (JDK) 17 or later.
- Keycloak 26.1.0 (latest stable release).

---

## Step 1: Update System and Install Java
### Install OpenJDK 17:
```sh
sudo apt update
sudo apt install openjdk-17-jdk -y
```
### Explanation:
- `sudo` → Runs the command with administrator (root) privileges.
- `apt install` → Installs software using the package manager in Ubuntu/Debian.
- `openjdk-17-jdk` → Installs Java Development Kit (JDK) version 17.
- `-y` → Automatically confirms installation without asking for user input.

Keycloak is a Java-based application and requires Java to run. Installing OpenJDK 17 provides the necessary runtime and development tools for Keycloak to function properly.

### Verify Java Installation:
```sh
java -version
```
#### Expected Output:
```sh
ankit@hp-laptop:~$ java -version
openjdk version "17.0.14" 2025-01-21
OpenJDK Runtime Environment (build 17.0.14+7-Ubuntu-124.04)
OpenJDK 64-Bit Server VM (build 17.0.14+7-Ubuntu-124.04, mixed mode, sharing)
```

---

## Step 2: Set JAVA_HOME Environment Variable
### Find the Installed Java Path:
```sh
sudo update-alternatives --config java
```
#### Example Output:
```sh
Selection    Path Priority   Status
------------------------------------------------------------
* 0          /usr/lib/jvm/java-17-openjdk-amd64/bin/java   1711      auto mode
  1          /usr/lib/jvm/java-11-openjdk-amd64/bin/java   1100      manual mode
```
### Set JAVA_HOME in the System Environment:
```sh
sudo nano /etc/environment
```
Add the following line at the end of the file:
```sh
JAVA_HOME="/usr/lib/jvm/java-17-openjdk-amd64"
```
### Save the File and Apply Changes:
```sh
source /etc/environment
```
### Verify:
```sh
echo $JAVA_HOME
```
#### Output:
```sh
/usr/lib/jvm/java-17-openjdk-amd64
```

---

## Step 3: Download Keycloak 26.1.0
#### Option 1: Download Manually
Download the Keycloak-26.1.0.zip file.

#### Option 2: Use `wget`
```sh
wget https://github.com/keycloak/keycloak/releases/download/26.1.0/keycloak-26.1.0.zip
```
### Move to the Keycloak Directory:
```sh
cd Downloads/keycloak-26.1.0/
```
### Extract the Downloaded ZIP File:
```sh
unzip keycloak-26.1.0.zip
```

---

## Step 4: Create a Systemd Service File
### Create a New Service File for Keycloak:
```sh
sudo vi /etc/systemd/system/keycloak.service
```
### Add the Following Content:
```ini
[Unit]
Description=Keycloak Server
After=network.target

[Service]
User=ankit
Group=ankit
WorkingDirectory=/home/ankit/Downloads/keycloak-26.1.0
ExecStart=/home/ankit/Downloads/keycloak-26.1.0/bin/kc.sh start-dev
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal
LimitNOFILE=102400

[Install]
WantedBy=multi-user.target
```

---

## Step 5: Set Execution Permission
Ensure the Keycloak script has execution permissions:
```sh
sudo chmod +x /home/ankit/Downloads/keycloak-26.1.0/bin/kc.sh
```

---

## Step 6: Reload Systemd and Enable Keycloak Service
Run the following commands:
```sh
sudo systemctl daemon-reload
sudo systemctl enable keycloak
sudo systemctl start keycloak
```

---

## Step 7: Verify the Service Status
Check if Keycloak is running:
```sh
sudo systemctl status keycloak
```
#### Expected Output:
```sh
● keycloak.service - Keycloak Server
     Loaded: loaded (/etc/systemd/system/keycloak.service; enabled; preset: enabled)
     Active: active (running) since Sat 2025-02-08 12:54:12 IST; 9min ago
   Main PID: 37900 (java)
      Tasks: 44 (limit: 9181)
     Memory: 398.6M (peak: 493.5M)
        CPU: 1min 4.346s
     CGroup: /system.slice/keycloak.service
```

> **Note:** Since the service is enabled, Keycloak will start automatically after a system reboot.
# Keycloak GUI Setup

When you visit Keycloak for the first time, it will prompt you to set a username and password.

![Fig 1.1 - Add User](#)

## Create a Realm
Use these steps to create the first realm:
1. Open the **Keycloak Admin Console**.
2. Click the word **Master** in the top-left corner, then click **Create Realm**.
3. Enter a **project** in the **Realm Name** field.
4. Click **Create**.

![Fig 2.1 - Create Realm](#)

## Create a Client

![Fig 2.2 - Create Client](#)

### Add Suitable Client ID and Name

![Fig 2.3 - Client ID and Name](#)

### Add Capability Config

![Fig 2.4 - Capability Config](#)

### Add Login Setting

![Fig 2.5 - Login Setting](#)

After clicking the **Save** button, your client will be created.

## Create Realm Role

![Fig 3.1 - Create Role](#)  
Click on **Create Role** and add the name `default-roles-project`.

![Fig 3.2 - Assign Roles](#)  

---


To test, restart your system. If everything is set up correctly, Keycloak will be accessible at:
```sh
http://localhost:8080
```
