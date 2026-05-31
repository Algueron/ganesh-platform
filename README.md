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
cat <<EOF  | sudo tee -a /etc/postgresql/18/main/postgresql.conf
listen_addresses = '*'
EOF
````

- Edit PostgreSQL authentication file (temporarily allow all)
````bash
cat <<EOF  | sudo tee -a /etc/postgresql/18/main/pg_hba.conf
# Allow Kerberos authentication
host    all     all     0.0.0.0/0   md5
EOF
````

- Restart PostgreSQL to apply changes
````bash
sudo systemctl restart postgresql
````

### Setup Keycloak

Keycloak will be used as the Identity Provider. It should be installed on a dedicated server.

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
sudo wget -O /opt/keycloak.zip https://github.com/keycloak/keycloak/releases/download/26.2.2/keycloak-26.2.2.zip
````

- Extract Keycloak archive
````bash
sudo unzip /opt/keycloak.zip -d /opt/
````

- Rename directory
````bash
sudo mv /opt/keycloak-26.2.2 /opt/keycloak
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
/opt/keycloak/bin/kcadm.sh create users -r ganesh -s username=ganesh -s enabled=true
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

- Create the Contour configuration
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/contour/contour-deployment-config.yaml
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

- Add the Rook Helm chart repository
````bash
helm repo add rook-release https://charts.rook.io/release
````

- Install the Rook Operator
````bash
helm install --create-namespace --namespace rook-ceph rook-ceph rook-release/rook-ceph -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/rook/rook-ceph-operator-values.yaml
````

- Install the Rook cluster
````bash
helm install --create-namespace --namespace rook-ceph rook-ceph-cluster --set operatorNamespace=rook-ceph rook-release/rook-ceph-cluster -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/rook/rook-ceph-cluster-values.yaml
````

- Create a HTTPRoute for Ceph Dashboard
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/rook/rook-dashboard-http-route.yaml
````

- Retrieve the admin password
````bash
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo
````

- Set Rook Block storage as the default StorageClass
````bash
kubectl patch storageclass rook-ceph-block -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
````

### Setup MinIO

- Add the MinIO Operator Helm chart repository
````bash
helm repo add minio-operator https://operator.min.io
````

- Install the MinIO Operator
````bash
helm install --namespace minio-operator --create-namespace minio-operator minio-operator/operator
````

- Create the Tenant namespace
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/minio/minio-tenant-namespace.yaml
````

- Download the Secret file manifest
````bash
wget https://raw.githubusercontent.com/Algueron/ganesh-platform/main/minio/minio-tenant-storage-configuration.yaml
````

- Generate Root access and secret keys
````bash
export ROOT_ACCESS_KEY=$(openssl rand -base64 -hex 18)
export ROOT_SECRET_KEY=$(openssl rand -base64 -hex 24)
````

- Fill the Tenant storage configuration manifest
````bash
sed -i -e "s/ROOT_ACCESS_KEY/$ROOT_ACCESS_KEY/g" minio-tenant-storage-configuration.yaml
sed -i -e "s/ROOT_SECRET_KEY/$ROOT_SECRET_KEY/g" minio-tenant-storage-configuration.yaml
````

- Create the Tenant storage configuration
````bash
kubectl apply -f minio-tenant-storage-configuration.yaml
````

- Download the MinIO admin user manifest
````bash
wget https://raw.githubusercontent.com/Algueron/ganesh-platform/main/minio/minio-tenant-storage-user.yaml
````

- Generate admin access and secret keys
````bash
export MINIO_ADMIN_LOGIN=admin
export MINIO_ADMIN_PASSWORD=$(openssl rand -base64 -hex 24)
````

- Fill the MinIO admin configuration manifest
````bash
sed -i -e "s/MINIO_ADMIN_LOGIN/$(echo -n $MINIO_ADMIN_LOGIN | base64 | tr --delete '\n')/g" minio-tenant-storage-user.yaml
sed -i -e "s/MINIO_ADMIN_PASSWORD/$(echo -n $MINIO_ADMIN_PASSWORD | base64 | tr --delete '\n')/g" minio-tenant-storage-user.yaml
````

- Create the Tenant storage user
````bash
kubectl apply -f minio-tenant-storage-user.yaml
````

- Create the tenant
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/minio/minio-tenant.yaml
````

- Create a HTTPRoute for MinIO console
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/minio/minio-tenant-console-http-route.yaml
````

- Create a HTTPRoute for MinIO S3 endpoint
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/minio/minio-tenant-s3-http-route.yaml
````

### MinIO Health check

- Download MinIO client
````bash
wget https://dl.min.io/client/mc/release/windows-amd64/mc.exe
````

- Create an alias for the S3 service
````bash
mc alias set ganesh https://s3.ganesh.algueron.io $MINIO_ADMIN_LOGIN $MINIO_ADMIN_PASSWORD
````

- Test the connection to the S3 service
````bash
mc admin info ganesh
````

## Ganesh services setup

### Setup Lakekeeper

- Create Lakekeeper namespace
````bash
kubectl create namespace lakekeeper
````

- Add the Lakekeeper Helm chart repository to Helm
````bash
helm repo add lakekeeper https://lakekeeper.github.io/lakekeeper-charts/
````

- Generate a password for the database
````bash
export LAKEKEEPER_DB_PASSWORD=$(openssl rand -base64 -hex 24)
````

- Download Lakekeeper database creation script
````bash
sudo wget https://raw.githubusercontent.com/Algueron/ganesh-platform/main/lakekeeper/lakekeeper-db-creation.sql
````

- Generate a password for the database and fill the configuration files
````bash
sed -i -e "s/LAKEKEEPER_DB_PASSWORD/$LAKEKEEPER_DB_PASSWORD/g" lakekeeper-db-creation.sql
````

- Create the Lakekeeper database and user
````bash
sudo su -c "psql -f lakekeeper-db-creation.sql" postgres
````

- Download Lakekeeper secret manifest
````bash
sudo wget https://raw.githubusercontent.com/Algueron/ganesh-platform/main/lakekeeper/lakekeeper-postgresql-secret.yaml
````

- Fill the credentials for PostgreSQL
````bash
export LAKEKEEPER_DB_USER=lakekeeper
sed -i -e "s/LAKEKEEPER_DB_USER/$(echo -n $LAKEKEEPER_DB_USER | base64 | tr --delete '\n')/g" lakekeeper-postgresql-secret.yaml
sed -i -e "s/LAKEKEEPER_DB_PASSWORD/$(echo -n $LAKEKEEPER_DB_PASSWORD | base64 | tr --delete '\n')/g" lakekeeper-postgresql-secret.yaml
````

- Create the secret for PostgreSQL credentials
````bash
kubectl apply -f lakekeeper-postgresql-secret.yaml
````

- Download Lakekeeper Helm values file
````bash
sudo wget https://raw.githubusercontent.com/Algueron/ganesh-platform/main/lakekeeper/lakekeeper-helm-values.yaml
````

- Fill the PostgreSQL host in the Helm values file
````bash
export LAKEKEEPER_DB_HOST=xxx.xxx.xxx.xxx  # Replace with your IP
sed -i -e "s/LAKEKEEPER_DB_HOST/$LAKEKEEPER_DB_HOST/g" lakekeeper-helm-values.yaml
````

- Deploy Lakekeeper
````bash
helm install -f lakekeeper-helm-values.yaml --namespace lakekeeper ganesh-lakekeeper lakekeeper/lakekeeper
````

- Create a HTTPRoute for Lakekeeper endpoint
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/lakekeeper/lakekeeper-http-route.yaml
````

- Bootstrap the server
````bash
curl --location 'https://lakekeeper.ganesh.algueron.io/management/v1/bootstrap' \
--header 'Content-Type: application/json' \
--data '{
    "accept-terms-of-use": true
}'
````

### Setup StarRocks

- Deploy StarRocks Custom Resource Definitions
````bash
kubectl apply -f https://raw.githubusercontent.com/StarRocks/starrocks-kubernetes-operator/main/deploy/starrocks.com_starrocksclusters.yaml
````

- Create StarRocks namespace
````bash
kubectl create namespace starrocks
````

- Deploy StarRocks Operator
````bash
kubectl apply -f https://raw.githubusercontent.com/StarRocks/starrocks-kubernetes-operator/main/deploy/operator.yaml
````

- Deploy StarRocks cluster
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/starrocks/starrocks-cluster.yaml
````

- Create a HTTPRoute for StarRocks
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/starrocks/starrocks-http-route.yaml
````

- Expose the MySQL port through a NodePort service
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/starrocks/starrocks-mysql-service.yaml
````

- Connect using MySQL client and update root password
````sql
SET PASSWORD = PASSWORD('<password>')
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

- Download Airflow Helm values file
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

- Create a Persistent Volume Claim for storing DAGs
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/airflow/airflow-dag-volume-claim.yaml
````

- Create a Persistent Volume Claim for storing logs
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/airflow/airflow-log-volume-claim.yaml
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

### Setup Windmill

- Add the Airflow Helm chart repository
````bash
helm repo add windmill https://windmill-labs.github.io/windmill-helm-charts/
````

- Deploy airflow
````bash
helm install windmill windmill/windmill --namespace=windmill --create-namespace
````

- Create a HTTPRoute for Windmill App Server
````bash
kubectl apply -f https://raw.githubusercontent.com/Algueron/ganesh-platform/main/windmill/windmill-http-route.yaml
````
