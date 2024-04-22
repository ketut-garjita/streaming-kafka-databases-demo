# streaming-kafka-databases-demo

## Install databases

*Only needs to be done once*

Testing :
- mysql (linux)
- postgresql (windows)
- oracle (windows)
- duckdb (linux)


## Prepare Database Table 

*Only needs to be done once*

Open session terminal (Session 1)

- Start mysql server

  ```
  sudo systemctl start mysql
  ```

- Connect to the mysql server
  
  Make sure you use the password given to you when the MySQL server starts.
  
  ```
  mysql --host=127.0.0.1 --port=3306 --user=root --password
  ```

- Create a database and table

  ```
  create database tolldata;
  use tolldata;
  create table livetolldata ( timestamp datetime , vehicle_id int , vehicle_type char ( 15 ), toll_plaza_id smallint );
  ```

- Same for another database platforms (postgresql, duckdb, oracle)

  
## Install Kafka

*Only needs to be done once*

Open a new session

- Download kafka ((prefer latest version, example : https://archive.apache.org/dist/kafka/3.6.2/kafka_2.12-3.6.2.tgz)
  ```
  wget https://archive.apache.org/dist/kafka/3.6.2/kafka_2.12-3.6.2.tgz
  ```

- Extract Kafka.
  ```
  tar -xzf kafka_2.12-3.6.2.tgz
  ```

## Install "kafka-python" python module

*Only needs to be done once*

```
python3 -m pip install kafka-python
```


## Install Database Connection Library

*Only needs to be done once*

- Mysql (ver.8.0)
  ```
  pip install mysql-connector-python==8.0.33
  ```

- PostgreSQL
  ```
  pip install psycopg2
  ```

- Duckdb
  ```
  pip install duckdb
  ```

- Oracle
  ```
  pip install cx_Oracle
  ```
    

## Start Kafka
  
- Start Zookeeper

  Open new Session
  
  ```
  cd $KAFKA_HOME
  bin/zookeeper-server-start.sh config/zookeeper.properties
  ```

-  Start Kafka Server

    Open new Session

    ```
    cd $KAFKA_HOME
    bin/kafka-server-start.sh config/server.properties
    ```

- Create a Topic

  Open new Session

  Topic name : toll

  ```
  cd $KAFKA_HOME
  bin/kafka-topics.sh --create --topic toll quickstart-events --bootstrap-server localhost:9092
  ```

- List a topic
  ```
  cd $KAFKA_HOME
  bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
  ```

- Describe a topic
  ```
  cd $KAFKA_HOME
  bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic toll
  ```
  

## Create Toll Traffic Simulator

*Only needs to be done once*

Open new Session

Create file : $KAFKA_HOME/scripts/toll_traffic_generator.py

```
"""
Top Traffic Simulator
"""
from time import sleep, time, ctime
from random import random, randint, choice
from kafka import KafkaProducer
producer = KafkaProducer(bootstrap_servers='localhost:9092')

TOPIC = 'toll'

VEHICLE_TYPES = ("car", "car", "car", "car", "car", "car", "car", "car",
                 "car", "car", "car", "truck", "truck", "truck",
                 "truck", "van", "van")
for _ in range(100000):
    vehicle_id = randint(10000, 10000000)
    vehicle_type = choice(VEHICLE_TYPES)
    now = ctime(time())
    plaza_id = randint(4000, 4010)
    message = f"{now},{vehicle_id},{vehicle_type},{plaza_id}"
    message = bytearray(message.encode("utf-8"))
    print(f"A {vehicle_type} has passed by the toll plaza {plaza_id} at {now}.")
    producer.send(TOPIC, message)
    sleep(random() * 2)
```


## Run the Toll Traffic Simulator

Open new Session

```
cd $KAFKA_HOME/scripts
python3 toll_traffic_generator.py
```


## Create Data Streaming Script

MySQL  : $KAFKA_HOME/scripts/streaming_data_reader_mysql.py
  
  ```
  """
  Streaming data consumer
  """
  from datetime import datetime
  from kafka import KafkaConsumer
  import mysql.connector
  
  TOPIC='toll'
  DATABASE = 'tolldata'
  USERNAME = 'root'
  PASSWORD = 'mysql'
  
  print("Connecting to the database")
  try:
      connection = mysql.connector.connect(host='localhost', database=DATABASE, user=USERNAME, password=PASSWORD)
  except Exception:
      print("Could not connect to database. Please check credentials")
  else:
      print("Connected to database")
  cursor = connection.cursor()
  
  print("Connecting to Kafka")
  consumer = KafkaConsumer(TOPIC)
  print("Connected to Kafka")
  print(f"Reading messages from the topic {TOPIC}")
  for msg in consumer:
  
      # Extract information from kafka
  
      message = msg.value.decode("utf-8")
  
      # Transform the date format to suit the database schema
      (timestamp, vehcile_id, vehicle_type, plaza_id) = message.split(",")
  
      dateobj = datetime.strptime(timestamp, '%a %b %d %H:%M:%S %Y')
      timestamp = dateobj.strftime("%Y-%m-%d %H:%M:%S")
  
      # Loading data into the database table
  
      sql = "insert into livetolldata values(%s,%s,%s,%s)"
      result = cursor.execute(sql, (timestamp, vehcile_id, vehicle_type, plaza_id))
      print(f"A {vehicle_type} was inserted into the database")
      connection.commit()
  connection.close()
  ```

PostgresSQL : $KAFKA_HOME/scripts/streaming_data_reader_postgres.py
  
  ```
  """
  Streaming data consumer
  """
  from datetime import datetime
  from kafka import KafkaConsumer
  import psycopg2
  import sys
  
  con = None
  
  TOPIC='toll'
  DATABASE = 'tolldata'
  USERNAME = 'postgres'
  PASSWORD = 'postgres'
  
  print("Connecting to the database")
  try:
          connection = psycopg2.connect(host='192.168.68.1', database=DATABASE, user=USERNAME, password=PASSWORD)
  except Exception:
          print("Could not connect to database. Please check credentials")
  else:
          print("Connected to database")
  cursor = connection.cursor()
  
  print("Connecting to Kafka")
  consumer = KafkaConsumer(TOPIC)
  print("Connected to Kafka")
  print(f"Reading messages from the topic {TOPIC}")
  for msg in consumer:
  
      # Extract information from kafka
  
      message = msg.value.decode("utf-8")
  
      # Transform the date format to suit the database schema
      (timestamp, vehcile_id, vehicle_type, plaza_id) = message.split(",")
  
      dateobj = datetime.strptime(timestamp, '%a %b %d %H:%M:%S %Y')
      timestamp = dateobj.strftime("%Y-%m-%d %H:%M:%S")
  
      # Loading data into the database table
  
      sql = "insert into livetolldata values(%s,%s,%s,%s)"
      result = cursor.execute(sql, (timestamp, vehcile_id, vehicle_type, plaza_id))
      print(f"A {vehicle_type} was inserted into the database")
      connection.commit()
  connection.close()
  ```

DuckDB : $KAFKA_HOME/scripts/streaming_data_reader_duckdb.py

  ```
  """
  Streaming data consumer
  """
  from datetime import datetime
  from kafka import KafkaConsumer
  
  from duckdb import connect
  
  TOPIC='toll'
  
  print("Connecting to the database")
  try:
          connection = connect(database='tolldata.db', read_only=False)  # Change the database file name as needed
  except Exception:
          print("Could not connect to database. Please check credentials")
  else:
          print("Connected to database")
  cursor = connection.cursor()
  
  print("Connecting to Kafka")
  consumer = KafkaConsumer(TOPIC)
  print("Connected to Kafka")
  print(f"Reading messages from the topic {TOPIC}")
  for msg in consumer:
  
      # Extract information from kafka
  
      message = msg.value.decode("utf-8")
  
      # Transform the date format to suit the database schema
      (timestamp, vehcile_id, vehicle_type, plaza_id) = message.split(",")
  
      dateobj = datetime.strptime(timestamp, '%a %b %d %H:%M:%S %Y')
      timestamp = dateobj.strftime("%Y-%m-%d %H:%M:%S")
  
      # Loading data into the database table
  
      sql = "insert into livetolldata values(?,?,?,?)"
      result = cursor.execute(sql, (timestamp, vehcile_id, vehicle_type, plaza_id))
      print(f"A {vehicle_type} was inserted into the database")
      connection.commit()
  connection.close()
  ```

  
## Run Data Streaming (Testing)

1. MySQL   
   ```
   python streaming_data_reader_mysql.py
   ```

3. PostgreSQL
   ```
   python streaming_data_reader_postgres.py
   ```

5. DuckDB
   ```
   python streaming_data_reader_pduckdb.py
   ```
   

## Health check of the streaming data pipeline

- MySQL
  ```
  mysql --host=127.0.0.1 --port=3306 --user=root --password
  
  use tolldata;
  select count(*) from livetolldata;
  select * from livetolldata limit 10;
  ```

- PostgreSQL
  ```
  psql -U postgres
  Password: ......
  \c tolldata
  select count(*) from livetolldata;
  select * from livetolldata limit 10;
  ```

- Duckdb

  Stop (Ctrl-C) duckDB data streaming
  ```
  cd $KAFKA_HOME/scripts
  duckdb  tolldata.db
  select count(*) from livetolldata;
  select * from livetolldata limit 10;
  ```
 
