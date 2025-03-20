# MySQL Master-Slave Replication with Docker

### Step 1: Create Docker Network (IF YOU USE DOCKER)
```bash
# Create a network for containers to communicate
sudo docker network create mysql-net
```

---

### Step 2: Create Master Container Where Your Current Database Locate
```bash
# Start master container
sudo docker run --name supersee --network <NETWORK_NAME> -e MYSQL_ROOT_PASSWORD=root -p 3306:3306 -d mysql:8.0
```

---

### Step 3: For Setup master container as master slave you need to set the master container config but when your database are fully created and seted.
#### This configuration use config the master side database.

```bash
# first run this command to go bash of master container
docker exec -it supersee bash

cd /etc/mysql

# if my.cnf not created then create it first and then add this lines with this code
echo "[mysqld]
server-id=1
log-bin=mysql-bin 
binlog-do-db=supersee
binlog-format=ROW
bind-address=0.0.0.0
gtid_mode=ON
enforce-gtid-consistency=ON
" > my.cnf


# now we restart the container 
sudo docker restart supersee
```
- Note for master config file ` my.cnf ` | `C:\ProgramData\MySQL\MySQL Server 8.0\my.ini` use this configuration <br>
  ```
  [mysqld]
  server-id=1 
  log-bin=mysql-bin 
  binlog-do-db=supersee 
  binlog-format=ROW 
  bind-address=0.0.0.0 
  gtid_mode=ON
  enforce-gtid-consistency=ON
  ```

---

### Step 4: Configure Slave MySQL Container
#### Create Slave Container
```bash
# Start slave container
sudo docker run --name mysql-slave --network <NETWORK NAME> -e MYSQL_ROOT_PASSWORD=root -p 3308:3306 -d mysql:8.0 --server-id=2 --log-bin=mysql-bin --relay-log=mysql-relay-bin --binlog-format=ROW --gtid-mode=ON --enforce-gtid-consistency=ON
```
- Note for slave/replica config file ` my.cnf ` | `C:\ProgramData\MySQL\MySQL Server 8.0\my.ini` use this configuration <br>
  ```
  [mysqld]
  server-id=2
  relay-log=mysql-relay-bin
  log-bin=mysql-bin
  gtid_mode=ON
  enforce_gtid_consistency=ON
  ```

---

### Step 5: Configure Master for Replication
#### Connect to master
```bash
sudo docker exec -it supersee mysql -uroot -proot
```

#### Create user in master container or master side for slaves that can connect to master with this user.
```sql
CREATE USER 'raoinfotech'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'raoinfotech'@'%';
FLUSH PRIVILEGES;
```

#### Check Master Status
```sql
SHOW MASTER STATUS;
```

- To ensure data consistency when taking a backup of your MySQL database, especially for replication, you should lock the database to prevent any writes during the backup process.

#### After configured successfully user export backup database of current in master side.
```bash
USE <databasename>;

# For taking backup without lock the table and maintain consistency use this way
sudo docker exec supersee mysqldump -uroot -proot --single-transaction --master-data=2 --flush-logs --hex-blob --set-gtid-purged=ON --databases supersee > /your-path/backup.sql
```
- Note : <br>
`--single-transaction` is for without lock database it can create a backup files. <br>
`--master-data=2` for getting log_file_name log_file_pos in backup sql file.
`--set-gtid-purged=ON` for set the gtid status on

#### Now we first import backup of master database of master-data.sql in slave container.
```bash
sudo docker cp master-data.sql mysql-slave:/master-data.sql

# Direct import in MySQL Slave
SOURCE 'PATH'

# It will save in bash of container or without docker don't take this step those directly goto the next step of importing database in slave.
```

#### Import Data to the Slave
```
sudo docker exec -it mysql-slave bash

mysql -uroot -proot database_name < /master-data.sql

mysql -uroot -proot
```

---

### Step 6: Configure Slave for Replication

#### Connect to Slave
```bash
docker exec -it mysql-slave mysql -uroot -proot
```

#### Configure Slave
```sql
STOP SLAVE;
-- On Slave Side Use This Configuration.
-- With GTID Configuration.
CHANGE MASTER TO
  MASTER_HOST='Master Host Name Ex. supersee',
  MASTER_USER='<username>',
  MASTER_PASSWORD='<password>',
  MASTER_AUTO_POSITION=1;


START SLAVE;
```

#### Verify Slave Status
```sql
SHOW SLAVE STATUS\G
```

#### Check status of connection
```bash
# Check this configuration 
Master_Host: supersee
Master_User: raoinfotech
Master_Port: 3306
Connect_Retry: 60
Master_Log_File: mysql-bin.000002
Read_Master_Log_Pos: 157
Relay_Log_File: 2eac3f2cee88-relay-bin.000001
Relay_Log_Pos: 4
Relay_Master_Log_File: mysql-bin.000002



# Check the connection status
Slave_IO_Running: Connecting # THIS IS FOR IO CONNECTION Yes if all is set
Slave_SQL_Running: Yes # THIS IS FOR SQL DB CONNECTION Yes if all is set

# If any of the above is Not yes then check the log for error

# THIS WILL CHECK FOR LOGS OF IO
Last_IO_Errno: 0 # THIS WILL SHOW'S ERROR NO
Last_IO_Error: '' # THIS WILL SHOW'S ERROR CAUSE

# THIS WILL CHECK FOR LOGS OF SQL CONNECTION
Last_SQL_Errno: 0 # THIS WILL SHOW'S ERROR NO
Last_SQL_Error: '' # THIS WILL SHOW'S ERROR CAUSE

# FOR EX: CHECK THIS ERROR.
Last_IO_Errno: 2061
Last_IO_Error: Error connecting to source 'raoinfotech@supersee:3306'. This was attempt 1/86400, with a delay of 60 seconds between attempts. Message: Authentication plugin 'caching_sha2_password' reported error: Authentication requires secure connection.
  - NOTE : THIS ERROR CAUSE THE MASTER CONFIG IS SET AS A 'caching_sha2_password' TO FIX USE THIS

# ON MASTER CHANGE CONFIG TO GO IN MASTER DOCKER CONTAINER AND LOGIN IN MYSQL AND RUN THIS QUERY.
SELECT user, host, plugin FROM mysql.user WHERE user ='raoinfotech';

# THIS WILL PRINT SOMTHING LIKE THIS WAY
+-------------+------+-----------------------+
| user        | host | plugin                |
+-------------+------+-----------------------+
| raoinfotech | %    | caching_sha2_password |
+-------------+------+-----------------------+

# ON THIS U CAN SEE THE PLUGIN IS `caching_sha2_password` CHNAGE TO `mysql_native_password` BY THIS QUERY ON MASTER
ALTER USER 'raoinfotech'@'%' IDENTIFIED WITH mysql_native_password BY 'raoinfotech';
FLUSH PRIVILEGES;

# RESTART SLAVE OR REPLICA(WINDOWS)
STOP SLAVE;
START SLAVE;

```

---

### Step 7: Test Replication

#### Insert Data into Master
```sql
USE employees;
INSERT INTO departments (dept_no, dept_name) VALUES ('d010', 'New Department');
```

#### Check Data in Slave
```sql
USE employees;
SELECT * FROM departments;
```

If the data appears, replication is working correctly!

---

### Troubleshooting
- Check error logs in the containers:
```bash
docker logs supersee
docker logs mysql-slave
```
- Verify network connectivity:
```bash
docker network inspect mysql-net
```

### For GTID Information use this document :
- https://dev.mysql.com/doc/refman/8.4/en/replication-gtids-howto.html
