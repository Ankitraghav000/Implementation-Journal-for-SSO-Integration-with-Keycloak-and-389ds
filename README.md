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
    
*   Create a new realm named "project".
    

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
oauth2-proxy --config=/etc/oauth2_proxy/oauth2-proxy.cfg

### **7\. Nginx Setup**

*   bashCopy code1# Install Nginx2sudo apt update3sudo apt install nginx45# Check if Nginx is running6sudo systemctl status nginx
    
*   bashCopy code1# Create configuration files for Grafana, MinIO, GitLab, Keycloak, and OAuth2 Proxy2sudo nano /etc/nginx/sites-available/grafana3# Add the following content4upstream grafana\_servers {5 server 127.0.0.1:3000;6}78server {9 listen 80;10 server\_name grafana.keen.com;1112 auth\_request /internal-auth/oauth2/auth;1314 error\_page 401 =403 http://oauth.keen.com/oauth2/sign\_in?rd=$scheme://$host$request\_uri;1516 location / {17 proxy\_pass http://grafana\_servers;18 proxy\_set\_header Host $host;19 proxy\_set\_header X-Real-IP $remote\_addr;20 proxy\_set\_header X-Forwarded-For $proxy\_add\_x\_forwarded\_for;21 proxy\_set\_header X-Forwarded-Proto $scheme;22 proxy\_connect\_timeout 5s;23 proxy\_read\_timeout 60s;24 proxy\_send\_timeout 60s;25 proxy\_http\_version 1.1;26 proxy\_set\_header Upgrade $http\_upgrade;27 proxy\_set\_header Connection 'upgrade';28 }2930 location /internal-auth/ {31 internal;32 proxy\_set\_header Host $host;33 proxy\_set\_header X-Real-IP $remote\_addr;34 proxy\_set\_header X-Forwarded-Uri $request\_uri;35 proxy\_pass http://oauth2-proxy/;36 }37}3839# Repeat similar steps for MinIO, GitLab, Keycloak, and OAuth2 Proxy configurations
    
*   bashCopy code1sudo ln -s /etc/nginx/sites-available/grafana /etc/nginx/sites-enabled/2sudo ln -s /etc/nginx/sites-available/minio /etc/nginx/sites-enabled/3sudo ln -s /etc/nginx/sites-available/gitlab /etc/nginx/sites-enabled/4sudo ln -s /etc/nginx/sites-available/keycloak /etc/nginx/sites-enabled/5sudo ln -s /etc/nginx/sites-available/oauth2-proxy /etc/nginx/sites-enabled/67# Restart Nginx8sudo systemctl restart nginx.service
    

### **8\. 389 Directory Server Setup**

*   bashCopy code1# Create a DS volume2podman volume create ds-data34# Create the 389 Directory Server container5podman run -d \\6 --name ds \\7 -e DS\_DM\_PASSWORD=redhat \\8 -p 3389:3389 \\9 -p 3636:3636 \\10 -v ds-data:/data \\11 -it \\12 docker.io/389ds/dirsrv1314# Check running containers15podman ps -a1617# Verify LDAP connection18podman exec ds ldapsearch \\19 -H ldap://localhost.localdomain:3389 \\20 -D "cn=Directory Manager" \\21 -w redhat \\22 -x \\23 -b "" \\24 -s base2526# View container logs27podman logs -f ds2829# Access the LDAP container shell30podman exec -it ds bash3132# Check if LDAP is working33ldapsearch -H ldap://localhost:3389 -D "cn=Directory Manager" -w redhat3435# Create a new LDAP database36dsconf localhost backend create --be-name exampleDB --suffix dc=example,dc=com3738# Add a base structure in LDAP39vi base.ldif40# Add the following content41dn: dc=example,dc=com42objectClass: top43objectClass: domain44dc: example4546dn: ou=Users,dc=example,dc=com47objectClass: organizationalUnit48ou: Users4950# Add the base structure to LDAP51ldapadd -H ldap://localhost:3389 -D "cn=Directory Manager" -w redhat -f base.ldif5253# Add a user to LDAP54vi user2.ldif55# Add the following content56dn: uid=user2,ou=Users,dc=example,dc=com57objectClass: inetOrgPerson58cn: User two59sn: two60uid: user261userPassword: user26263# Add the user to LDAP64ldapadd -H ldap://localhost:3389 -D "cn=Directory Manager" -w redhat -f user2.ldif6566# Search for a user in LDAP67ldapsearch -H ldap://localhost:3389 -D "cn=Directory Manager" -w redhat "uid=user2"
    

**Testing and Verification**
----------------------------

*   Accessed Grafana, GitLab, and MinIO using the configured SSO credentials.
    
*   Verified that users could log in once and access all services without re-entering credentials.
    
*   Confirmed that the LDAP integration with Keycloak was functioning correctly.
    

**Conclusion**
--------------

The SSO integration with Keycloak and 389ds for Grafana, GitLab, and MinIO was successfully implemented. The setup enhances user experience by providing seamless access across multiple services with a single login. Future considerations include monitoring and maintaining the security of the integrated services.
