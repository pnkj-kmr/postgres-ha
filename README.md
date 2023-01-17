# PostgreSQL with repmgr - High Availability Setup

_postgres [14.6] and repmgr [5.3.3] with 3 linux machine | choose higher version if you like_

## postgres on HA - these setups help

1. Postgres Installation [skip if already installed]
2. repmgr Installation [it helps to data syncing]
3. Hostname Resolution [or we can skip this by IP address]
4. SSH authentication with ssh-keygen
5. Postgres configuration
6. repmgr configuration
7. Always connect to primary [after failover or sync]

![PostgreSQL](./helpers/postgres.png)

### 1. Postgres Installation

---

_Here we are refering the linux machine to install postgres. To install [refer here](https://www.postgresql.org/download/) or below command on terminal to install_

```
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
yum -qy module disable postgresql
yum install -y postgresql14-server
```

_if linux machine has firewall service running, we need to enable port by running below command_

```
firewall-cmd --permanent --add-port=5432/tcp
firewall-cmd --reload
```

_If you need [timescaledb](https://docs.timescale.com/install/latest/self-hosted/installation-redhat/) as well, install by the following below commands_

```
yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-$(rpm -E %{rhel})-x86_64/pgdg-redhat-repo-latest.noarch.rpm
tee /etc/yum.repos.d/timescale_timescaledb.repo <<EOL
[timescale_timescaledb]
name=timescale_timescaledb
baseurl=https://packagecloud.io/timescale/timescaledb/el/$(rpm -E %{rhel})/\$basearch
repo_gpgcheck=1
gpgcheck=0
enabled=1
gpgkey=https://packagecloud.io/timescale/timescaledb/gpgkey
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
EOL

yum -y install timescaledb-2-postgresql-14
```

_Initilize the postgres database_

```
/usr/pgsql-14/bin/postgresql-14-setup initdb
```

_now, let's run postgres as a service on linux, and verify the postgres service status_

```
systemctl enable postgresql-14
systemctl start postgresql-14
systemctl status postgresql-14
```

_**REPEAT** the step in Each & Every machine_

### 2. repmgr Installation

---

_Why we are installing repmgr? because it's help to copy data from one postgres machine to another postgres machine, it's a third party package and widely used in postgres community for postgres replication._

_To [install](https://repmgr.org/docs/5.3/installation-packages.html) the repmgr in machine, run the below commands on terminal_

```
curl https://dl.enterprisedb.com/default/release/get/14/rpm | sudo bash
yum install repmgr14 -y
```

_**REPEAT** the step in Each & Every machine_

### 3. Hostname Resolution

---

_We are setting up postgres with 3 linux machine, here we are the ips and their hostname reference as, modify as per your structure_

_Now, let's add hosts as known host for the machine, login to hostname1 machine and run the below command_

```
cat >> /etc/hosts <<EOF
192.168.1.101 hostname1
192.168.1.102 hostname2
192.168.1.103 hostname3
EOF

cat /etc/hosts
```

_**REPEAT** the step in Each & Every machine_

### 4. SSH authentication with ssh-keygen

---

_Why we are doing ssh-keygen authentication, beacause repmgr needs the passwordless access between machines to do data replication._

#### 4.1 ssh-keygen in hostname1

_We will be generating rsa key and syncing with other machines, In here hostname1 machine will generate and sync with hostname2 and hostname3, so hostname2 and hostname3 machines user can login to hostname1 without any password_

_We need press mutliple [enters] after ssh-keygen commands for default key and location or else modify you like_

```
ssh-keygen -t rsa -b 4096
ssh-copy-id -i /root/.ssh/id_rsa.pub root@hostname2
ssh-copy-id -i /root/.ssh/id_rsa.pub root@hostname3
```

_And we need to modify ssh configuration, by doing below changes_

_Modify or add below keys in file as_

_[linux]: vi /etc/ssh/sshd_config_

```
RSAAuthentication yes
PubkeyAuthentication yes
```

_Restart the ssh service_

```
service sshd restart
```

#### 4.2 ssh-keygen in hostname2

_Same as step [4.1] by modifying carefully [like hostname2 -> hostname1]_

#### 4.3 ssh-keygen in hostname3

_Same as step [4.1] by modifying carefully [like hostname3 -> hostname1]_

### 5. Postgres configuration

---

_Here are setting up the postgres database and few machine configuration as_

##### _For linux machine configuration_

_When we install postgres than a default postgres user also get created in mcahine level, so we need to set password for the postgres user and make the postgres user as a sudo user for machine as well._

_making postgres as sudo user, by doing below changes carefully_

_[linux]: vi /etc/sudoers_

```
## Allow root to run any commands anywhere      # THIS LINE ALREADY EXISTS
root    ALL=(ALL)       ALL                     # THIS LINE ALREADY EXISTS
postgres ALL=(ALL)      ALL                     # ADD THIS LINE

...

## Same thing without a password                # THIS LINE ALREADY EXISTS
# %wheel        ALL=(ALL)       NOPASSWD: ALL   # THIS LINE ALREADY EXISTS
postgres        ALL=(ALL)       NOPASSWD: ALL   # ADD THIS LINE
```

_We made the postgres user as sudo, we need to set postgres user password as well, by running below commands as_

```
# Set the password you like to make sure
sudo passwd postgres
```

##### _For postgres configuration_

_To setup the postgres configuration, we need to modify the postgresql.conf, pg_hba.conf files and repmgr setting as well._

_To create repmgr setup_

_We will be creating repmgr user and repmgr database in postgres, by doing below changes_

```
su - postgres               # switching linux user(root) to postgres
createuser -s repmgr        # creating repmgr as superuser in postgres db
createdb repmgr -O repmgr   # creating repmgr database with owner repmgr user
```

_Modify or add the below lines in postgresql.conf, each line exists at different place in file or [refer](https://pgtune.leopard.in.ua/) to tune postgres conf._

_[linux]: vi /var/lib/pgsql/14/data/postgresql.conf_

```
wal_level = 'hot_standby'
hot_standby = on
archive_mode = on
archive_command = '/bin/true'
max_wal_senders = 10
wal_keep_size = 1024
shared_preload_libraries = 'repmgr'         # 'timescaledb,repmgr' - if installed
max_connections=1000
listen_addresses='*'
```

_Modify or add below lines in pg_hba.conf, these line mainly for repmgr user_

_[linux]: vi /var/lib/pgsql/14/data/pg_hba.conf_

```
local   replication,repmgr      repmgr                          trust
host    replication,repmgr      repmgr  127.0.0.1/32            trust
host    replication,repmgr      repmgr  192.168.1.0/24          trust
```

_Here, we used 192.168.1.0/24 because we have added ip configuration in step [3]_

_Finally, restart the postgresql server to apply the file changes_

```
systemctl restart postgresql-14
systemctl status postgresql-14
```

_**REPEAT** the step in Each & Every machine_

### 6. repmgr configuration

---

_Till now, we have setup our database and machines ready_

_Now, we will be starting the repmgr configuration to start the postgres replication, we need to choose our primary(master) node_

##### _repmgr primary node configuration_

_Here we refering hostname1 machine as primary node_

_We need to modify [repmgr configuration file](https://repmgr.org/docs/5.3/quickstart-repmgr-conf.html), each line exists at different place in file_

_[linux]: sudo vi /etc/repmgr/14/repmgr.conf_

```
node_id=1
node_name='hostname1'
conninfo='host=hostname1 user=repmgr dbname=repmgr connect_timeout=2'
priority=100
data_directory='/var/lib/pgsql/14/data'
# to define automatically failover
failover='automatic'
promote_command='/usr/pgsql-14/bin/repmgr standby promote --log-to-file'
follow_command='/usr/pgsql-14/bin/repmgr standby follow --log-to-file --upstream-node-id=%n'
```

_After changing repmgr configuration, we need to register with postgres as primary node, by following below lines_

```
su - postgres
/usr/pgsql-14/bin/repmgr -v primary register
```

_To verify the status_

```
su - postgres
/usr/pgsql-14/bin/repmgr -v cluster show
```

##### _repmgr secondary node configuration_

_Here we refering hostname2 and hostname3 machine as secondary node_

_We need to modify [repmgr configuration file](https://repmgr.org/docs/5.3/quickstart-repmgr-conf.html), each line exists at different place in file_

_[linux]: sudo vi /etc/repmgr/14/repmgr.conf_

```
node_id=2                                                               # change -> 3 for hostname3
node_name='hostname2'                                                   # change with 'hostname3'
conninfo='host=hostname2 user=repmgr dbname=repmgr connect_timeout=2'   # change with 'hostname3'
priority=99                                                             # change 99 -> 98
data_directory='/var/lib/pgsql/14/data'
# to define automatically failover
failover='automatic'
promote_command='/usr/pgsql-14/bin/repmgr standby promote --log-to-file'
follow_command='/usr/pgsql-14/bin/repmgr standby follow --log-to-file --upstream-node-id=%n'
```

_After changing repmgr configuration, we need to register with postgres as secondary node, by following below lines_

```
sudo systemctl disable postgresql-14
sudo systemctl stop postgresql-14

# to test primary server connectivity
su - postgres
psql 'host=hostname1 user=repmgr dbname=repmgr connect_timeout=2'

# pre-check standby with dry-run
/usr/pgsql-14/bin/repmgr -h hostname1 -U repmgr -d repmgr -v standby clone --dry-run

# MAIN COMMANDS - to backup primary node and register - it'll take a while to complete
/usr/pgsql-14/bin/repmgr -h hostname1 -U repmgr -d repmgr -v standby clone -F
sudo systemctl start postgresql-14
/usr/pgsql-14/bin/repmgr -v standby register
```

_To verify the status_

```
/usr/pgsql-14/bin/repmgr -v cluster show
```

_We have started the repmgr with postgres successfully_

_Connect your app to primary node for write and secondary for read operations if you like_

_If you are looking for auto primary node binding with your applications, please follow the step [7] to do so_

### 7. Always connect to primary

---

Why this point matters:

- What if your primary node went down - your app will loss connection with primary
- auto failover - it will be done by repmgr but your app will be pointing to previous primary
- What if primary node came up after long period down - your app will be point to old data state
- What if you want to make previous primary as secondary - data sync should be completed first

To answer these points, Here [**Infraon**](https://infraon.io) take place, Infraon gives a complete solutions out of the box by their product **InfraonHA** for above points, which take care all the failovers and data syncing part and **always connect** you app **to primary**.

_:)_

### ADDITIONAL

---

#### A. How to run postgres with docker compose

_**prerequisites**_

- Docker should be running
- Docker-compose should be installed

_navigate to the path of **docker-compose.yml**, and run the command as_

```
docker-compose up --build --force
```

#### B. How to convert standby to master purposefully

```
/usr/pgsql-14/bin/repmgr -v standby switchover
```

#### C. How to follow the master server if standby has some lack

_This command will be executed in standby server only_

```
/usr/pgsql-14/bin/repmgr -v standby follow
```

#### D. How to update bash_profile for postgres

_[linux]: sudo vi /var/lib/pgsql/.pgsql_profile_

```
PATH=$PATH:/usr/pgsql-14/bin
export PATH
```
