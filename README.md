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

### Create Realm

- On the Keycloak server, authenticate as admin
````bash
/opt/keycloak/bin/kcadm.sh config credentials --server https://keycloak.algueron.io --realm master --user $KEYCLOAK_ADMIN --password $KEYCLOAK_ADMIN_PASSWORD
````

- Create the Ganesh realm
````bash
/opt/keycloak/bin/kcadm.sh create realms -s realm=ganesh -s enabled=true
````
