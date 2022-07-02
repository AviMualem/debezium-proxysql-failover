# Debezium + Proxy sql failover Demo

This demo aims to show Debezium failover behaviour when working with proxy sql.
connector which will be created via the request file will be set to work with the proxy sql address.

## Topology

The deployment consists of the following components

* Database
  * MySQL 1 instance (configured as a slave to MySQL 2) with GTID enabled
  * MySQL 2 instance (configured as a slave to MySQL 1) with GTID enabled
* Proxy
	* proxy sql (https://proxysql.com/)
* Streaming system
  * Apache ZooKeeper
  * Apache Kafka broker
  * Apache Kafka Connect with Debezium MySQL Connector - the connector will connect to proxy sql

## Demonstration

###creating proxy sql monitor user in mysql:
proxy sql rquires monitoring user to be confiured in mysql,  you can create in in one of the mysql servers because user will be replicated between mysql server 1 and mysql server 2
```
CREATE USER 'proxysql'@'%' IDENTIFIED WITH mysql_native_password by '$3Kr$t';
GRANT USAGE ON *.* TO 'proxysql'@'%';
FLUSH privileges;
```

### configure proxy sql
start a ssh session to the proxy sql and run the following.

```
mysql -u admin -padmin -h 127.0.0.1 -P 6032
---
INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (0,'mysql1',3306);

INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (0,'mysql2',3306);

UPDATE global_variables SET variable_value='proxysql' WHERE variable_name='mysql-monitor_username';
UPDATE global_variables SET variable_value='$3Kr$t' WHERE variable_name='mysql-monitor_password';

 INSERT INTO mysql_users (username,password,fast_forward) VALUES ('debezium','dbz',1);

 LOAD MYSQL USERS TO RUNTIME;
 SAVE MYSQL USERS TO DISK;

update global_variables set variable_value='false' where variable_name='admin-hash_passwords';

load admin variables to runtime; 
save admin variables to disk;
load mysql users to runtime;
save mysql users to disk;
LOAD MYSQL SERVERS TO RUNTIME;
LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;   
LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;

```

### creating connector
Start the components and register Debezium to stream changes from the database
```
export DEBEZIUM_VERSION=1.8
docker-compose up --build
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" http://localhost:8083/connectors/ -d @register-mysql.json
```

### where can i see my messages 
messages can be seen at http://localhost:8000 in the customer topic


### Connect to MySQL 1, check server UUID and create two records
```
docker-compose exec mysql1 bash -c 'mysql -u root -p$MYSQL_ROOT_PASSWORD inventory'

  SHOW GLOBAL VARIABLES LIKE 'server_uuid';
  INSERT INTO customers VALUES (default, 'John','Doe','john.doe@example.com');
  INSERT INTO customers VALUES (default, 'Jane','Doe','jane.doe@example.com');
  
  
 ###  Check UUID in change message, the 'source' will contain field "gtid":"50303655-f22a-11e8-92a5-0242ac1d0003:2"
```

### initiate a failover

Stop the mysql1 server
```
docker-compose stop mysql1
```
Create two more records in the mysql2 database  and check the `UUID` of the mysql2 server.
```
# Connect to MySQL 2, check server UUID and create two records
docker-compose exec mysql2 bash -c 'mysql -u root -p$MYSQL_ROOT_PASSWORD inventory'
  SHOW GLOBAL VARIABLES LIKE 'server_uuid';
  INSERT INTO customers VALUES (default, 'Peter','Doe','peter.doe@example.com');
  INSERT INTO customers VALUES (default, 'Paul','Doe','paul.doe@example.com');
```

you can see that streaming messages are now being fetched from mysqlserver 2 and messages will have the mysql2 server uuid.

you are free to put any of ther servers up and down and see that debezium keeps on working as expected.

Stop the demo
```
docker-compose down
```
