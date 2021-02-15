# Step By Step Script to implement CDC using KSQL

 ## Kafka/zookeeper
 - docker-compose up

 ## Postgres
 - docker run -it -p 5432:5432 -e POSTGRES_PASSWORD=postgres -e POSTGRES_USER=postgres  -e POSTGRES_DB=example_db -v $PWD/pgdata:/var/lib/postgresql/data debezium/postgres
 - export PGPASSWORD='postgres'; psql -h 'localhost' -U 'postgres' -d 'example_db' ;
 - ```
     CREATE TABLE customers (
       id SERIAL,
       first_name VARCHAR(255) NOT NULL,
       last_name VARCHAR(255) NOT NULL,__
       email VARCHAR(255) NOT NULL,
       PRIMARY KEY(id)
     );
     
      insert into customers (id,first_name,last_name,email) values (1,'Test','User','abc@gmail.com');
      insert into customers values (2,'Test2','User2','abc@gmail.com');
      update customers set first_name='Test3', last_name='User3' where id=2;
      delete from customers where id=1;
      
     CREATE TABLE payments (
         id SERIAL,
         product_name VARCHAR(255) NOT NULL,
         amount integer NOT NULL,
         unit integer NOT NULL,
         customer_id integer NOT NULL,
         PRIMARY KEY(id)
       );
     
       insert into payments values (1,'prod1',235.50,2,2);
       insert into payments values (3,'prod2',23,4,1);        
    ```
 -      

 ## KSQL DB
 - docker run -it -p 8088:8088 --env-file ./ksql_server.list  confluentinc/ksqldb-server:0.13.0 

  
 ## Debezium Postgres Connector
 - docker run -it --rm -p 8083:8083 -e GROUP_ID=1 -e BOOTSTRAP_SERVERS=IP_ADDRESS:9092 -e GROUP_ID=sde_group -e CONFIG_STORAGE_TOPIC=sde_storage_topic -e OFFSET_STORAGE_TOPIC=sde_offset_topic  debezium/connect
 
 - curl -H "Accept:application/json" localhost:8083/connectors/
 []
 
 ## Add Source Connector
 ### Whitelist Version
 - curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" \
 localhost:8083/connectors/ -d '{"name": "sde-connector", "config": {"connector.class": "io.debezium.connector.postgresql.PostgresConnector", "database.hostname": "IP_ADDRESS", "database.port": "5432", "database.user": "postgres", "database.password": "postgres", "database.dbname" : "example_db", "database.server.name": "example_db", "table.whitelist": "bank.holding"}}'
 
 ### Read All
 - curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" \
  localhost:8083/connectors/ -d '{"name": "sde-connector", "config": {"connector.class": "io.debezium.connector.postgresql.PostgresConnector", "database.hostname": "IP_ADDRESS", "database.port": "5432", "database.user": "postgres", "database.password": "postgres", "database.dbname" : "example_db", "database.server.name": "example_db"}}'

 ## KSQL CLI
 - docker run -it confluentinc/ksqldb-cli:0.13.0 ksql http://IP_ADDRESS:8088  
  
   ---
   ```
       SET 'auto.offset.reset' = 'earliest';
       create stream customer_stream (payload STRUCT<after STRUCT<id INTEGER,first_name VARCHAR, last_name VARCHAR,email VARCHAR>>) with (KAFKA_TOPIC='example_db.public.customers',VALUE_FORMAT='JSON');
       
       create stream customer_stream (payload STRUCT<after STRUCT<id INTEGER,first_name VARCHAR, last_name VARCHAR,email VARCHAR>, op VARCHAR>) with (KAFKA_TOPIC='example_db.public.customers',VALUE_FORMAT='JSON');
       select * from customer_stream emit changes;
       create stream customer_dtl as select PAYLOAD->AFTER->ID as id,PAYLOAD->AFTER->FIRST_NAME as first_name,PAYLOAD->AFTER->LAST_NAME as last_name,PAYLOAD->AFTER->EMAIL as email, PAYLOAD->OP as OP from customer_stream PARTITION BY PAYLOAD->AFTER->ID;
       select * from customer_dtl emit changes;
       create table customer_tbl (id integer primary key, first_name varchar, last_name varchar, email varchar, op varchar) with (kafka_topic='CUSTOMER_DTL', value_format='JSON');
       select * from customer_tbl emit changes;
       
      
       create stream payments_stream (payload STRUCT<after STRUCT<id INTEGER,product_name VARCHAR, amount INTEGER,unit INTEGER,customer_id INTEGER>, op VARCHAR>) with (KAFKA_TOPIC='example_db.public.payments',VALUE_FORMAT='JSON');
       select * from payments_stream emit changes;
       create stream payments_dtl as select PAYLOAD->AFTER->ID as id,PAYLOAD->AFTER->PRODUCT_NAME as PRODUCT_NAME,PAYLOAD->AFTER->AMOUNT as AMOUNT,PAYLOAD->AFTER->UNIT as UNIT,PAYLOAD->AFTER->CUSTOMER_ID as CUSTOMER_ID, PAYLOAD->OP as OP from payments_stream PARTITION BY PAYLOAD->AFTER->ID;
       select * from payments_dtl emit changes;
      
    
      
       create stream transaction as select c.id as id, c.first_name, c.last_name, c.email, p.product_name, p.amount, p.unit, p.amount*p.unit as total, c.op, p.op from payments_dtl p left join customer_tbl c on c.id = p.customer_id emit changes;
       select * from transaction emit changes;
   ```
  
  
## ElasticSearch Sink Connector

- curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" \
  localhost:8083/connectors/ -d '{"name":"es-sink","config":{"connector.class":"io.confluent.connect.elasticsearch.ElasticsearchSinkConnector","tasks.max":"1","topics":"TRANSACTION","key.ignore":"true","schema.ignore":"true","connection.url":"http://IP_ADDRESS:9200","type.name":"test-type","name":"es-sink"}}'

- curl -X GET 'http://localhost:9200/transaction/_search?pretty'

- docker run -it -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node"  -e 'ES_JAVA_OPTS: -Xms1g -Xmx1g' docker.elastic.co/elasticsearch/elasticsearch:7.10.2  
  
## Start Kafka Connect

- docker run -it -v $PWD/config/kconnect/connect-standalone.properties:/etc/kafka/connect-standalone.properties -v $PWD/config/kconnect/elasticsearch-connect.properties:/etc/kafka/elasticsearch-connect.properties  rajat965ng/cp-kafka-connect
- connect-standalone /etc/kafka/connect-standalone.properties /etc/kafka/elasticsearch-connect.properties  


- docker run -d -v $PWD/connect-standalone.properties:/etc/kafka/connect-standalone.properties -v $PWD/elasticsearch-connect.properties:/etc/kafka/elasticsearch-connect.properties  rajat965ng/cp-kafka-connect connect-standalone /etc/kafka/connect-standalone.properties /etc/kafka/elasticsearch-connect.properties