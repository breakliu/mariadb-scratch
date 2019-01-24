# MariaDB Scratch
> This repo is for MariaDB official package installation.

# Installation

## 1. Download
* mariadb-10.3.12-linux-systemd-x86_64.tar.gz [link](http://downloads.mariadb.com/MariaDB/mariadb-10.3.12/bintar-linux-systemd-x86_64/mariadb-10.3.12-linux-systemd-x86_64.tar.gz)

## 2. Unpack and Install manually
```
groupadd mysql
useradd -g mysql mysql

mkdir -p /data/mariadb/conf
mkdir -p /data/mariadb/logs
mkdir -p /data/mariadb/run
mkdir -p /data/mariadb/tmp

chown -R mysql:mysql /data/mariadb/logs
chown -R mysql:mysql /data/mariadb/run
chown -R mysql:mysql /data/mariadb/tmp

cd /data/mariadb
tar xvf /data/downloads/mariadb-10.3.12-linux-systemd-x86_64.tar.gz -C .
ln -sv mariadb-10.3.12-linux-systemd-x86_64 env
cd env
chown -R mysql:mysql .

# edit my.cnf.example
cp my.cnf.example /data/mariadb/conf/

./scripts/mysql_install_db --user=mysql --defaults-file=/data/mariadb/conf/my.cnf
chown -R root .
chown -R mysql /data/mariadb/data

# for test
# ./bin/mysqld --defaults-file=/data/mariadb/conf/my.cnf --user=mysql
# ./bin/mysql -h 127.0.0.1
```

## 3. Encryption ( Optional )
* Generate keys
    ```
    openssl rand -hex 32 >> /data/mariadb/conf/file_key.txt
    openssl rand -hex 32 >> /data/mariadb/conf/file_key.txt
    openssl rand -hex 32 >> /data/mariadb/conf/file_key.txt
    ```
* Edit file_key.txt
    ```
    # MariaDB encryption file key
    1;5d9c6e89f8d5dde3c6d7196bb283f298d3d19da9cec53fb4a6e9aa55044acc52
    2;5430ffa7136092956181e58ec350f2ebca8096be7c1a78bf64d63771b0245874
    3;4acbbb47ce9fa291a24cca66b7a97627ee6ec455a13832e27fb6b54f3f67b936
    ```
* Update my.cnf
    * Not encrypt file_key.txt
        ```
        ### edit my.cnf

        # Encryption
        plugin_dir                                  = /data/mariadb/env/lib/plugin
        plugin-load-add                             = file_key_management.so
        file_key_management_filename                = /data/mariadb/conf/file_key.txt

        innodb-encrypt-tables
        innodb-encrypt-log
        innodb-encryption-threads                   = 4

        encrypt-binlog
        encrypt_tmp_files
        encrypt_tmp_disk_tables
        aria_encrypt_tables
        ```
    * Encrypt file_key.txt
        ```
        # ASE encrypt file_key.txt
        openssl enc -aes-256-cbc -md sha1 -k file_key_encrypt_key -in file_key.txt -out file_key_enc.txt
        ```
        ```
        ### edit my.cnf

        #file_key_management_filename                = /data/mariadb/conf/file_key.txt
        file_key_management_filename                = /data/mariadb/conf/file_key_enc.txt
        file_key_management_encryption_algorithm    = aes_cbc
        file_key_management_filekey                 = file_key_encrypt_key
        ```
* Check
    * mysql: `show status LIKE '%encrypt%';`

## 4. Systemd
* Default
    ```
    support-files/systemd/mariadb.service
    ```
* Customization
    ```
    # mariadb.service.example
    Alias=mysql_custom.service
    Alias=mysqld_custom.service
    Alias=mariadb_custom.service

    Environment="MYSQLD_OPTS=--defaults-file=/data/mariadb/conf/my.cnf"
    ExecStart=/data/mariadb/env/bin/mysqld $MYSQLD_OPTS $_WSREP_NEW_CLUSTER $_WSREP_START_POSITION
        
    LimitNOFILE=102400
    ```
* Systemd control
    ```
    cp mariadb.service.example /usr/lib/systemd/system/mariadb_custom.service

    systemctl daemon-reload

    # auto start
    systemctl enable mariadb_custom

    # start, stop , restart
    systemctl start mariadb_custom
    systemctl stop mariadb_custom
    systemctl restart mariadb_custom
    ```
## 5. SQL
* Create unencrypt table
    ```
    create table unencrypt_t(id int, name varchar(32)) ENCRYPTED=NO;
    ```
* Create encrypt table with encryption key
    ```
    # default key id = 1
    CREATE TABLE encrypt_t(id INT, name VARCHAR(32)) ENCRYPTED=YES;

    # key id = 3
    CREATE TABLE encrypt_t(id INT, name VARCHAR(32)) ENCRYPTED=YES ENCRYPTION_KEY_ID=3;
    ```
* Alter unencrypt -> encrypt
    ```
    ALTER TABLE unencrypt_t ENCRYPTED=YES;
    ```
* Alter encrypt -> unencrypt
    ```
    ALTER TABLE unencrypt_t ENCRYPTED=NO;
    ```
* Check
    ```
    SHOW TABLE STATUS\G
    ```

## 6. Reference
* https://cloud.tencent.com/developer/article/1004438
* https://tools.percona.com/wizard
* 加密开启后的主备同步
    * 开启加密后，主机和备机之间的binlog传输是不加密的，由备机在写relaylog/binlog/数据文件时进行加密。所以主备之间的密钥可以不同，但id信息必须一致，否则建表语句在备机上无法执行成功，将会导致slave SQL线程中止。
* 加密和压缩
    * 数据加密和数据压缩可以同时使用，MariaDB先做数据压缩再做数据加密，可以节约很大的存储空间。