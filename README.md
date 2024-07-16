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

- Download ArgoCD client and move it in your PATH as argocd.exe
````bash
wget https://github.com/argoproj/argo-cd/releases/download/v2.7.2/argocd-windows-amd64.exe
````

- Retrieve the admin password
````bash
argocd admin initial-password -n argocd
````

### Setup Rook

- Install the Rook Operator
````bash
export ROOK_VERSION=1.14
kubectl apply -f https://raw.githubusercontent.com/rook/rook/release-$ROOK_VERSION/deploy/examples/crds.yaml
kubectl apply -f https://raw.githubusercontent.com/rook/rook/release-$ROOK_VERSION/deploy/examples/common.yaml
kubectl apply -f https://raw.githubusercontent.com/rook/rook/release-$ROOK_VERSION/deploy/examples/operator.yaml
````

- Create the Rook cluster
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/rook/rook-cluster.yaml
````

- Create the Block Storage
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/rook/rook-block-replicapool.yaml
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/rook/rook-block-storage-class.yaml
````

- Create the CephFS Storage
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/rook/rook-cephfs-filesystem.yaml
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/rook/rook-cephfs-storage-class.yaml
````

- Create the Object Store
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/rook/rook-ceph-object-store.yaml
````

- Create a HTTPRoute for Ceph Dashboard
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/rook/rook-dashboard-http-route.yaml
````

- Create a HTTPRoute for Ceph Object Store
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/rook/rook-object-store-http-route.yaml
````

- Retrieve the admin password
````bash
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo
````

- Set Rook Block storage as the default StorageClass
````bash
kubectl patch storageclass rook-ceph-block -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
````

## Ganesh services setup

### Setup Airbyte

- Create Airbyte namespace
````bash
kubectl create namespace airbyte
````

- Add the Airbyte Helm chart repository to Helm
````bash
helm repo add airbyte https://airbytehq.github.io/helm-charts
````

- Deploy Airbyte
````bash
helm install --namespace airbyte airbyte airbyte/airbyte
````

### Configure Airbyte reverse proxy

- Create a HTTPRoute for Airbyte WebApp
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/airbyte/airbyte-http-route.yaml
````

### Setup Airflow

- Add the Airflow Helm chart repository
````bash
helm repo add airflow https://airflow.apache.org
````

- Create Airflow namespace
````bash
kubectl create namespace airflow
````

- Download Airflow database creation script
````bash
sudo wget https://raw.githubusercontent.com/Algueron/ganesh-platform/main/airflow/airflow-db-creation.sql
````

- Download Keycloak configuration file
````bash
sudo wget https://raw.githubusercontent.com/Algueron/ganesh-platform/main/airflow/airflow-helm-values.yaml
````

- Generate a password for the database and fill the configuration files
````bash
export AIRFLOW_DB_PASSWORD=$(openssl rand -base64 18)
sed -i -e "s/AIRFLOW_DB_PASSWORD/$AIRFLOW_DB_PASSWORD/g" airflow-db-creation.sql
sed -i -e "s/AIRFLOW_DB_PASSWORD/$AIRFLOW_DB_PASSWORD/g" airflow-helm-values.yaml
````

- Create the Airflow database and user
````bash
sudo su -c "psql -f airflow-db-creation.sql" postgres
````

- Fill the PostgreSQL host in the Helm values file
````bash
export AIRFLOW_DB_HOST=xxx.xxx.xxx.xxx  # Replace with your IP
sed -i -e "s/AIRFLOW_DB_HOST/$AIRFLOW_DB_HOST/g" airflow-helm-values.yaml
````

- Deploy airflow
````bash
helm install airflow airflow/airflow --namespace airflow --values airflow-helm-values.yaml
````

- Create a HTTPRoute for Airflow Web Server
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/airflow/airflow-http-route.yaml
````

- Connect and change the admin password to something secure

### Setup DataHub

- Create DataHub namespace
````bash
kubectl create namespace datahub
````

- Add the DataHub Helm chart repository
````bash
helm repo add datahub https://helm.datahubproject.io/
````

- Install DataHub Prerequisites (Kafka and Elasticsearch)
````bash
helm install prerequisites datahub/datahub-prerequisites --namespace datahub --values https://raw.githubusercontent.com/Algueron/ganesh-platform/main/datahub/datahub-prerequisites-helm-values.yaml
````

- Download DataHub database creation script
````bash
sudo wget https://raw.githubusercontent.com/Algueron/ganesh-platform/main/datahub/datahub-db-creation.sql
````

- Download DataHub Helm values file
````bash
sudo wget https://raw.githubusercontent.com/Algueron/ganesh-platform/main/datahub/datahub-helm-values.yaml
````

- Generate a password for the database and fill the configuration files
````bash
export DATAHUB_DB_PASSWORD=$(openssl rand -base64 18)
sed -i -e "s/DATAHUB_DB_PASSWORD/$DATAHUB_DB_PASSWORD/g" datahub-db-creation.sql
sed -i -e "s/DATAHUB_DB_PASSWORD/$DATAHUB_DB_PASSWORD/g" datahub-helm-values.yaml
````

- Create the DataHub database and user
````bash
sudo su -c "psql -f datahub-db-creation.sql" postgres
````

- Fill the PostgreSQL host in the Helm values file
````bash
export DATAHUB_DB_HOST=xxx.xxx.xxx.xxx  # Replace with your IP
sed -i -e "s/DATAHUB_DB_HOST/$DATAHUB_DB_HOST/g" datahub-helm-values.yaml
````

- Download the users file
````bash
sudo wget https://raw.githubusercontent.com/Algueron/ganesh-platform/main/datahub/user.props
````

- Generate an admin password
````bash
export DATAHUB_ADMIN_PASSWORD=$(openssl rand -base64 18)
sed -i -e "s/DATAHUB_ADMIN_PASSWORD/$DATAHUB_ADMIN_PASSWORD/g" user.props
````

- Create a secret to store the users file
````bash
kubectl create secret generic datahub-users-secret -n datahub --from-file=user.props=./user.props
````

- Deploy DataHub
````bash
helm install datahub datahub/datahub --namespace datahub --values datahub-helm-values.yaml --timeout 20m
````

- Create a HTTPRoute for DataHub frontend
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/datahub/datahub-http-route.yaml
````
