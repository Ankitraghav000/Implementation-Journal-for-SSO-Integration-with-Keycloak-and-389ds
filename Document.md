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

1. Access `grafana.keen.com` to sign in using SSO credentials.
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
# Grafana

## Introduction to Grafana
Grafana is an open-source data visualization and monitoring tool that helps create dashboards and graphs from different data sources. It is widely used for analytics and monitoring system performance.

---

## Steps to Run Grafana in a Podman Container

### 1. Install Podman (if not already installed)
Before running Grafana, you need to install Podman on your system.

#### Update your system:
```sh
sudo apt update
```

#### Install Podman:
```sh
sudo apt install -y podman
```

#### Verify Podman installation:
```sh
podman --version
```
Expected output:
```sh
podman version 4.9.3
```
This command checks whether Podman is installed correctly.

---

### 2. Pull the Grafana Podman Image
Download the official Grafana image from Docker Hub using Podman.
```sh
podman pull docker.io/grafana/grafana
```
This command fetches the latest Grafana image from the internet.

---

### 3. Run the Grafana Container
Start a Grafana container using the following command:
```sh
podman run -d --name=grafana -p 3000:3000 grafana/grafana
```
#### Explanation of the Command:
- `podman run` → Runs a new Podman container.
- `-d` → Runs the container in the background (detached mode).
- `--name=grafana` → Assigns the container a name (`grafana`).
- `-p 3000:3000` → Maps port `3000` on your machine to port `3000` in the container, allowing web access.
- `grafana/grafana` → Specifies the image to use.

---

### 4. Access Grafana
Once the container is running, you can access Grafana through your web browser:
```sh
http://localhost:3000
```

#### Default Login Credentials:
- **Username:** `admin`
- **Password:** `admin`

After logging in, you can change the password.

# MinIO

MinIO is a high-performance, distributed object storage system. It is API-compatible with Amazon S3 (Simple Storage Service) and is designed for large-scale data storage.

---

## Step 1: Pull the MinIO Docker Image
Download the latest MinIO image from Docker Hub:
```sh
podman pull docker.io/minio/minio
```
This command fetches the latest MinIO image from Docker Hub.

---

## Step 2: Create a Directory for Data Storage
Create a directory on your host machine to store MinIO data:
```sh
mkdir -p ~/minio-data
```
#### Explanation:
- `mkdir -p ~/minio-data` → Creates a directory named `minio-data` in the user's home directory (`~`).
- The `-p` flag ensures that the directory is created without errors if it already exists.

---

## Step 3: Run the MinIO Docker Container
Start the MinIO container with the following command:
```sh
docker run -d -p 9000:9000 -p 9001:9001 \
  --name minio \
  -v ~/minio-data:/data \
  -e "MINIO_ROOT_USER=admin" \
  -e "MINIO_ROOT_PASSWORD=admin123" \
  minio/minio server /data --console-address ":9001"
```

### Command Breakdown:
- `docker run -d` → Runs the container in detached mode (background execution).
- `-p 9000:9000` → Maps port `9000` on the host machine to port `9000` inside the container (MinIO API endpoint).
- `-p 9001:9001` → Maps port `9001` on the host machine to port `9001` inside the container (MinIO web console).
- `--name minio` → Assigns the name `minio` to the container.
- `-v ~/minio-data:/data` → Mounts the `minio-data` directory from the host machine to `/data` inside the container.
- `-e "MINIO_ROOT_USER=admin"` → Sets the MinIO admin username to `admin`.
- `-e "MINIO_ROOT_PASSWORD=admin123"` → Sets the MinIO admin password to `admin123`.
- `minio/minio server /data --console-address ":9001"` → Starts the MinIO server using the `/data` directory and assigns the web console to port `9001`.

---

## Step 4: Access the MinIO Console
Once the container is running, access MinIO’s web interface:

1. Open your browser and go to:
   ```sh
   http://localhost:9001
   ```
2. Replace `<your-server-ip>` with the actual IP address of your server if needed.
3. Log in using the credentials specified in the container command:
   - **Username:** `admin`
   - **Password:** `admin123`

# GitLab

## Step 1: Pull GitLab Image
Pull the GitLab image from Docker Hub:
```sh
podman pull docker.io/gitlab/gitlab-ce:latest
```

---

## Step 2: Create a Persistent Volume
To store GitLab’s data, create a volume:
```sh
mkdir -p /srv/gitlab/config /srv/gitlab/logs /srv/gitlab/data
sudo chmod -R 777 /srv/gitlab
```

---

## Step 3: Run GitLab Container
Run the GitLab container with the necessary ports and environment variables:
```sh
podman run --detach \
  --hostname gitlab.local \
  --publish 8081:80 \
  --publish 8443:443 \
  --publish 2222:22 \
  --name gitlab \
  --restart always \
  --shm-size 256m \
  --volume /srv/gitlab/config:/etc/gitlab \
  --volume /srv/gitlab/logs:/var/log/gitlab \
  --volume /srv/gitlab/data:/var/opt/gitlab \
  docker.io/gitlab/gitlab-ce:latest
```

### Explanation of the Command:
- `podman run --detach` → Runs the container in the background (detached mode).
- `--hostname gitlab.local` → Sets the hostname to `gitlab.local`.
- `--publish 8081:80` → Maps GitLab's web UI to port `8081` on the host.
- `--publish 8443:443` → Exposes port `8443` for HTTPS (SSL).
- `--publish 2222:22` → Maps GitLab's SSH access to port `2222`.
- `--name gitlab` → Names the container `gitlab`.
- `--restart always` → Ensures the container automatically restarts after a system reboot.
- `--shm-size 256m` → Allocates 256MB shared memory to prevent memory-related issues.
- `--volume /srv/gitlab/config:/etc/gitlab` → Mounts the configuration directory for persistence.
- `--volume /srv/gitlab/logs:/var/log/gitlab` → Mounts the logs directory for tracking errors and activity.
- `--volume /srv/gitlab/data:/var/opt/gitlab` → Mounts the data directory for storing GitLab user data.
- `docker.io/gitlab/gitlab-ce:latest` → Pulls and runs the latest GitLab Community Edition image.

---

## Step 4: Verify Container is Running
Check the status of the running container:
```sh
podman ps
```

---

## Step 5: Access GitLab Web UI
1. Open the following URL in your browser:
   ```sh
   http://localhost:8081
   ```
2. **Username:** `root`
3. **Password:** Follow the steps below to retrieve the password from the container:
   ```sh
   podman exec -it gitlab bash
   cat /etc/gitlab/initial_root_password
   ```
# OAuth2 Proxy

OAuth2 Proxy is a tool that enables authentication for web applications by integrating with identity providers like Keycloak. This guide provides step-by-step instructions to install and configure OAuth2 Proxy for securing your web applications.

---

## Installation Steps

### Step 1: Install OAuth2 Proxy

#### 1.1 Download OAuth2 Proxy
Download the latest release from the official GitHub page or use the following command:
```sh
wget https://github.com/oauth2-proxy/oauth2-proxy/releases/download/v9.2.0/oauth2-proxy-v9.2.0.linux-amd64.tar.gz
```

#### 1.2 Extract the Archive
```sh
tar -xvzf oauth2-proxy-v9.2.0.linux-amd64.tar.gz
```

#### 1.3 Move Binary to System Path
```sh
sudo mv oauth2-proxy-v9.2.0.linux-amd64/oauth2-proxy /usr/local/bin/oauth2-proxy
```

#### 1.4 Verify Installation
```sh
oauth2-proxy --version
```

---

## Step 2: Configure OAuth2 Proxy

### 2.1 Create a Configuration File
Create a configuration directory and file:
```sh
sudo mkdir -p /etc/oauth2_proxy
sudo nano /etc/oauth2_proxy/oauth2-proxy.cfg
```

### Add the following configurations:
```ini
http_address="0.0.0.0:4180"
cookie_secret="0HjVrbq01u6SHzdkl2uk+LHgQ0ixTfC="
email_domains="gmail.com"
cookie_secure="false"
cookie_domains=[".keen.com"]
whitelist_domains = [".keen.com"]
grafana.keen.comreverse_proxy="true"
session_cookie_minimal="false"
cookie_expire="0h5m0s"
insecure_oidc_allow_unverified_email="true"
provider = "keycloak-oidc"
oidc_issuer_url = "http://keycloak.keen.com/realms/project"
client_id = "grafana-client"
client_secret = "tW2c0rL26Z0kP74lxLVvF6c18eEwCMeI"
redirect_url = "http://oauth.keen.com/oauth2/callback"
scope="openid email"
provider_display_name="Ankit-SSO"
pass_access_token="true"
set_xauthrequest="true"
pass_user_headers="true"
upstreams=["http://127.0.0.1:80"]
```

### Explanation of Configuration:
- `http_address="0.0.0.0:4180"`: Specifies the address and port where OAuth2 Proxy will listen for incoming requests.
- `cookie_secret="0HjVrbq01u6SHzdkl2uk+LHgQ0ixTfC="`: Secret key used to encrypt session cookies. Generate a secure key using:
  ```sh
  openssl rand -base64 16
  ```
- `email_domains="gmail.com"`: Only allows Gmail accounts to log in.
- `cookie_secure="false"`: Allows cookies to be sent over both HTTP and HTTPS.
- `cookie_domains=[".keen.com"]`: Defines the domain for which the cookie is valid.
- `whitelist_domains = [".keen.com"]`: Lists allowed domains for OAuth2 Proxy access.
- `reverse_proxy="true"`: Indicates OAuth2 Proxy is behind a reverse proxy (e.g., Nginx).
- `session_cookie_minimal="false"`: Stores more session information in cookies.
- `cookie_expire="0h5m0s"`: Sets cookie expiration time to 5 minutes.
- `insecure_oidc_allow_unverified_email="true"`: Allows users with unverified emails to log in (not recommended for production).
- `provider = "keycloak-oidc"`: Specifies Keycloak as the identity provider using OpenID Connect (OIDC).
- `oidc_issuer_url = "http://keycloak.keen.com/realms/project"`: URL of the Keycloak server for authentication.
- `client_id = "grafana-client"`: Client ID registered in Keycloak.
- `client_secret = "tW2c0rL26Z0kP74lxLVvF6c18eEwCMeI"`: Client secret for authentication (should be kept secure).
- `redirect_url = "http://oauth.keen.com/oauth2/callback"`: OAuth2 Proxy callback URL after authentication.
- `scope="openid email"`: Requested authentication scopes.
- `provider_display_name="Ankit-SSO"`: Friendly display name for authentication provider.
- `pass_access_token="true"`: Passes OAuth access token to the upstream server (e.g., Grafana).
- `set_xauthrequest="true"`: Sets `X-Auth-Request-User` header for user identification.
- `pass_user_headers="true"`: Ensures user-related headers (like email) are passed to the upstream server.
- `upstreams=["http://127.0.0.1:80"]`: Specifies the backend application OAuth2 Proxy will protect (e.g., Grafana on localhost).

> **Note:** Change the client ID, client secret, cookie secret, and other credentials according to your Keycloak configuration.

---

## Step 4: Run OAuth2 Proxy
Start OAuth2 Proxy using:
```sh
oauth2-proxy --config=/etc/oauth2_proxy/oauth2-proxy.cfg
```

---

## Access OAuth2 Proxy
Visit:
```sh
http://localhost:4180
```
# Nginx Installation

## Step 1: Install Nginx
```sh
sudo apt update
sudo apt install nginx
```

After installation, you can check if Nginx is running:
```sh
sudo systemctl status nginx
```

---

## Add Configuration in Nginx
### Setup a Reverse Proxy for Grafana, MinIO, and GitLab

This Nginx configuration sets up a reverse proxy for Grafana, MinIO, and GitLab, with OAuth2 authentication handled by an OAuth2 Proxy.

### Configure Grafana Reverse Proxy
#### Path: `/etc/nginx/sites-available/grafana`
```nginx
upstream grafana_servers {
    server 127.0.0.1:3000;
}

server {
    listen 80;
    server_name grafana.keen.com;

    auth_request /internal-auth/oauth2/auth;
    error_page 401 =403 http://oauth.keen.com/oauth2/sign_in?rd=$scheme://$host$request_uri;

    location / {
        proxy_pass http://grafana_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
    }

    location /internal-auth/ {
        internal;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Uri $request_uri;
        proxy_pass http://oauth2-proxy/;
    }
}
```

---

### Configure MinIO Reverse Proxy
#### Path: `/etc/nginx/sites-available/minio`
```nginx
upstream minio_servers {
    server 127.0.0.1:9001;
}

server {
    listen 80;
    server_name minio.keen.com;

    auth_request /internal-auth/oauth2/auth;
    error_page 401 =403 http://oauth.keen.com/oauth2/sign_in?rd=$scheme://$host$request_uri;

    location / {
        proxy_pass http://minio_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
    }

    location /internal-auth/ {
        internal;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Uri $request_uri;
        proxy_pass http://oauth2-proxy/;
    }
}
```

---

### Configure GitLab Reverse Proxy
#### Path: `/etc/nginx/sites-available/gitlab`
```nginx
upstream gitlab_servers {
    server 127.0.0.1:8081;
}

server {
    listen 80;
    server_name gitlab.keen.com;

    auth_request /internal-auth/oauth2/auth;
    error_page 401 =403 http://oauth.keen.com/oauth2/sign_in?rd=$scheme://$host$request_uri;

    location / {
        proxy_pass http://gitlab_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
    }

    location /internal-auth/ {
        internal;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Uri $request_uri;
        proxy_pass http://oauth2-proxy/;
    }
}
```

---

## Setup a Reverse Proxy for Keycloak
This Nginx configuration sets up a reverse proxy for Keycloak, forwarding requests from `keycloak.keen.com` to `localhost:8080`.

#### Path: `/etc/nginx/sites-available/keycloak`
```nginx
server {
    listen 80;
    server_name keycloak.keen.com;
    
    location / {
        proxy_pass http://localhost:8080; # Forward OAuth-specific requests
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Uri $request_uri;
    }
}
```
# Setup a Reverse Proxy for OAuth2 Proxy

This Nginx configuration sets up a reverse proxy for OAuth2 Proxy, forwarding requests from `oauth.keen.com` to `127.0.0.1:4180`.

#### Path: `/etc/nginx/sites-available/oauth2-proxy`
```nginx
upstream oauth2-proxy {
    server 127.0.0.1:4180;
}

server {
    listen 80;
    server_name oauth.keen.com;

    location / {
        proxy_pass http://oauth2-proxy;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Uri $request_uri;
    }
}
```

---

## Create Symbolic Links
These commands create symbolic links (shortcuts) from the configuration files in `/etc/nginx/sites-available/` to `/etc/nginx/sites-enabled/`. This enables Nginx to load these configurations, activating the sites (Grafana, Keycloak, OAuth2 Proxy, etc.) when Nginx restarts. Without these links, Nginx won’t recognize or use these configurations!

```sh
sudo ln -s /etc/nginx/sites-available/grafana /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/minio /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/gitlab /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/keycloak /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/oauth2-proxy /etc/nginx/sites-enabled/
```

---

## Restart Nginx
After creating symbolic links, restart the Nginx service:
```sh
sudo systemctl restart nginx.service
```

---

## Update the Host File
Add the following entries in the host file to map domain names to local addresses.

#### Path: `/etc/hosts`
```sh
127.0.0.1 grafana.keen.com
127.0.0.1 oauth.keen.com
127.0.0.1 keycloak.keen.com
127.0.0.1 minio.keen.com
127.0.0.1 gitlab.keen.com
```
# 389 Directory Server (389ds)

389 Directory Server (389ds) is a robust, open-source LDAP server for managing user identities and authentication. It offers high scalability, security, and ease of integration for enterprise environments.

---

## 1. Create a DS Volume
Create a persistent storage volume for 389 Directory Server data:
```sh
podman volume create ds-data
```

---

## 2. Create the 389 Directory Server Container
Run the container with the necessary configuration:
```sh
podman run -d \
--name ds \
-e DS_DM_PASSWORD=redhat \
-p 3389:3389 \
-p 3636:3636 \
-v ds-data:/data \
-it \
docker.io/389ds/dirsrv
```

---

## 3. Check Running Containers
Verify if the container is running:
```sh
podman ps -a
```
This lists all running and stopped containers.

---

## 4. Verify LDAP Connection
Execute the following command to check the LDAP connection:
```sh
podman exec ds ldapsearch \
-H ldap://localhost.localdomain:3389 \
-D "cn=Directory Manager" \
-w redhat \
-x \
-b "" \
-s base
```

---

## 5. View Container Logs
Monitor the logs of the running container:
```sh
podman logs -f ds
```

---

## 6. Access the LDAP Container Shell
To access the shell inside the LDAP container:
```sh
podman exec -it ds bash
```

---

## 7. Check if LDAP is Working
Run the following command to test LDAP connectivity:
```sh
ldapsearch -H ldap://localhost:3389 -D "cn=Directory Manager" -w redhat
```

---

## 8. Create a New LDAP Database
Execute the command below to create a new backend database in LDAP:
```sh
dsconf localhost backend create --be-name exampleDB --suffix dc=example,dc=com
```

---

## 9. Add a Base Structure in LDAP
### Create `base.ldif`
```sh
vi base.ldif
```
Add the following content:
```ldif
dn: dc=example,dc=com
objectClass: top
objectClass: domain
dc: example

dn: ou=Users,dc=example,dc=com
objectClass: organizationalUnit
ou: Users
```
### Add the Base Structure:
```sh
ldapadd -H ldap://localhost:3389 -D "cn=Directory Manager" -w redhat -f base.ldif
```

---

## 10. Add a User to LDAP
### Create `user2.ldif`
```sh
vi user2.ldif
```
Add the following content:
```ldif
dn: uid=user2,ou=Users,dc=example,dc=com
objectClass: inetOrgPerson
cn: User Two
sn: Two
uid: user2
userPassword: user2
```
### Add the User to LDAP:
```sh
ldapadd -H ldap://localhost:3389 -D "cn=Directory Manager" -w redhat -f user2.ldif
```

---

## 11. Search for a User in LDAP
To search for a user in LDAP:
```sh
ldapsearch -H ldap://localhost:3389 -D "cn=Directory Manager" -w redhat "uid=user1"
```
