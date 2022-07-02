




# Debezium + Proxy sql failover Demo

This demo aims to show Debezium failover behaviour when working with proxy sql.
it is heavily influenced by the debezium + HA proxy demo which can be seen here:

https://github.com/debezium/debezium-examples/tree/main/failover 

its important to mention that the connector which will be created via the request file
will be set to work with the proxy sql address.


![alt text](https://github.com/AviMualem/debezium-proxysql-failover/blob/main/demo.jpeg?raw=true)



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

loading the environment
```
export DEBEZIUM_VERSION=1.8
docker-compose up --build
```

### Creating proxy sql monitor user in mysql:
proxy sql rquires monitoring user to be confiured in mysql,  you can create in in one of the mysql servers because
user will be replicated between mysql server 1 and mysql server 2

```
// make sure you are in terminal in the folder of the compose file

docker-compose exec mysql1 bash -c 'mysql -u root -pdebezium inventory'

// creating the user

CREATE USER 'proxysql'@'%' IDENTIFIED WITH mysql_native_password by '$3Kr$t';
GRANT USAGE ON *.* TO 'proxysql'@'%';
FLUSH privileges;
```

### Configure proxy sql
start a ssh session to the proxy sql and run the following.

```
// connect to proxy sql
// make sure you are in terminal in the folder of the compose file

docker-compose exec proxysql bash -c 'mysql -u admin -padmin -h 127.0.0.1 -P 6032'

// adding the two mysql servers to proxy sql topology 
// (the 10000000 weight on mysql 1 will make sure it will function as "primary" if available) 
// you can feel free to change it to suite your needs.

INSERT INTO mysql_servers(hostgroup_id,hostname,port,weight) VALUES (0,'mysql1',3306,10000000);

INSERT INTO mysql_servers(hostgroup_id,hostname,port,weight) VALUES (0,'mysql2',3306,1);

// set the proxy sql monitor user we created before on mysql servers
UPDATE global_variables SET variable_value='proxysql' WHERE variable_name='mysql-monitor_username';
UPDATE global_variables SET variable_value='$3Kr$t' WHERE variable_name='mysql-monitor_password';

// creating user which will be used by the connenctor
 INSERT INTO mysql_users (username,password,fast_forward) VALUES ('debezium','dbz',1);

 LOAD MYSQL USERS TO RUNTIME;
 SAVE MYSQL USERS TO DISK;

// allows connections from mysql workbench
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

### Creating connector
Start the components and register Debezium to stream changes from the database
```
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" http://localhost:8083/connectors/ -d @register-mysql.json
```

### Where can i see my messages 
messages can be seen at http://localhost:8000 in the customer topic


### Connect -directly- to MySQL 1, check server UUID and create two records
```
// make sure you are in terminal in the folder of the compose file

docker-compose exec mysql1 bash -c 'mysql -u root -pdebezium inventory'

// get the uuid of the server
  SHOW GLOBAL VARIABLES LIKE 'server_uuid';
  
// insert 2 new records
  INSERT INTO customers VALUES (default, 'John','Doe','john.doe@example.com');
  INSERT INTO customers VALUES (default, 'Jane','Doe','jane.doe@example.com');
  
  
 // Check UUID in change message, the 'source' will contain the uuid that  you just queried.
```

### Initiate a failover

Stop the mysql1 server
```
docker-compose stop mysql1
```
### Connect -directly- to MySQL 2, check server UUID and create two records

```
// make sure you are in terminal in the folder of the compose file

docker-compose exec mysql2 bash -c 'mysql -u root -pdebezium inventory'

// get the uuid of the server
  SHOW GLOBAL VARIABLES LIKE 'server_uuid';
  
  // insert 2 new records
  INSERT INTO customers VALUES (default, 'Peter','Doe','peter.doe@example.com');
  INSERT INTO customers VALUES (default, 'Paul','Doe','paul.doe@example.com');
  
   //Check UUID in change message, the 'source' will contain the uuid that  you just queried.
```

you can see that streaming messages are now being fetched from mysqlserver 2 and messages will have the mysql2 server uuid.

you are free to put any of ther servers up and down and see that debezium keeps on working as expected and able to fetch data from both servers while keeping track of GTID

### How to start and stop mysql servers in order to check various failover scenarios
```
docker-compose stop mysql1
docker-compose start mysql1

docker-compose stop mysql2
docker-compose start mysql2


```

### Stop the demo
```
docker-compose down
```
### script to kill all containers :)
```
docker rm -f $(docker ps -qa)
```
