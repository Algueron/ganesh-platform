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
