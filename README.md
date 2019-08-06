# Kafka, Java, GO, Javascript, Postgres

A dockerized solution for logs management. 

It generates sample logs like below:
```
 London   | 2019-08-06 20:48:36.482 | INFO  | Brain Beatty-6535278545336357
 Beijing  | 2019-08-06 20:48:37.482 | FATAL | Kacey Metz-2221001727843784
 Istanbul | 2019-08-06 20:48:38.482 | WARN  | Citlalli Mayer-375830681084863
 Istanbul | 2019-08-06 20:48:39.482 | WARN  | Derek Blick-4383860516614541
 Istanbul | 2019-08-06 20:48:40.482 | INFO  | Eldora Bauch-348738062311245
 Beijing  | 2019-08-06 20:48:41.482 | ERROR | Clement Dickens-4174461203278603
 Istanbul | 2019-08-06 20:48:42.482 | WARN  | Keshawn Medhurst-6532607805583183
 Moscow   | 2019-08-06 20:48:43.482 | FATAL | Desmond Hyatt-376626163520652
 Istanbul | 2019-08-06 20:48:44.482 | FATAL | Elinore Breitenberg-6512216753126578
```
Then it loads the logs to Kafka and Postgres. 
Soon we will present results on a browser as realtime events.

It utilizes ideas and codes from:

<https://docs.confluent.io/current/quickstart/ce-docker-quickstart.html#ce-docker-quickstart> ,
<https://medium.com/@stephane.maarek/how-to-use-apache-kafka-to-transform-a-batch-pipeline-into-a-real-time-one-831b48a6ad85> ,
<https://github.com/mingrammer/flog> ,

# Prerequisites and Building 

All the instructions are in [run.sh](run.sh)

```bash
sudo nano /etc/hosts 
#append the line below:
127.0.1.1       broker scahema-registry zookeeper rest-proxy connect ksql-server ksql-cli ksql-datagen control-center postgres
#save and exit

#install Prerequisites - if not already installed
sudo apt-get install openjdk-8-jdk maven git curl

#if docker and docker-compose is not installed install it as below:
curl -sSL https://get.docker.com/ | sh
sudo usermod -aG docker {your-account-username}
sudo curl -L https://github.com/docker/compose/releases/download/1.24.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

#clone repository
mkdir ~/bigdata && cd ~/bigdata
git clone https://github.com/mehmetaydar/kafka-docker-logs.git teb-docker

echo "Creating log folder:"
cd ~/bigdata/teb-docker
mkdir logs

echo "Building java source code .."
cd ~/bigdata/teb-docker/java-producer-consumer
mvn clean package
```

# Start dockers
```bash
cd ~/bigdata/teb-docker/java-producer-consumer
#this will also generate logs in ~/bigdata/teb-docker/logs
docker-compose up -d --build

#check logs
echo "Sample logs generated in: "
ls -lth ~/bigdata/teb-docker/logs

echo "10mb size of sample logs generated and splitted on every 2mb. To generate 20mb size of logs run: "
docker exec mylog flog -t log -o /logs/gen.log -b 20485760 -s 1 -p 2097152 -w
ls -lth ~/bigdata/teb-docker/logs
```

# Build kafka topics
```bash
echo "Building kafka topics: "
docker exec broker kafka-topics --create --topic teb-logs --partitions 1 --replication-factor 1 --zookeeper zookeeper:2181
docker exec broker kafka-topics --create --topic teb-logs-valid --partitions 1 --replication-factor 1 --zookeeper zookeeper:2181
docker exec broker kafka-topics --create --topic teb-logs-fraud --partitions 1 --replication-factor 1 --zookeeper zookeeper:2181

echo "list kafka topics: "
docker exec broker kafka-topics --describe --zookeeper zookeeper:2181
```

# Start kafka producers and consumers
```bash
echo "on another terminal please start console consumers for main teb-logs: "
docker exec schema-registry kafka-avro-console-consumer --topic teb-logs --bootstrap-server broker:9092

echo "on another terminal please start console consumers for main teb-logs-valid: "
docker exec schema-registry kafka-avro-console-consumer --topic teb-logs-valid --bootstrap-server broker:9092

echo "on another terminal please start console consumers for main teb-logs-fraud: "
docker exec schema-registry kafka-avro-console-consumer --topic teb-logs-fraud --bootstrap-server broker:9092

echo "And launch our first producer for sample logs in another terminal ! "
java -jar ~/bigdata/teb-docker/java-producer-consumer/udemy-reviews-producer/target/uber-udemy-reviews-producer-1.0-SNAPSHOT.jar

echo "And launch producer for valid/fraud logs in another terminal ! "
java -jar ~/bigdata/teb-docker/java-producer-consumer/udemy-reviews-fraud/target/uber-udemy-reviews-fraud-1.0-SNAPSHOT.jar

echo "Load sample logs from kafka to postgre database: "
curl -X POST http://connect:8083/connectors -H "Content-Type: application/json" -d '{
        "name": "SinkTopics2",
        "config": {
			"connector.class" : "io.confluent.connect.jdbc.JdbcSinkConnector",
			"tasks.max" : 3,
			"connection.url" : "jdbc:postgresql://postgres:5432/postgres",
			"connection.user" : "postgres",
			"connection.password" : "postgres",
			"insert.mode" : "upsert",
			"pk.mode" : "record_value",
			"pk.fields" : "created",
			"auto.create" : true,
			"topics" : "teb-logs",
			"key.converter" : "org.apache.kafka.connect.storage.StringConverter"
                }
        }'

echo "In another terminal - Run sqls on postgres to check the loaded logs - to exit run \q: "
docker exec -it postgres psql -U postgres
select * from "teb-logs";\q
```

# Stopping
```bash
echo "Stopping already running applications"
cd ~/bigdata/teb-docker
docker container stop $(docker container ls -a -q -f "label=io.confluent.docker")
docker stop postgres
docker stop mylog
echo "y" | docker container prune
```

# Todo
We will show a real time graphic of the aggregated log results.
