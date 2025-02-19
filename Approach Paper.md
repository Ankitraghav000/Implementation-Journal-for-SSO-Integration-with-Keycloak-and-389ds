# Approach Paper for SSO Integration with Keycloak and 389ds for Grafana, GitLab, and MinIO Access
## 1. Introduction

This document outlines the approach for implementing a Single Sign-On (SSO) solution using Keycloak and 389 Directory Server (389ds). The integration aims to provide seamless access to multiple services, including Grafana, GitLab, and MinIO, through a single login process.

## 2. Objectives

- **Seamless User Experience**: Enable users to access Grafana, GitLab, and MinIO with a single set of credentials.
- **Centralized Authentication**: Utilize Keycloak as the central identity provider for managing user identities and access control.
- **Enhanced Security**: Implement OAuth2 protocols to secure user.

## 3. Components Involved

- **Keycloak(Keycloak 26.1.0)**: An open-source identity and access management tool that provides SSO capabilities.
- **389 Directory Server (Version: 3.1.1)**: An LDAP server for managing user identities and authentication.
- **Grafana(grafana version 11.5.1)**: A data visualization and monitoring tool.
- **GitLab(Gitlab Ruby Gem 4.19.0)**: A web-based DevOps lifecycle tool that provides a Git repository manager.
- **MinIO (RELEASE.2025-02-03T21-03-04Z)**: A high-performance, distributed object storage system.
- **OAuth2 Proxy (oauth2-proxy v7.8.1 )**: A reverse proxy that provides authentication for web applications.

## 4. Prerequisites

- Basic knowledge of Linux operating system administration.
- Access to a Linux environment for software installation and configuration.
- Basic knowledge of podman.
- Administrative access to the LDAP server for integration with Keycloak.
## 5. Architecture Diagram:


## 5. Implementation Steps

### 5.1 Keycloak Configuration

- **Install Keycloak**: Follow the installation steps to set up Keycloak on the base system.
- **Create a Realm**: Set up a realm in Keycloak for managing users and clients.

### 5.2 Grafana Integration

- **Install Grafana**: Deploy Grafana using Podman.
- **Configure Grafana**: Set Grafana to use Keycloak as its authentication provider by enabling OAuth2.

### 5.3 MinIO Integration

- **Install MinIO**: Deploy MinIO using Podman.
- **Configure MinIO**: Set MinIO to use Keycloak for authentication, ensuring secure access to storage.

### 5.4 GitLab Integration

- **Install GitLab**: Deploy GitLab using Podman.
- **Configure GitLab**: Set GitLab to use with Keycloak for user authentication.

### 5.5 OAuth2 Proxy Setup

- **Install OAuth2 Proxy**: Download and configure OAuth2 Proxy to handle authentication requests.
- **Configure Nginx**: Set up Nginx as a reverse proxy for Grafana, MinIO, GitLab, and OAuth2 Proxy.

### 5.6 389ds Setup

- **Install 389ds**: Deploy 389 Directory Server using Podman.
- **Configure LDAP**: Set up the LDAP structure and add users for authentication.

## 6. Testing and Validation

- **Access Services**: Test the SSO functionality by accessing Grafana, GitLab, and MinIO using Keycloak credentials.
- **Verify User Authentication**: Ensure that users can log in once and access all services without re-entering credentials.

## 7. Conclusion

The implementation of SSO using Keycloak and 389ds will enhance user experience by providing seamless access to multiple services with a single login. This approach not only simplifies user management but also strengthens security through centralized authentication.
