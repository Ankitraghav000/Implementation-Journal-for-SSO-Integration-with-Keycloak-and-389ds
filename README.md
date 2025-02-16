Implementation Journal for SSO Integration with Keycloak and 389ds
==================================================================

**Date:** February 16, 2025

**Overview**
------------

This journal documents the implementation of Single Sign-On (SSO) integration using Keycloak and 389 Directory Server (389ds) for seamless access to Grafana, GitLab, and MinIO. The integration allows users to authenticate once and gain access to all three services without re-entering credentials.

**Setup Process**
-----------------

### **1\. Keycloak Installation**

*   **System Requirements:**
    
    *   64-bit Linux OS
        
    *   Minimum 2 GB RAM
        
    *   Java Development Kit (JDK) 17 or later
        
    *   Keycloak version 26.1.0
        
```
# Update the system and install OpenJDK 17
sudo apt update
sudo apt install openjdk-17-jdk -y

# Verify Java installation
java -version

# Set JAVA_HOME environment variable
sudo update-alternatives --config java
sudo nano /etc/environment
# Add the following line at the end of the file
JAVA_HOME="/usr/lib/jvm/java-17-openjdk-amd64"
# Save and apply changes
source /etc/environment
echo $JAVA_HOME

# Download Keycloak 26.1.0
wget https://github.com/keycloak/keycloak/releases/download/26.1.0/keycloak-26.1.0.zip

# Extract the downloaded ZIP file
unzip keycloak-26.1.0.zip

# Create a systemd service file for Keycloak
sudo vi /etc/systemd/system/keycloak.service
# Add the following content
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

# Set execution permission
sudo chmod +x /home/ankit/Downloads/keycloak-26.1.0/bin/kc.sh

# Reload systemd and enable Keycloak service
sudo systemctl daemon-reload
sudo systemctl enable keycloak
sudo systemctl start keycloak

# Verify the service status
sudo systemctl status keycloak
```
### **2\. Keycloak GUI Setup**

*   Access the Keycloak Admin Console at **http://localhost:8080**.
    
# Creating the First Realm in Keycloak

1. Open the **Keycloak Admin Console**.  
2. Click on **master** in the top-left corner, then select **Create Realm**.  
3. Enter a suitable name in the **Realm Name** field.  
4. Click **Create**.  

![Image](https://github.com/user-attachments/assets/7d78fa3a-e3f9-4851-993b-3307d2361e59) 

---

## Creating a Client  

1. Navigate to the **Clients** section.  
2. Click **Create Client**.  

![Image](https://github.com/user-attachments/assets/438f72b2-cf5d-4582-9893-d884557e404d) 

3. Enter a **Client ID** and **Name**.  
![Image](https://github.com/user-attachments/assets/2f345b27-ca00-4942-aa5b-c54b5ce2a5b8) 

4. Configure the **Capability Settings** as needed.  

![Image](https://github.com/user-attachments/assets/0ee88829-725f-4007-a0b0-60817e4f5524)
5. Adjust the **Login Settings** if required.  

![Image](https://github.com/user-attachments/assets/1a481278-608c-4349-93d0-3325d0233923)
6. Click **Save** to create the client.  

---

## Creating a Realm Role  

1. Go to the **Roles** section.  
2. Click **Create Role**.  
3. Enter `default-roles-project` as the role name.  

![Image](https://github.com/user-attachments/assets/8e4f0ede-43b0-4a3b-a717-0db8a2515b75)
4. Assign the necessary roles to the created role.  

![Image](https://github.com/user-attachments/assets/0e2faaa7-6f58-4d39-9d88-e88ea5c28eed)

After completing these steps, your realm, client, and roles will be successfully configured in Keycloak. 

    

### **3\. Grafana Integration**
```
# Install Podman
sudo apt update
sudo apt install -y podman

# Pull the Grafana image
podman pull docker.io/grafana/grafana

# Run the Grafana container
podman run -d --name=grafana -p 3000:3000 grafana/grafana
```
### Access Grafana
Open your browser and go to http://localhost:3000         

### **4\. MinIO Integration**
```
# Pull the MinIO image
podman pull docker.io/minio/minio

# Create a directory for data storage
mkdir -p ~/minio-data

# Run the MinIO container

podman run -d -p 9000:9000 -p 9001:9001 \
  --name minio \
  -v ~/minio-data:/data \
  -e "MINIO_ROOT_USER=admin" \
  -e "MINIO_ROOT_PASSWORD=admin123" \
  minio/minio server /data --console-address ":9001"
```
### Access MinIO Console
Open your browser and go to http://localhost:9001    

### **5\. GitLab Integration**

```
# Pull the GitLab image
podman pull docker.io/gitlab/gitlab-ce:latest

# Create persistent volumes for GitLab data
mkdir -p /srv/gitlab/config /srv/gitlab/logs /srv/gitlab/data
sudo chmod -R 777 /srv/gitlab

# Run the GitLab container
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
### Access GitLab Web UI
Open your browser and go to http://localhost:8081    

### **6\. OAuth2 Proxy Installation**
```
# Download OAuth2 Proxy
wget https://github.com/oauth2-proxy/oauth2-proxy/releases/download/v9.2.0/oauth2-proxy-v9.2.0.linux-amd64.tar.gz

# Extract the archive
tar -xvzf oauth2-proxy-v9.2.0.linux-amd64.tar.gz

# Move binary to system path
sudo mv oauth2-proxy-v9.2.0.linux-amd64/oauth2-proxy /usr/local/bin/oauth2-proxy

# Verify installation
oauth2-proxy --version
```   
*  configuration
```  # Create a configuration directory and file

sudo mkdir -p /etc/oauth2_proxy
sudo nano /etc/oauth2_proxy/oauth2-proxy.cfg

# Add the following configurations

http_address="0.0.0.0:4180"
cookie_secret="0HjVrbq01u6SHzdkl2uk+LHgQ0ixTfC="
email_domains="gmail.com"
cookie_secure="false"
cookie_domains=[".keen.com"]
whitelist_domains = [".keen.com"]
reverse_proxy="true"
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
### Run OAuth2 Proxy
```
oauth2-proxy --config=/etc/oauth2_proxy/oauth2-proxy.cfg
```
### **7\. Nginx Setup**
### Setup a reverse proxy for Nginx
```
 vim /etc/nginx/sites-available/grafna 
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
          proxy_set_header Host      $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-Uri $request_uri;

          proxy_pass http://oauth2-proxy/;
       }

}
```
### Setup a reverse proxy for MinIO
```
 vim /etc/nginx/sites-available/minio
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
          proxy_set_header Host      $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-Uri $request_uri;

          proxy_pass http://oauth2-proxy/;
       }
```

### Setup a reverse proxy for Gitlab
```
vim /etc/nginx/sites-available/gitlab

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
          proxy_set_header Host      $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-Uri $request_uri;

          proxy_pass http://oauth2-proxy/;
       }

}
```
### Setup a reverse proxy for keycloak:

```
vim /etc/nginx/sites-available/keycloak

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
### Setup a reverse proxy for OAuth2 Proxy:
```
vim /etc/nginx/sites-available/oauth2-proxy

upstream oauth2-proxy {
        server 127.0.0.1:4180;
  }


server {
    listen 80;
    server_name oauth.Keen.com;

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
### Create symbolic links :
```
sudo ln -s /etc/nginx/sites-available/grafana /etc/nginx/sites-enabled/

sudo ln -s /etc/nginx/sites-available/minio /etc/nginx/sites-enabled/

sudo ln -s /etc/nginx/sites-available/gitlab /etc/nginx/sites-enabled/

sudo ln -s /etc/nginx/sites-available/keycloak /etc/nginx/sites-enabled/

sudo ln -s /etc/nginx/sites-available/oauth2-proxy /etc/nginx/sites-enabled/

```
- Now , restart the nginx
```
sudo systemctl restart nginx.service
```





### **8\. 389 Directory Server Setup**
```
# Create a DS volume
podman volume create ds-data

# Create the 389 Directory Server container
podman run -d \
  --name ds \
  -e DS_DM_PASSWORD=redhat \
  -p 3389:3389 \
  -p 3636:3636 \
  -v ds-data:/data \
  -it \
  docker.io/389ds/dirsrv

# Check running containers
podman ps -a

# Verify LDAP connection
podman exec ds ldapsearch \
  -H ldap://localhost.localdomain:3389 \
  -D "cn=Directory Manager" \
  -w redhat \
  -x \
  -b "" \
  -s base

# View container logs
podman logs -f ds

# Access the LDAP container shell
podman exec -it ds bash

# Check if LDAP is working
ldapsearch -H ldap://localhost:3389 -D "cn=Directory Manager" -w redhat

# Create a new LDAP database
dsconf localhost backend create --be-name exampleDB --suffix dc=example,dc=com

# Add a base structure in LDAP
vi base.ldif
# Add the following content
dn: dc=example,dc=com
objectClass: top
objectClass: domain
dc: example

dn: ou=Users,dc=example,dc=com
objectClass: organizationalUnit
ou: Users

# Add the base structure to LDAP
ldapadd -H ldap://localhost:3389 -D "cn=Directory Manager" -w redhat -f base.ldif

# Add a user to LDAP
vi user2.ldif
# Add the following content
dn: uid=user2,ou=Users,dc=example,dc=com
objectClass: inetOrgPerson
cn: User two
sn: two
uid: user2
userPassword: user2

# Add the user to LDAP
ldapadd -H ldap://localhost:3389 -D "cn=Directory Manager" -w redhat -f user2.ldif

# Search for a user in LDAP
ldapsearch -H ldap://localhost:3389 -D "cn=Directory Manager" -w redhat "uid=user2"    
```
**Testing and Verification**
----------------------------

*   Accessed Grafana, GitLab, and MinIO.
    
*   Verified that users could log in once and access all services without re-entering credentials.
    
*   Confirmed that the LDAP integration with Keycloak was functioning correctly.
    

**Conclusion**
--------------

The SSO integration with Keycloak and 389ds for Grafana, GitLab, and MinIO was successfully implemented. The setup enhances user experience by providing seamless access across multiple services with a single login. Future considerations include monitoring and maintaining the security of the integrated services.
