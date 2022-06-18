# Install Dependencies

Pull an Ubuntu 20.04 docker image and start a container. Install dependencies.

```bash
apt update && apt upgrade -y && apt install sudo build-essential git vim zsh cmake curl wget pkg-config libssl-dev libsnappy-dev libgflags-dev libreadline-dev bison libbz2-dev libaio-dev libtirpc-dev libudev-dev libcurl4-openssl-dev zlib1g-dev libncurses5-dev liblz4-dev libldap-dev libsasl2-dev libsasl2-modules-gssapi-mit libkrb5-dev openjdk-17-jdk maven -y
```

Install oh-my-zsh. This is not necessary, but it makes life easier.

```bash
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

# Set up MySQL

Download, build. and install Percona server and RocksDB trace analyzer.

```bash
cd ~
git clone https://github.com/CDSBen/percona-server.git -b 8.0 --depth 1
cd percona-server
git submodule update --init --recursive
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Debug -DWITH_SSL=system -DWITH_ZLIB=bundled -DMYSQL_MAINTAINER_MODE=1 -DENABLE_DTRACE=1 -DDOWNLOAD_BOOST=1 -DWITH_BOOST=boost
sudo make install
cd ../storage/rocksdb/rocksdb
make trace_analyzer
```

Start MySQL.

```bassh
cd /usr/local/mysql/bin
./mysqld --initialize-insecure --gdb
./mysqld --user=root
```

In another terminal, enable RocksDB storage engine and connect to MySQL.

```bash
./ps-admin --enable-rocksdb -uroot
./mysql -uroot
```

Change the password of MySQL.

```SQL
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'pswd000111';
GRANT ALL ON *.* TO 'root'@'localhost' WITH GRANT OPTION;
CREATE USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'pswd000111';
GRANT ALL ON *.* TO 'root'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

Now, the RocksDB storage engine is added to MySQL but is not the default engine. Therefore, we first shut it down and then restart it.

```bash
./mysqladmin -uroot -ppswd000111 shutdown
./mysqld --user=root --default-storage-engine=rocksdb --rocksdb-trace-queries="1:1024:query_trace" --rocksdb-trace-sst-api=ON
```

Connect to MySQL again and list the engines.

```mysql
show engines;
```

```
mysql> show engines;
+--------------------+---------+----------------------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                                    | Transactions | XA   | Savepoints |
+--------------------+---------+----------------------------------------------------------------------------+--------------+------+------------+
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                                         | NO           | NO   | NO         |
| ROCKSDB            | DEFAULT | RocksDB storage engine                                                     | YES          | YES  | YES        |
| InnoDB             | YES     | Percona-XtraDB, Supports transactions, row-level locking, and foreign keys | YES          | YES  | YES        |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables                  | NO           | NO   | NO         |
| MyISAM             | YES     | MyISAM storage engine                                                      | NO           | NO   | NO         |
| FEDERATED          | NO      | Federated MySQL storage engine                                             | NULL         | NULL | NULL       |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                                      | NO           | NO   | NO         |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears)             | NO           | NO   | NO         |
| CSV                | YES     | CSV storage engine                                                         | NO           | NO   | NO         |
| ARCHIVE            | YES     | Archive storage engine                                                     | NO           | NO   | NO         |
+--------------------+---------+----------------------------------------------------------------------------+--------------+------+------------+
10 rows in set (0.00 sec)
```

# Set up TPC-C

Install an open source implementation of TPC-C, benchbase. Since the RocksDB storage engine has limited support over foreign key and isolation level, we slightly modify benchbase.
Compile benchbase.

```bash
cd ~
git clone --depth 1 https://github.com/wateryloo/benchbase.git
cd benchbase
./mvnw clean package -P mysql -DskipTests
cd target
tar -xvf benchbase-mysql.tgz
cd benchbase-mysql
```

Edit the config file.

```bash
vim config/mysql/sample_tpcc_config.xml
```

Modify line 7, 8, 9,

```xml
<?xml version="1.0"?>
<parameters>

    <!-- Connection details -->
    <type>MYSQL</type>
    <driver>com.mysql.cj.jdbc.Driver</driver>
    <url>jdbc:mysql://localhost:3306/tpcc_10?rewriteBatchedStatements=true&sslMode=DISABLED</url>
    <username>root</username>
    <password>pswd000111</password>
    <isolation>TRANSACTION_READ_COMMITTED</isolation>
    <batchsize>128</batchsize>

    <!-- Scale factor is the number of warehouses in TPCC -->
    <scalefactor>10</scalefactor>

    <!-- The workload -->
    <terminals>10</terminals>
    <works>
        <work>
            <time>60</time>
            <rate>10000</rate>
            <weights>45,43,4,4,4</weights>
        </work>
    </works>

    <!-- TPCC specific -->
    <transactiontypes>
        <transactiontype>
            <name>NewOrder</name>
        </transactiontype>
        <transactiontype>
            <name>Payment</name>
        </transactiontype>
        <transactiontype>
            <name>OrderStatus</name>
        </transactiontype>
        <transactiontype>
            <name>Delivery</name>
        </transactiontype>
        <transactiontype>
            <name>StockLevel</name>
        </transactiontype>
    </transactiontypes>
</parameters>
```

Connect MySQL and create the corresponding tpcc_10 database.

```bash
cd /usr/local/mysql/bin
./mysql -uroot -ppswd000111
```

```sql
create database tpcc_10;
```

Load data.

```bash
cd ~/benchbase/target/benchbase-mysql
java -jar benchbase.jar -b tpcc -c config/mysql/sample_tpcc_config.xml --create=true --load=true
```

# References

1. https://www.percona.com/doc/percona-server/8.0/myrocks/install.html
