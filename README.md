# ganesh-platform
An OSS Data Platform based on Kubernetes

## Prerequisites

### Kubernetes

You should have a Kubernetes cluster (tested with 1.29).

### Helm

You should have Helm > 3.14.x

### S3 Client

You should use s5cmd, available at https://github.com/peak/s5cmd


## Infrastructure setup

### Setup PostgreSQL

PostgreSQL should be installed on a dedicated server.

- Install system utilities
````bash
sudo apt install -y curl ca-certificates
````

- Create a directory to store PostgreSQL repository certificates
````bash
sudo install -d /usr/share/postgresql-common/pgdg
````

- Download PostgreSQL repository certificates
````bash
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
````

- Add PostgreSQL repository
````bash
sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
````

- Update repositories
````bash
sudo apt update
````

- Install PostgreSQL
````bash
sudo apt install -y postgresql
````

- Edit PostgreSQL configuration (set listen_addresses = '*')
````bash
cat <<EOF  | sudo tee -a /etc/postgresql/16/main/postgresql.conf
listen_addresses = '*'
EOF
````

- Edit PostgreSQL authentication file (temporarily allow all)
````bash
cat <<EOF  | sudo tee -a /etc/postgresql/16/main/pg_hba.conf
# Allow Kerberos authentication
host    all     all     0.0.0.0/0   md5
EOF
````

- Restart PostgreSQL to apply changes
````bash
sudo systemctl restart postgresql
````

### Setup Keycloak

Keycloak will be used as the Identity Provider. It is currently installed on the same server than PostgreSQL.

- Install system utilities
````bash
sudo apt install -y unzip
````

- Install OpenJDK 21
````bash
sudo apt install -y openjdk-21-jdk
````

- Create service account for Keycloak
````bash
sudo adduser keycloak
````

- Download Keycloak
````bash
sudo wget -O /opt/keycloak.zip https://github.com/keycloak/keycloak/releases/download/25.0.0/keycloak-25.0.0.zip
````

- Extract Keycloak archive
````bash
sudo unzip /opt/keycloak.zip -d /opt/
````

- Rename directory
````bash
sudo mv /opt/keycloak-25.0.0 /opt/keycloak
````

- Set ownership to Keycloak service account
````bash
sudo chown keycloak:keycloak -R /opt/keycloak
````

- Create a directory for configuration
````bash
sudo mkdir -p /etc/keycloak
````

- Download Keycloak configuration file
````bash
sudo wget -O /etc/keycloak/keycloak.conf https://raw.githubusercontent.com/Algueron/ganesh-platform/main/keycloak/keycloak.conf
````

- Download Keycloak database creation script
````bash
sudo wget -O /etc/keycloak/keycloak-db-creation.sql https://raw.githubusercontent.com/Algueron/ganesh-platform/main/keycloak/keycloak-db-creation.sql
````

- Set ownership to Keycloak service account
````bash
sudo chown keycloak:keycloak -R /etc/keycloak
````

- Generate a password for the database and fill the configuration files
````bash
export KEYCLOAK_DB_PASSWORD=$(openssl rand -base64 18)
sudo su -c "sed -i -e \"s/KEYCLOAK_PASSWORD/$KEYCLOAK_DB_PASSWORD/g\" /etc/keycloak/keycloak-db-creation.sql" keycloak
sudo su -c "sed -i -e \"s/KEYCLOAK_PASSWORD/$KEYCLOAK_DB_PASSWORD/g\" /etc/keycloak/keycloak.conf" keycloak
````

- Get the Host IP
````bash
export POSTGRESQL_IP=$(hostname --ip-address | cut -d ' ' -f1)
sudo su -c "sed -i -e \"s/POSTGRESQL_IP/$POSTGRESQL_IP/g\" /etc/keycloak/keycloak.conf" keycloak
````

- Set the FQDN for the keycloak instance
````bash
export KEYCLOAK_HOSTNAME=keycloak.algueron.io
sudo su -c "sed -i -e \"s/KEYCLOAK_HOSTNAME/$KEYCLOAK_HOSTNAME/g\" /etc/keycloak/keycloak.conf" keycloak
````

- Download the certificate and key files and place them into /etc/keycloak/, then change ownership
````bash
sudo chown keycloak:keycloak -R /etc/keycloak/*.pem
````

- Configure a Revert proxy to point to your Keycloak instance

- Create the Keycloak database and user
````bash
sudo su -c "psql -f /etc/keycloak/keycloak-db-creation.sql" postgres
````

- Generate credentails for Keycloak admin user, e.g.
````bash
export KEYCLOAK_ADMIN=algueron
export KEYCLOAK_ADMIN_PASSWORD=$(openssl rand -base64 18)
````

- Start a first time Keycloak to generate the admin user, and kill it once it runs.
````bash
sudo sh -c "export KEYCLOAK_ADMIN=$KEYCLOAK_ADMIN && export KEYCLOAK_ADMIN_PASSWORD=$KEYCLOAK_ADMIN_PASSWORD && /opt/keycloak/bin/kc.sh --config-file=/etc/keycloak/keycloak.conf start"
````

- Download the systemd unit
````bash
sudo wget -O /lib/systemd/system/keycloak.service https://raw.githubusercontent.com/Algueron/ganesh-platform/main/keycloak/keycloak.service
````

- Reload systemd definitions
````bash
sudo systemctl daemon-reload
````

- Start Keycloak service
````bash
sudo systemctl enable --now keycloak
````

### Create Realm and Admin user

- On the Keycloak server, authenticate as admin
````bash
/opt/keycloak/bin/kcadm.sh config credentials --server https://keycloak.algueron.io --realm master --user $KEYCLOAK_ADMIN --password $KEYCLOAK_ADMIN_PASSWORD
````

- Create the Ganesh realm
````bash
/opt/keycloak/bin/kcadm.sh create realms -s realm=ganesh -s enabled=true
````

- Create your admin user
````bash
/opt/keycloak/bin/kcadm.sh create users -r ganesh -s username=algueron -s enabled=true
````

- Set your password
````bash
/opt/keycloak/bin/kcadm.sh set-password -r ganesh --username ganesh --new-password NEWPASSWORD
````

## Kubernetes services setup

### Setup Project Contour

Project Contour will be used as the Gateway API Provisioner.

- Create the Gateway Provisioner
````bash
kubectl apply -f https://projectcontour.io/quickstart/contour-gateway-provisioner.yaml
````

- Create the GatewayClass
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/contour/contour-gateway-class.yaml
````

- Download certificate and key for the wildcard sub-domain

- Download the Secret file manifest
````bash
wget https://raw.githubusercontent.com/Algueron/ganesh-platform/main/contour/contour-secret.yaml
````

- Fill the certificate and key data in the secret manifest
````bash
sed -i -e "s/GANESH_ENCODED_CERT/$(cat certificate.pem | base64 | tr --delete '\n')/g" contour-secret.yaml
sed -i -e "s/GANESH_ENCODED_KEY/$(cat key.pem | base64 | tr --delete '\n')/g" contour-secret.yaml
````

- Create the TLS Secret
````bash
kubectl apply -f contour-secret.yaml
````

- Create the Gateway
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/contour/contour-gateway.yaml
````

### Setup ArgoCD

- Create ArgoCD namespace
````bash
kubectl create namespace argocd
````

- Install ArgoCD
````bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
````

- Create a HTTPRoute for ArgoCD
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/argocd/argocd-http-route.yaml
````

- Switch ArgoCD to insecure mode, as TLS termination is handled by the reverse proxy
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/argocd/argocd-patch.yaml
````

- Restart ArgoCD server for changes to be applied
````bash
kubectl delete pod -n argocd --selector=app.kubernetes.io/name=argocd-server
````

### Configure ArgoCD authentication

- On the Keycloak server, authenticate as admin
````bash
/opt/keycloak/bin/kcadm.sh config credentials --server https://keycloak.algueron.io --realm master --user $KEYCLOAK_ADMIN --password $KEYCLOAK_ADMIN_PASSWORD
````

- Create the ArgoCD client
````bash
/opt/keycloak/bin/kcadm.sh create clients \
    -r ganesh \
    -s clientId=argocd \
    -s name="ArgoCD Client" \
    -s fullScopeAllowed=true \
    -s standardFlowEnabled=true \
    -s directAccessGrantsEnabled=true \
    -s rootUrl="https://argocd.ganesh.algueron.io" \
    -s baseUrl="/applications" \
    -s 'redirectUris=["https://argocd.ganesh.algueron.io/auth/callback"]' \
    -s 'webOrigins=["https://argocd.ganesh.algueron.io"]' \
    -s adminUrl="https://argocd.ganesh.algueron.io" \
    -s 'attributes={"post.logout.redirect.uris":"+"}'
````

- Retrieve the client secret generated
````bash
export KEYCLOAK_SECRET=$(/opt/keycloak/bin/kcadm.sh get clients -r ganesh --query clientId=argocd --fields secret --format csv --noquotes)
````

- Create a client scope using Keycloak web UI to allow ArgoCD to get group membership, name it "groups" and enable "Include in token scope"

- In the "Mappers" tab, click on "Configure a new mapper" and choose Group Membership.

- Make sure to set the Name as well as the Token Claim Name to "groups". Also disable the "Full group path".

- Go back to the client we've created earlier and go to the Tab "Client Scopes". Click on "Add client scope", choose the groups scope and add it either to the Default Client Scope.

- Create the ArgoCD Admin Group
````bash
/opt/keycloak/bin/kcadm.sh create groups -r ganesh -s name="ArgoCDAdmins"
````

- Add your admin user to the group

- Edit the ArgoCD secret to include your keycloak secret
````bash
kubectl edit secret argocd-secret -n argocd
````

- Add the line in data with a key "oidc.keycloak.clientSecret" and your secret encoded in bae64

- Edit the ArgoCD configuration to configure the keycloak instance
````bash
kubectl edit configmaps -n argocd argocd-cm
````

- Add the following lines in the file
````yaml
data:
  url: https://argocd.ganesh.algueron.io
  oidc.config: |
    name: Keycloak
    issuer: https://keycloak.algueron.io/realms/ganesh
    clientID: argocd
    clientSecret: $oidc.keycloak.clientSecret
    requestedScopes: ["openid", "profile", "email", "groups"]
````

- Map the group ArgoCDAdmins created earlier to the role admin by editing the RBAC config map
````bash
kubectl edit configmaps -n argocd argocd-rbac-cm
````

- Add the following lines in the file
````yaml
data:
  policy.csv: |
    g, ArgoCDAdmins, role:admin
````
