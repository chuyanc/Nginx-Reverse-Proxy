This is the documentation of how to set up Keycloak and Vault for authentication and authorization of a virtual machine cluster for a group.

## Task Description
### Goal
We aim to create a Group VM Single Sign-On (SSO) system for the team, ensuring basic authentication and authorization. This solution should be flexible for user authentication, easy for administrators, and maintain a balance between simplicity and security. The goal is to swiftly integrate the SSO system within the team, unblocking projects that need VM access and fostering a secure, user-friendly VM environment.

### Reference
[1. Vault + KeyCloak OIDC](https://drpdishant.medium.com/identity-based-ssh-with-vault-and-keycloak-part-1-3-47ab2181ceae)

[2. Vault SSH Certificate](https://drpdishant.medium.com/identity-based-ssh-with-vault-and-keycloak-part-2-3-signed-ssh-certificate-c9fb2c4dde64)

### Environment Setting Up
1. We are using **Ubuntu 22.04** instead of the Virtual Box + Vagrant in reference 1, so some steps might be different. But I'll refer some of the gif in the reference for the operations on user interface.

2. Please ensure **at least 4GiB of RAM** for your Keycloak + Vault VM instance (like t2.medium or t2.large for Amazon EC2)

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
Access Vault UI via: https://{vault-server-ip}:8200/. Follow the UI operation steps of Initialize, Unseal and Login in Reference 1.

#### 1.3 Initialize and Run Keycloak
##### 1) Self-sign certificate
```
$ openssl req -x509 -nodes -newkey rsa:2048 -keyout myserver.key -out myserver.crt -days 365 -subj "/CN=myserver" -addext "subjectAltName = IP: {vault-server-ip}"
$ chmod 755 myserver.key
```
##### 2) Update CA
```
$ sudo cp myserver.crt /usr/local/share/ca-certificates/
$ sudo update-ca-certificates
```
##### 3) Install Docker
```
$ curl -sSL https://get.docker.com/ | sudo bash
```
##### 4) Run Keycloak
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

#### 1.4 Set up Keycloak through admin dashboard UI
##### 1) Set vault client and create user
Access Keycloak UI via: https://{vault-server-ip}:8443/, log in as admin using the credentials we set above, and follow the UI operation steps in Reference 1 for set vault client on keycloak and create user. 

Remember to modify the redirect URIs ip address to the real ip address of your vault server:![image](https://github.com/chuyanc/WRDS-Documentations/assets/103159777/d7c1615b-bf6d-4f97-8e38-cae232838ff3)

##### 2) Add mapper for group claim
In the admin dashboard, go to tab: Clients > [Client] > Client Scopes > [vault-dedicated (the scope with description = "Dedicated scope and mappers for this client")]

And then add a mapper by configuration: "Group Membership", then set the other fields according to this picture, click "save": <img width="1068" alt="image" src="https://github.com/chuyanc/WRDS-Documentations/assets/103159777/754c71fc-706f-452a-bab4-922c1f474e65">

##### 3) Create groups and assign users to groups
In the Groups tab on Keycloak admin dashboard, click "Create group" to add groups.

In User-Groups tab, we can also click "Join Group" to assign some groups to the user. <img width="1908" alt="image" src="https://github.com/chuyanc/WRDS-Documentations/assets/103159777/ca4ae7bc-995e-4903-9d60-75a6548c1f39">



#### 1.5 Access Vault CLI
```
$ export VAULT_SKIP_VERIFY=true
$ echo 'export VAULT_SKIP_VERIFY=true' >> ~/.bashrc
$ source ~/.bashrc
$ export VAULT_TOKEN={your token}
$ export VAULT_ADDR='https://127.0.0.1:8200'
$ vault login $VAULT_TOKEN
```

#### 1.6 Create Policies
Open a new file {policy-name}.hcl, and write the policy to indicate which capability of which path you'd like to give access to the users with this policy.

Generally speaking, we can enable the access of certain ssh signer role they have access to (created in section 2.1) and a path to see all the ssh signer roles available in vault:
```
path "ssh-client-signer/sign/{signer-role-name}" {
    capabilities = ["create", "update"]
}

path "ssh-client-signer/roles" {
    capabilities = ["read", "list"]
}
```
then assign the policy to certain policy name
```
$ vault policy write {policy-name} {policy-name}.hcl
```

#### 1.7 Configure OIDC Auth
```
$ vault auth enable oidc

$ export KC_DOMAIN="https://{vault-server-ip}:8443/realms/master"
$ export KC_CLIENT_ID=vault
$ export KC_CLIENT_SECRET={your-client-secret}
$ vault write auth/oidc/config \
        oidc_discovery_url="$KC_DOMAIN" \
        oidc_client_id="$KC_CLIENT_ID" \
        oidc_client_secret="$KC_CLIENT_SECRET" \
        default_role="{role-name}"
```
(*if doesn’t work, use the following command to check the vault logs:
```
$ sudo journalctl -u vault.service -n 50
```

if is unknown authority issue, make sure the CA is updated and restart vault and then unseal again)
```
$ sudo systemctl restart vault
```
Then, we'll create a role for OIDC:
```
vault write auth/oidc/role/{role-name} -<<EOF
{
  "allowed_redirect_uris":["https://3.209.163.26:8200/ui/vault/auth/oidc/oidc/callback", "https://127.0.0.1:8250/oidc/callback"],
  "user_claim": "email",
  "policies": ["{policy-name}"],
  "bound_claims": {"groups":["{group-name}"]}
}
EOF
```
({group-name} is the name of groups you created using Keycloak admin dashboard UI in section 1.4)

(*There might be redirect URI differences in the vault.json when import client, make sure the Valid redirect URIs set in Keycloak Vault client in section 1.4 exactly match the URIs input above!)


*You can repeat 1.6 and 1.7 to create more different roles with different policies and groups bound claims, and use ```vault list auth/oidc/role``` command to see what roles were created.



#### 1.8 Use OIDC to log in Vault UI
This is part of the user flow. Go to https://{vault-server-ip}:8200/ and select OIDC as the log in method, follow the last UI operation step in Reference 1 to log in. 

* Users also have to indicate a role they want to log in as, if the user is not in the groups that the role is bounded with, "error validating claims: claim "groups" does not match any associated bound claim values" will occur.

(*If you are logged in with admin account of Keycloak, remember to log out first, otherwise the email and password input box will be skipped and directly let you log in Vault as Keycloak admin. And if you didn't add an email for Keycloak admin in this time, it will occur error: claim "email" not found in token)

### 2. Configuring Vault SSH Secrets engine for Signed SSH Certificates
#### 2.1 Enabling and Configuring SSH Engine
##### 1) Enable Secrets Engine
```
$ vault secrets enable -path=ssh-client-signer ssh
```
##### 2) Configure CA for signing keys
```
$ vault write ssh-client-signer/config/ca generate_signing_key=true
```
##### 3) Configure Signing Role

Assume we want to have different usernames to log into different VMs, let's say we created 3 different roles following section 1.6 and 1.7: role1, role2, role3, and they can access VM1, VM2, VM3 respectively. We can add VM users: vm-user1, vm-user2, vm-user3 on VM1, VM2, VM3 respectively following section 2.2, and create 3 ssh signer roles like:
```
$ vault write ssh-client-signer/roles/{signer-role-name} -<<"EOH"
{
 "algorithm_signer": "rsa-sha2-512",
 "allow_user_certificates": true,
 "allowed_users": "{vm-username}",
 "allowed_extensions": "permit-pty,permit-port-forwarding",
 "default_extensions": [
   {
     "permit-pty": ""
   }
 ],
 "key_type": "ca",
 "default_user": "{vm-username}",
 "ttl": "30m0s"
}
EOH
```
then add the capabilities of "create" and "update" for path "ssh-client-signer/roles/k8s-admin" as said in section 1.6.

#### 2.2 Configure SSH Host
*The SSH Host is the VM group members wants to access, so these configurations happen on the Group VM, instead of the Vault server.
##### 1) Enter SSH Host VM and download public key
```
$ sudo curl -k https://{vault-server-ip}:8200/v1/ssh-client-signer/public_key -o /etc/ssh/trusted-user-ca-keys.pem
```
##### 2) Configure CA public key as trusted
```
$ sudo nano /etc/ssh/sshd_config
```
then add the following line: 
```
TrustedUserCAKeys /etc/ssh/trusted-user-ca-keys.pem
```
Restart the ssh daemon:
```
$ sudo systemctl restart sshd
```
##### 3) Add a new user to VM

As said in section 2.1 step3, we have to create a new user on VMs which correspond to the "allowed_users" we set in signing role configuation. If you'd like to grant the user with ability to execute commands with superuser privileges, you'll need to add them to the sudo group:

```
$ sudo adduser {vm_username}
$ sudo usermod -aG sudo {vm_username}
```

#### 2.3 Client Authentication
This step is part of the user flow, happens in the user's local machine (make sure to have Vault installed).

##### 1）Generate SSH Keypair
```
$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/vagrant/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/vagrant/.ssh/id_rsa
Your public key has been saved in /home/vagrant/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:/rZ/5imrr/vW8Ex10sibkgGiJpSzUfgLCdrXFbLlvIo vagrant@ssh-client
The key's randomart image is:
+---[RSA 3072]----+
|     +o o.       |
|  . *  *o .      |
| o o B.oo. . . o |
|. . * =  .  . + +|
|   . + .S    o =.|
|     ..o    + +  |
|    E . .    B   |
|         .. o *. |
|         .*X=*o  |
+----[SHA256]-----+
```

##### 2) Configure Vault CLI
```
$ export VAULT_ADDR=https://{vault-server-ip}:8200
$ export VAULT_SKIP_VERIFY=true 
## Use Token Copied from UI (see the gif in reference 2)
$ export VAULT_TOKEN={copied token}
$ echo $VAULT_TOKEN | vault login -
```

##### 3) Signing SSH Public Key

First, use command ```vault list ssh-client-signer/roles``` to see what are the signer roles available

```
vault write -field=signed_key ssh-client-signer/sign/{signer-role-name} \
    public_key=@$HOME/.ssh/id_rsa.pub > $HOME/.ssh/id_rsa-cert.pub
```
This will authorize the user with vault, and request for the public key at ‘~/.ssh/id_rsa.pub’ to be signed by the CA, then generate the SSH certificate.

*If the user is requesting a signer role which is beyond the access of the current logged in user role, it will show 403 permission denied error.

To check the details of the signed certificate, run:
```
$ ssh-keygen -Lf ~/.ssh/id_rsa-cert.pub
```

It will show similar results as below:
```
/home/vagrant/.ssh/id_rsa-cert.pub:
        Type: ssh-rsa-cert-v01@openssh.com user certificate
        Public key: RSA-CERT SHA256:/rZ/5imrr/vW8Ex10sibkgGiJpSzUfgLCdrXFbLlvIo
        Signing CA: RSA SHA256:uKzJOaH/cPaxmDDgrwAEoo0eY9YUKYh2dqvrno1UYEc (using ssh-rsa)
        Key ID: "vault-oidc-testing@example.com-feb67fe629abaffbd6f04c75d2c89b9201a22694b351f80b09dad715b2e5bc8a"
        Serial: 9812201999919852695
        Valid: from 2020-12-18T01:17:26 to 2020-12-18T01:27:56
        Principals: 
                ubuntu
        Critical Options: (none)
        Extensions: 
                permit-pty
```

**After doing this, the user can directly access the group VMs they have access to by running:**
```
$ ssh {vm-username}@{vm-public-ip}
```

*If user of role1 are trying to use the key to access VM2, it will show error "Permission denied (publickey)"





