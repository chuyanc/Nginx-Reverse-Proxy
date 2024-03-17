This is the documentation of how to set up Keycloak and Vault for authentication and authorization of a virtual machine cluster for a group.

## Task Description
### Goal
We aim to create a Group VM Single Sign-On (SSO) system for the team, ensuring basic authentication and authorization. This solution should be flexible for user authentication, easy for administrators, and maintain a balance between simplicity and security. The goal is to swiftly integrate the SSO system within the team, unblocking projects that need VM access and fostering a secure, user-friendly VM environment.

### Reference
[1. Vault + KeyCloak OIDC](https://drpdishant.medium.com/identity-based-ssh-with-vault-and-keycloak-part-1-3-47ab2181ceae)

[2. Vault SSH Certificate](https://drpdishant.medium.com/identity-based-ssh-with-vault-and-keycloak-part-2-3-signed-ssh-certificate-c9fb2c4dde64)

### Environment Setting Up
1. We are using **Ubuntu 22.04** instead of the Virtual Box + Vagrant in reference 1, so some steps might be different. But I'll refer some of the gif in the reference for the operations on user interface.

2. Please ensure **at least 4GiB of RAM** for your VM instance (like t2.medium or t2.large for Amazon EC2)

## Solution
### 1. Vault + KeyCloak OIDC
#### 1.1 Install and Start Vault
##### 1) Vault installation
```
$ curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
$ sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
$ sudo apt-get update
$ sudo apt-get install -y vault
$ sudo systemctl start vault
$ sudo systemctl enable vault
```

##### 2) Vault.service configuration
Open /lib/systemd/system/vault.service and configure it like this, in order to tell Vault where to look for trusted CA certificates (will be used later):
```
[service] 
Environment="SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt"
Environment="SSL_CERT_DIR=/etc/ssl/certs"
```

Then reload the systemd and restart Vault:
```
$ sudo systemctl daemon-reload
$ sudo systemctl restart vault
```

#### 1.2 Get Vault token and keys, unseal, then log in to Vault
Access Vault UI via: https://{your IP Address}:8200/. Follow the UI operation steps of Initialize, Unseal and Login in Reference 1.

#### 1.3 Initialize and Run Keycloak
##### 1) Self-sign certificate
```
$ openssl req -x509 -nodes -newkey rsa:2048 -keyout myserver.key -out myserver.crt -days 365 -subj "/CN=myserver" -addext "subjectAltName = IP: {your Public IP}"
$ chmod 755 myserver.key
```
##### 2) Update CA
```
$ sudo cp myserver.crt /usr/local/share/ca-certificates/
$ sudo update-ca-certificates
```
#### 3) Install Docker
```
$ curl -sSL https://get.docker.com/ | sudo bash
```
#### 4) Run Keycloak
```
$ sudo docker run \
  --name keycloak \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=password \
  -e KC_HTTPS_CERTIFICATE_FILE=/opt/keycloak/conf/myserver.crt \
  -e KC_HTTPS_CERTIFICATE_KEY_FILE=/opt/keycloak/conf/myserver.key \
  -v $PWD/myserver.crt:/opt/keycloak/conf/myserver.crt \
  -v $PWD/myserver.key:/opt/keycloak/conf/myserver.key \
  -p 8443:8443 \
  quay.io/keycloak/keycloak:19.0.1 \
  start-dev
```
If it's not your first time running Keycloak on this machine, and there exists a 'keycloak' container, directly use
```
$ sudo docker start keycloak
```

#### 1.4 Configure Vault client and create testing user in Keycloak
Access Keycloak UI via: https://{your IP Address}:8443/, and follow the UI operation steps in Reference 1 for set vault client on keycloak and create test user.

#### 1.5 Access Vault CLI
```
$ export VAULT_SKIP_VERIFY=true
$ echo 'export VAULT_SKIP_VERIFY=true' >> ~/.bashrc
$ source ~/.bashrc
$ export VAULT_TOKEN={your token}
$ export VAULT_ADDR='https://127.0.0.1:8200'
$ vault login $VAULT_TOKEN
```

#### 1.6 Create Admin Policy
Open a new file admin.hcl, and write:
```
path "*" {
    capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}
```
then
```
$ vault policy write admin admin.hcl
```

#### 1.7 Configure OIDC Auth
```
$ vault auth enable oidc

$ export KC_DOMAIN="https://{your ip addr}:8443/realms/master"
$ export KC_CLIENT_ID=vault
$ export KC_CLIENT_SECRET={your-client-secret}
$ vault write auth/oidc/config \
        oidc_discovery_url="$KC_DOMAIN" \
        oidc_client_id="$KC_CLIENT_ID" \
        oidc_client_secret="$KC_CLIENT_SECRET" \
        default_role="default"
```
(*if doesn’t work, use the following command to check the vault logs:
```
$ sudo journalctl -u vault.service -n 50
```

if is unknown authority issue, make sure the CA is updated and restart vault and then unseal again)
```
$ sudo systemctl restart vault
```
Then, we'll create the 'default' role for OIDC:
```
$ export VAULT_UI=https://{your public ip address}:8200
$ export VAULT_CLI=https://127.0.0.1:8250
$ vault write auth/oidc/role/default \
allowed_redirect_uris="${VAULT_UI}/ui/vault/auth/oidc/oidc/callback" \
allowed_redirect_uris="${VAULT_CLI}/oidc/callback" \
user_claim="email" \
policies="admin"
```
(*There might be redirect URI differences in the vault.json when import client, so make sure the Valid redirect URIs in Keycloak Vault client exactly match the URIs input above!)
![image](https://github.com/chuyanc/WRDS-Documentations/assets/103159777/0f8cbed9-44dd-4d7b-9417-41e17b1c7c03)



#### 1.8 Use OIDC to log in Vault UI
Go to https://{your IP Address}:8200/ and select OIDC as the log in method, follow the last UI operation step in Reference 1 to log in.

(*Remember to log out the admin account via Keycloak first, otherwise the email and password input box will be skipped and directly let you log in Vault as Keycloak admin. And if you didn't add an email for Keycloak admin, it will occur error: claim "email" not found in token)



