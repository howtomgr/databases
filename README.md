## MySQL/MariaDB

```bash
echo '[mariadb]
name = MariaDB
baseurl = https://mirror.its.dal.ca/mariadb/yum/10.6/centos$releasever-$basearch
module_hotfixes=1
gpgkey=https://mirror.its.dal.ca/mariadb/yum/RPM-GPG-KEY-MariaDB
gpgcheck=1' | tee /etc/yum.repos.d/MariaDB.repo &>/dev/null
sudo yum install mariadb-server
sudo systemctl enable --now mariadb
sudo mysql_secure_installation

```

## PostgreSQL

```bash
sudo yum install https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-$releasever-$basearch/pgdg-centos96-9.6-3.noarch.rpm
sudo yum install postgresql96-server
sudo /usr/pgsql-9.6/bin/postgresql96-setup initdb
```

<https://comtronic.com.au/how-to-upgrade-postgresql-server-to-9-6/>  

## Microsoft SQL Server

```bash
echo '[packages-microsoft-com-mssql-server-2019]
name=packages-microsoft-com-mssql-server-2019
baseurl=https://packages.microsoft.com/rhel/$releasever/mssql-server-2019/
enabled=1
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc' | tee /etc/yum.repos.d/mssql-server.repo &>/dev/null
yum install -y mssql-server
/opt/mssql/bin/mssql-conf setup
systemctl enable --now mssql-server
```

## MongoDB

```bash
echo '[mongodb-org-5.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/5.0/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-5.0.asc' | tee /etc/yum.repos.d/mongodb-org-5.0.repo &>/dev/null
yum install -y mongodb-org
systemctl enable --now mongod
```

## CouchDB

```bash
echo '[bintray--apache-couchdb-rpm]
name=bintray--apache-couchdb-rpm
baseurl=http://apache.bintray.com/couchdb-rpm/el$releasever/$basearch/
gpgcheck=0
repo_gpgcheck=0
enabled=1' | tee /etc/yum.repos.d/apache-couchdb.repo &>/dev/null
yum install -y couchdb
systemctl enable --now couchdb
```

