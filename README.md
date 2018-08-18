# Event-Tracking-API
An instrumentation of an API server to catch and analyze event types

Included in this repository is the following:

- The .yml file used to spin up the docker cluster
- Various python files used in this project, primarily to generate events
- A Jupyter Notebook where the code that read parquet hdfs files into spark sql
- This README file that provides the primary report and corresponding code

## Project Overview

In this project, I set up a web server to run an API service using flask. The API service wrote API calls to a kafka topic called "events". I used curl to make API calls to the web service. The first was for the default response, and I purchased a sword with the second call. Then, I manually consumed the kafka topic "events". This way, I was able to verify the web API service is working as intended. Also, instead of publishing text, I used pyspark to format the API calls into json objects. I then printed the events to ensure my code was working properly. In addition to this, I also wrote to parquet, showed events, replaced parquet files, and separated the events. 

In addition to this, to build on using curl to test the web API, I instead used the Apache Bench utility and different pyspark code to handle multiple schema. I also used Jupyter Notebook instead of the command like to write pyspark code and read the parquet files. I registered these files as temporary tables and executed spark SQL against them.  

## Annotated Steps and Output
### Update docker images

After logging into my Digital Ocean droplet, I ran these commands to make sure I was working with the most up to date docker images. 

```
docker pull confluentinc/cp-zookeeper:latest
docker pull confluentinc/cp-kafka:latest
docker pull midsw205/cdh-minimal:latest
docker pull midsw205/spark-python:0.0.5
docker pull midsw205/base:0.1.9
```
All images were up to date. 

### Navigate to the full-stack file and check contents

I created and navigated to the full-stack directory, then checked its contents. Knowing I would need to use both flask and spark, I had three windows open: one for docker, one for flask, and one for spark. Moving forward, this annotation is in chronological order, with mentions of when I switched windows. 

```
mkdir /w205/full-stack
cd ~/w205/full-stack
ls
``` 

### Create/examine the docker-compose.yml file

Below is an annotated version of the .yml file. 

```
vi docker-compose.yml
```
```
---
version: '2'
services:
  zookeeper:    # Zookeeper is the service managing the docker cluster
    image: confluentinc/cp-zookeeper:latest    # Zookeeper image most recently downloaded
    environment:
      ZOOKEEPER_CLIENT_PORT: 32181
      ZOOKEEPER_TICK_TIME: 2000
    expose:
      - "2181"
      - "2888"
      - "32181"
      - "3888"
    extra_hosts:
      - "moby:127.0.0.1"    # Connection to the external world
      
kafka:
    image: confluentinc/cp-kafka:latest    # Kafka image most recently downloaded
    depends_on:
      - zookeeper    # Zookeeper is the service managing the docker cluster
    environment:    # Configuring the broker ID explicitly, connecting to zookeeper, and also setting the listener configuration explicitly
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:32181    # Connects to the zookeeper port
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    expose:
      - "9092"
      - "29092"    # Internal connection
    extra_hosts:
      - "moby:127.0.0.1"    # Connection to the external world

cloudera:
    image: midsw205/cdh-minimal:latest    # Hadoop image most recently downloaded
    expose:
      - "8020" # nn
      - "50070" # nn http
      - "8888" # hue
    #ports:
    #- "8888:8888"
    extra_hosts:
      - "moby:127.0.0.1"    # Connection to the external world

spark:
    image: midsw205/spark-python:0.0.5    # Pyspark image most recently downloaded
    stdin_open: true
    tty: true
    volumes:
      - /home/science/w205:/w205    # File where this is run
    expose:
      - "8888"
    ports:
      - "8888:8888" 
    depends_on:
      - cloudera    # Hadoop cluster 
    environment:
      HADOOP_NAMENODE: cloudera
    extra_hosts:
      - "moby:127.0.0.1"    # Connection to the external world
    command: bash

mids:
    image: midsw205/base:latest    # MIDSw205 image most recently downloaded. I believe this is the container that is "off the shelf"
    stdin_open: true
    tty: true  
    volumes:
      - /home/science/w205:/w205    # File where this is run    
    expose:
      - "5000" 
    ports:
      - "5000:5000"    # Port assigned to the mids container 
    extra_hosts:
      - "moby:127.0.0.1"    # Connection to the external world

```

### Spin up the docker cluster and create a kafka topic

#### Code:
```
docker-compose up -d
```
#### Output
```
Creating network "fullstack_default" with the default driver
Creating fullstack_zookeeper_1
Creating fullstack_cloudera_1
Creating fullstack_mids_1
Creating fullstack_kafka_1
Creating fullstack_spark_1

```
#### Code:
```
docker-compose exec kafka \
   kafka-topics \
     --create \
     --topic events \
     --partitions 1 \
     --replication-factor 1 \
     --if-not-exists \
     --zookeeper zookeeper:32181
```
#### Output
```
Created topic "events".
```

### Create and examine the python flask module

```
vi game_api.py
```
```python
#!/usr/bin/env python    # This code tells the shell script to run this as a python file
import json    # Import the json module to format calls to json
from kafka import KafkaProducer    # Import KafkaProducer to publish json files to kafka
from flask import Flask    # Import the flask module to run Flask

app = Flask(__name__)    # Creates a Flask instance called "app"
producer = KafkaProducer(bootstrap_servers='kafka:29092')    # Creates a KafkaProducer called producer that publishes to the kafka server 29092 (as described in the .yml file)

def log_to_kafka(topic, event):    
    event.update(request.headers)    # Update the event and log headers
    producer.send(topic, json.dumps(event).encode())    # Defines a function there the producer logs the calls to kafka and formats them in json

@app.route("/")    # Decorator for the default call
def default_response():    # Default response method
    default_event = {'event_type': 'default'}    # Dictionary format for the call
    log_to_kafka('events', default_event)    # Logs the call to kafka in json
    return "\nThis is the default response!\n"    # When this is called, returns "This is the default response!". This is helpful to ensure the program is working

@app.route("/purchase_a_sword")    # Decorator for the Flask instance "app" route to purchase a sword. 
def purchase_sword():    # Purchase sword method
    # business logic to purchase sword
    purchase_sword_event = {'event_type': 'purchase_sword'}    # Dictionary format for the call
    log_to_kafka('events', purchase_sword_event)    # Logs the call to kafka in json
    return "\nSword Purchased!\n"    # When this is called, returns "Sword Purchased!". 
        
``` 

### Initialize web API using Flask

Now that the docker clusters are up and running, with the kafka topic created, the next step was to run the following python script in the mids container. This script initialized Flask using game_api.py, which prompted me to switch windows once this was up and running. 

#### Code
```
docker-compose exec mids \
  env FLASK_APP=/w205/full-stack/game_api.py \
  flask run --host 0.0.0.0
```
#### Output
```
 * Serving Flask app "game_api"
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
```
### Run kafcacat in continuous mode

Next, I ran kafkacat in a separate window using continuous mode. That way, I could see events as they came through. I did this by taking the -e off the end of the code

#### Code
```
docker-compose exec mids \
  kafkacat -C -b kafka:29092 -t events -o beginning
```
#### Output
```
The output was blank because I hadn't called any events. 
```
### Use Apache Bench to Generate Multiple Requests

For the next step, I used Apache Bench to stress test the API. I generated multiple requests of the same call, using spoofed addresses. Specifically, I used the -n option to generate 10 of each API call. Below is sample output from the kafkacat, flask, and command line windows. I only showed output from the first command, for the sake of brevity. 

#### Code
```
docker-compose exec mids \
  ab \
    -n 10 \
    -H "Host: user1.comcast.com" \
    http://localhost:5000/
    
docker-compose exec mids \
  ab \
    -n 10 \
    -H "Host: user1.comcast.com" \
    http://localhost:5000/purchase_a_sword

docker-compose exec mids \
  ab \
    -n 10 \
    -H "Host: user2.att.com" \
    http://localhost:5000/

docker-compose exec mids \
  ab \
    -n 10 \
    -H "Host: user2.att.com" \
    http://localhost:5000/purchase_a_sword
```
#### Output-kafkacat
```
{"Host": "user1.comcast.com", "event_type": "default", "Accept": "*/*", "User-Agent": "ApacheBench/2.3"}
{"Host": "user1.comcast.com", "event_type": "default", "Accept": "*/*", "User-Agent": "ApacheBench/2.3"}
{"Host": "user1.comcast.com", "event_type": "default", "Accept": "*/*", "User-Agent": "ApacheBench/2.3"}
{"Host": "user1.comcast.com", "event_type": "default", "Accept": "*/*", "User-Agent": "ApacheBench/2.3"}
{"Host": "user1.comcast.com", "event_type": "default", "Accept": "*/*", "User-Agent": "ApacheBench/2.3"}
{"Host": "user1.comcast.com", "event_type": "default", "Accept": "*/*", "User-Agent": "ApacheBench/2.3"}
{"Host": "user1.comcast.com", "event_type": "default", "Accept": "*/*", "User-Agent": "ApacheBench/2.3"}
{"Host": "user1.comcast.com", "event_type": "default", "Accept": "*/*", "User-Agent": "ApacheBench/2.3"}
{"Host": "user1.comcast.com", "event_type": "default", "Accept": "*/*", "User-Agent": "ApacheBench/2.3"}
{"Host": "user1.comcast.com", "event_type": "default", "Accept": "*/*", "User-Agent": "ApacheBench/2.3"}
```
#### Output-flask
```
127.0.0.1 - - [27/Jul/2018 20:58:05] "GET / HTTP/1.0" 200 -
127.0.0.1 - - [27/Jul/2018 20:58:05] "GET / HTTP/1.0" 200 -
127.0.0.1 - - [27/Jul/2018 20:58:05] "GET / HTTP/1.0" 200 -
127.0.0.1 - - [27/Jul/2018 20:58:05] "GET / HTTP/1.0" 200 -
127.0.0.1 - - [27/Jul/2018 20:58:05] "GET / HTTP/1.0" 200 -
127.0.0.1 - - [27/Jul/2018 20:58:05] "GET / HTTP/1.0" 200 -
127.0.0.1 - - [27/Jul/2018 20:58:05] "GET / HTTP/1.0" 200 -
127.0.0.1 - - [27/Jul/2018 20:58:05] "GET / HTTP/1.0" 200 -
127.0.0.1 - - [27/Jul/2018 20:58:05] "GET / HTTP/1.0" 200 -
127.0.0.1 - - [27/Jul/2018 20:58:05] "GET / HTTP/1.0" 200 -
```
#### Output-command line
```
This is ApacheBench, Version 2.3 <$Revision: 1706008 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient).....done


Server Software:        Werkzeug/0.14.1
Server Hostname:        localhost
Server Port:            5000

Document Path:          /
Document Length:        30 bytes

Concurrency Level:      1
Time taken for tests:   0.035 seconds
Complete requests:      10
Failed requests:        0
Total transferred:      1850 bytes
HTML transferred:       300 bytes
Requests per second:    288.03 [#/sec] (mean)
Time per request:       3.472 [ms] (mean)
Time per request:       3.472 [ms] (mean, across all concurrent requests)
Transfer rate:          52.04 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.1      0       0
Processing:     1    3   2.8      3      11
Waiting:        0    2   3.2      1      11
Total:          1    3   2.8      3      11

Percentage of the requests served within a certain time (ms)
  50%      3
  66%      3
  75%      4
  80%      4
  90%     11
  95%     11
  98%     11
  99%     11
 100%     11 (longest request)

```
### Create, review, and run just_filtering using spark-submit

Then, I created, review, and ran just_filtering.py, which is the altered version of separate_events.py. In this case, the code was altered to handle multiple schemas for events.

```python
#!/usr/bin/env python
"""Extract events from kafka and write them to hdfs
"""
import json
from pyspark.sql import SparkSession, Row
from pyspark.sql.functions import udf


@udf('boolean')
def is_purchase(event_as_json):
    event = json.loads(event_as_json)
    if event['event_type'] == 'purchase_sword':
        return True
    return False


def main():
    """main
    """
    spark = SparkSession \
        .builder \
        .appName("ExtractEventsJob") \
        .getOrCreate()

    raw_events = spark \
        .read \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "kafka:29092") \
        .option("subscribe", "events") \
        .option("startingOffsets", "earliest") \
        .option("endingOffsets", "latest") \
        .load()

    purchase_events = raw_events \
        .select(raw_events.value.cast('string').alias('raw'),
                raw_events.timestamp.cast('string')) \
        .filter(is_purchase('raw'))

    extracted_purchase_events = purchase_events \
        .rdd \
        .map(lambda r: Row(timestamp=r.timestamp, **json.loads(r.raw))) \
        .toDF()
    extracted_purchase_events.printSchema()
    extracted_purchase_events.show()


if __name__ == "__main__":
    main()
    
```
#### Code
```
docker-compose exec spark \
  spark-submit /w205/full-stack/just_filtering.py
```
#### Output
```
While the output was too long to print in its entirety, here is some sample output, specifically, shown events:
+------+-----------------+---------------+--------------+--------------------+
|Accept|             Host|     User-Agent|    event_type|           timestamp|
+------+-----------------+---------------+--------------+--------------------+
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-07-27 21:02:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-07-27 21:02:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-07-27 21:02:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-07-27 21:02:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-07-27 21:02:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-07-27 21:02:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-07-27 21:02:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-07-27 21:02:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-07-27 21:02:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-07-27 21:02:...|
|   */*|    user2.att.com|ApacheBench/2.3|purchase_sword|2018-07-27 21:03:...|
|   */*|    user2.att.com|ApacheBench/2.3|purchase_sword|2018-07-27 21:03:...|
|   */*|    user2.att.com|ApacheBench/2.3|purchase_sword|2018-07-27 21:03:...|
|   */*|    user2.att.com|ApacheBench/2.3|purchase_sword|2018-07-27 21:03:...|
|   */*|    user2.att.com|ApacheBench/2.3|purchase_sword|2018-07-27 21:03:...|
|   */*|    user2.att.com|ApacheBench/2.3|purchase_sword|2018-07-27 21:03:...|
|   */*|    user2.att.com|ApacheBench/2.3|purchase_sword|2018-07-27 21:03:...|
|   */*|    user2.att.com|ApacheBench/2.3|purchase_sword|2018-07-27 21:03:...|
|   */*|    user2.att.com|ApacheBench/2.3|purchase_sword|2018-07-27 21:03:...|
|   */*|    user2.att.com|ApacheBench/2.3|purchase_sword|2018-07-27 21:03:...|
+------+-----------------+---------------+--------------+--------------------+

```

### Stop flask, add purchase_a_knife, restart flask, and modify spark accordingly

Next, I used ^C to stop the flask API server, and added the following code to the game_api.py file: 

```python
@app.route("/purchase_a_knife")
def purchase_a_knife():
    purchase_knife_event = {'event_type': 'purchase_knife',
                            'description': 'very sharp knife'}
    log_to_kafka('events', purchase_knife_event)
    return "Knife Purchased!\n"
```
Then, I used the following modified spark code to write the events out to hadoop hdfs files in parquet format, using massively parallel processing. I also used the overwrite option, which will overwrite any existing directories of the same name. That way, it can be read back in quickly if it's a large data set. Below is the python file filtered_writes.py:

```python
#!/usr/bin/env python
"""Extract events from kafka and write them to hdfs
"""
import json
from pyspark.sql import SparkSession, Row
from pyspark.sql.functions import udf


@udf('boolean')
def is_purchase(event_as_json):
    event = json.loads(event_as_json)
    if event['event_type'] == 'purchase_sword':
        return True
    return False


def main():
    """main
    """
    spark = SparkSession \
        .builder \
        .appName("ExtractEventsJob") \
        .getOrCreate()

    raw_events = spark \
        .read \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "kafka:29092") \
        .option("subscribe", "events") \
        .option("startingOffsets", "earliest") \
        .option("endingOffsets", "latest") \
        .load()

    purchase_events = raw_events \
        .select(raw_events.value.cast('string').alias('raw'),
                raw_events.timestamp.cast('string')) \
        .filter(is_purchase('raw'))

    extracted_purchase_events = purchase_events \
        .rdd \
        .map(lambda r: Row(timestamp=r.timestamp, **json.loads(r.raw))) \
        .toDF()
    extracted_purchase_events.printSchema()
    extracted_purchase_events.show()

    extracted_purchase_events \
        .write \
        .mode('overwrite') \
        .parquet('/tmp/purchases')


if __name__ == "__main__":
    main()
```

### Restart flask, and use spark-submit to submit the filtered_writes.py file to spark

#### Code
```
docker-compose exec mids \
  env FLASK_APP=/w205/full-stack/game_api.py \
  flask run --host 0.0.0.0
```
#### Output 
```
 * Serving Flask app "game_api"
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit) 
```
#### Code
```
docker-compose exec spark \
  spark-submit 
	/w205/full-stack/filtered_writes.py
```
#### Output 
```
# The output was too long to reproduce here, but the code ran successfully. 
```

### To check to make sure the code ran successfully, I checked to verify that the files that should have been written were written to the hadoop hdfs file system.

#### Code
```
docker-compose exec cloudera hadoop fs -ls /tmp/
```
#### Output
```
Found 3 items
drwxrwxrwt   - mapred mapred              0 2018-02-06 18:27 /tmp/hadoop-yarn
drwx-wx-wx   - root   supergroup          0 2018-07-27 20:55 /tmp/hive
drwxr-xr-x   - root   supergroup          0 2018-07-27 21:14 /tmp/purchases
```

#### Code
```
docker-compose exec cloudera hadoop fs -ls /tmp/purchases/
```
#### Output
```
Found 2 items
-rw-r--r--   1 root supergroup          0 2018-07-27 21:14 /tmp/purchases/_SUCCESS
-rw-r--r--   1 root supergroup       1640 2018-07-27 21:14 /tmp/purchases/part-00000-91237e09-bed9-45bd-ae63-9f637b022547-c000.snappy.parquet
```

### Start the Jupyter Notebook 

#### Code
```
docker-compose exec spark \
  env \
    PYSPARK_DRIVER_PYTHON=jupyter \
    PYSPARK_DRIVER_PYTHON_OPTS='notebook --no-browser --port 8888 --ip 138.68.2.190 --allow-root' \
  pyspark
```
#### Output
```
Copy/paste this URL into your browser when you connect for the first time, to login with a token: http://0.0.0.0:8888/?token=e22d32c4a69e7eae06d99960a693ade445900e8ac451e40e

```
For the sake of brevity, I didn't include all output. However, I did include the token as shown above. Then, I copy/pasted the following token http://138.68.2.190:8888/?token=e22d32c4a69e7eae06d99960a693ade445900e8ac451e40e (inserting my droplet URL)

### Save Jupyter Notebook

To save space here, please refer to the commented Jupyter Notebook included in this repository. All code and output is there, and the code is thoroughly commented. 

To be able to save the Jupyter Notebook to the correct file, and have it survive the docker cluster, I created a symbolic link to my Jupyter Notebook.

#### Code
```
docker-compose exec spark bash
```
#### Output
```
root@371b12de0b62:/spark-2.2.0-bin-hadoop2.6# 
```
#### Code
```
df -k
```
#### Output
```
Filesystem     1K-blocks     Used Available Use% Mounted on
overlay         81120924 21760568  59343972  27% /
tmpfs              65536        0     65536   0% /dev
tmpfs            2023208        0   2023208   0% /sys/fs/cgroup
/dev/vda1       81120924 21760568  59343972  27% /w205
shm                65536        0     65536   0% /dev/shm
tmpfs            2023208        0   2023208   0% /proc/scsi
tmpfs            2023208        0   2023208   0% /sys/firmware 
```
#### Code
```
ln -s /w205 w205
ls -l
exit
```
#### Output
```
total 116
-rw-r--r-- 1  500  500 17881 Jun 30  2017 LICENSE
-rw-r--r-- 1  500  500 24645 Jun 30  2017 NOTICE
drwxr-xr-x 3  500  500  4096 Jun 30  2017 R
-rw-r--r-- 1  500  500  3809 Jun 30  2017 README.md
-rw-r--r-- 1  500  500   128 Jun 30  2017 RELEASE
-rw-r--r-- 1 root root  1319 Jul 27 21:38 Untitled.ipynb
drwxr-xr-x 2  500  500  4096 Jun 30  2017 bin
drwxr-xr-x 1  500  500  4096 Jul 27 20:54 conf
drwxr-xr-x 5  500  500  4096 Jun 30  2017 data
-rw-r--r-- 1 root root   697 Jul 27 21:19 derby.log
lrwxrwxrwx 1 root root    34 Feb 18 22:18 entrypoint.sh -> usr/local/bin/docker-entrypoint.sh
drwxr-xr-x 4  500  500  4096 Jun 30  2017 examples
drwxr-xr-x 1  500  500  4096 Feb 18 22:18 jars
drwxr-xr-x 2  500  500  4096 Jun 30  2017 licenses
drwxr-xr-x 5 root root  4096 Jul 27 21:19 metastore_db
drwxr-xr-x 1  500  500  4096 Jun 30  2017 python
drwxr-xr-x 2  500  500  4096 Jun 30  2017 sbin
drwxr-xr-x 2 root root  4096 Jul 27 21:06 spark-warehouse
drwxr-xr-x 2 root root  4096 Feb 18 22:18 templates
lrwxrwxrwx 1 root root     5 Jul 27 21:44 w205 -> /w205
drwxr-xr-x 2  500  500  4096 Jun 30  2017 yarn
```

### Exit flask, tear down docker container, check to make sure all docker containers are down, and exit

After using control-C to exit flask, kafkacat, and Jupyter Notebook, I tore down the docker containers and made sure everything was down before exiting. 

#### Code
```
docker-compose down
```
#### Output
```
Stopping fullstack_kafka_1 ... done
Stopping fullstack_spark_1 ... done
Stopping fullstack_zookeeper_1 ... done
Stopping fullstack_cloudera_1 ... done
Stopping fullstack_mids_1 ... done
Removing fullstack_kafka_1 ... done
Removing fullstack_spark_1 ... done
Removing fullstack_zookeeper_1 ... done
Removing fullstack_cloudera_1 ... done
Removing fullstack_mids_1 ... done
Removing network fullstack_default
```
#### Code
```
docker-compose ps
```
#### Output
```
Name   Command   State   Ports 
------------------------------
```
#### Code
```
docker ps -a
exit
```
#### Output
```
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

