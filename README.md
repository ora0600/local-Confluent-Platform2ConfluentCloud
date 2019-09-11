# Configure local Confluent Platform /Apache Kafka components to use Confluent Cloud Cluster
You have a local Apache Kafka or Confluent Platform setup and want to use the Confluent Cloud Cluster for some components.
This Example shows to setup KSQL and REST Proxy with Confluent Cloud on your local machine or e.g. an AWS ec2 compute instance.
If you do not have a local Apache/Confluent Platform running, you can provision very easy a setup in your cloud provider account. Please see a terraform script for deploy Confluent Platform 5.3 in AWS - [Deploy Confluent Platform 5.3 in AWS EC2](https://github.com/ora0600/cpe53-singlenodeonaws).

HINT:
If you run the mentioned terraform to create your Confluent Platform in aws EC2, than please stop all services before continue:
```BASH
# stop Confluent Platfrom
sudo systemctl stop kafka-rest
sudo systemctl stop ksql-server
sudo systemctl stop kafka-connect
sudo systemctl stop control-center
sudo systemctl stop kafka
sudo systemctl stop zookeeper
```
## Create API Key for the Cluster and Schema Registry in Confluent Cloud
The Confluent Cloud should be runninng. If not follow these links:
First all you have to register into Confluent Cloud -
  * [Set up the cluster] (https://docs.confluent.io/current/quickstart/cloud-quickstart/index.html)
  * [create in confluent cloud API key for the cluster see] (https://confluent.cloud/environments/t6463/clusters/lkc-4vo2n/settings/keys)
  * [create in confluent cloud Schema Registry API keys see] (https://confluent.cloud/environments/t6463/schema-registry/keys)

##  Install confluent cloud cli
If the confluent cloud setup is running you can start to install and configure the cloud cli, follow [cli installation] (https://docs.confluent.io/current/cloud/cli/install.html)
Do some more tests with cli and confluent cloud, so that you are pretty sure that confluent cloud is running well.
```BASH
# install cloud cli
sudo -s 
curl -L https://cnfl.io/ccloud-cli | sh -s -- -b /usr/local/bin
exit
# login : Logins come from Perry
ccloud login --url https://confluent.cloud

# list environments
ccloud environment list

# choose the right environment id
ccloud environment use <env-id>

# list clusters
ccloud kafka cluster list

# use the right cluster id, named by Perry
ccloud kafka cluster use <cluster-id e.g. lkc-emmox>

# API Keys and secrect have to created first in Confluent cloud GUI, store and it with ccloud
ccloud api-key store <api-key> <api-secret> --cluster <cluster-id>

# use API key
ccloud api-key use <api-key>

# follow this document: consume, produce: https://docs.confluent.io/current/quickstart/cloud-quickstart/index.html

# create a topic
ccloud kafka topic create myusers
# produce to topic, enter values, close with ctrl+c
ccloud kafka topic produce myusers
# Enter e.g. > {"records":[{"value":{"name": "testUser"}}]}
# consume from beginning from topic, close with ctrl+c
ccloud kafka topic consume -b myusers
```
## Prepare your local installation to connect Confluent Cloud
To work with cloud we need some more propertery parameters than to work local. To make it easier Confluent has offered a script to generate all the properties file. [Please go to Github ccloud](https://github.com/confluentinc/examples/tree/5.3.0-post/ccloud)

I did copy the script into Confluent Cloud Platform AWS setup, so that you can execute it, if you want ([CP52 in AWS](https://github.com/ora0600/cpe53-singlenodeonaws)).
What you have to do?
  * Setup Confluent Cloud CLI (ccloud)
  * create a config file in ~/.ccloud/
This config file should look like this:
```
cat ~/.ccloud/config
bootstrap.servers=<confluent cloud server>:9092
ssl.endpoint.identification.algorithm=https
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="<API Key>" password="<API Secret>";
basic.auth.credentials.source=USER_INFO
schema.registry.basic.auth.user.info=<SCHEMA API Key>:<Schema API Secret>
schema.registry.url=https://<Schema Registry Host>
```
Then you have to generate the files. 
```
./ccloud-generate-cp-configs.sh $HOME/.ccloud/config
```
In the directory ./delta_configs/ all generated property files are stored.

## Connect Confluent Cloud with local KSQL Installation

[follow document](https://docs.confluent.io/current/cloud/connect/ksql-cloud-config.html)
For this sample we will use the local kafka/confluent installation. Just as an refresher, here is how to install confluent platform:
```BASH
# wget http://packages.confluent.io/archive/5.3/confluent-5.3.0-2.12.tar.gz
# tar -xvf confluent-5.3.0-2.12.tar.gz
# set environment 
# follow this document https://docs.confluent.io/current/quickstart/ce-quickstart.html
```
To run KSQL against Confluent Cloud, the configuration property file have to be changed. The best way is to create an own ksql-server.properties. Please replace <SCHEMA_REGISTRY_API_KEY>:<SCHEMA_REGISTRY_API_SECRET>  <CCLOUD_API_KEY> <CCLOUD_API_SECRET> <CCLOUD_BOOTSTRAP_SERVER>
```BASH
echo "# Configuration derived from template_delta_configs/example_ccloud_config
listeners=http://0.0.0.0:8088
ssl.endpoint.identification.algorithm=https
sasl.mechanism=PLAIN
request.timeout.ms=20000
retry.backoff.ms=500
security.protocol=SASL_SSL
bootstrap.servers=<CCLOUD-HOST>:9092
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username=\"<CCLOUD API KEY>\" password=\"<CCLOUD API Secret>\";
basic.auth.credentials.source=USER_INFO
schema.registry.basic.auth.user.info=<Schema API Key>:<Schema API Secret>
schema.registry.url=<SCHEMA_REG_URL>
# Confluent Monitoring Interceptor specific configuration
confluent.monitoring.interceptor.ssl.endpoint.identification.algorithm=https
confluent.monitoring.interceptor.sasl.mechanism=PLAIN
confluent.monitoring.interceptor.security.protocol=SASL_SSL
confluent.monitoring.interceptor.bootstrap.servers=<CCLOUD-HOST>:9092
confluent.monitoring.interceptor.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username=\"<CCLOUD API KEY>\" password=\"<CCLOUD API Secret>\";
# KSQL Server specific configuration
producer.interceptor.classes=io.confluent.monitoring.clients.interceptor.MonitoringProducerInterceptor
consumer.interceptor.classes=io.confluent.monitoring.clients.interceptor.MonitoringConsumerInterceptor
ksql.streams.producer.retries=2147483647
ksql.streams.producer.confluent.batch.expiry.ms=9223372036854775807
ksql.streams.producer.request.timeout.ms=300000
ksql.streams.producer.max.block.ms=9223372036854775807
ksql.streams.replication.factor=3
ksql.internal.topic.replicas=3
ksql.sink.replicas=3
# Confluent Schema Registry configuration for KSQL Server
ksql.schema.registry.basic.auth.credentials.source=USER_INFO
ksql.schema.registry.basic.auth.user.info=<SCHEMAREG-API-KEY>:<SCHEMAREQ-API-SECRET>
ksql.schema.registry.url=<SCHEMA_REG_URL>" > ccloud_ksql-server.properties
```

## Connect Clonfluent Cloud with local REST Proxy
Create your own ccloud_kafka-rest.properties and replace <SCHEMA_REGISTRY_API_KEY>:<SCHEMA_REGISTRY_API_SECRET>  <CCLOUD_API_KEY> <CCLOUD_API_SECRET> <CCLOUD_BOOTSTRAP_SERVER>
```BASH
echo "id=kafka-rest-with-ccloud
bootstrap.servers=<CCLOUD-HOST>:9092
client.sasl.mechanism=PLAIN
client.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username=\"<CCLOUD API KEY>\" password=\"<CCLOUD API Secret>\";
client.security.protocol=SASL_SSL
client.ssl.endpoint.identification.algorithm=https
# consumer only properties must be prefixed with consumer.
consumer.retry.backoff.ms=600
consumer.request.timeout.ms=25000
# producer only properties must be prefixed with producer.
producer.acks=1
# admin client only properties must be prefixed with admin.
admin.request.timeout.ms=50000
# uncomment and set correct value if using with schema registry
schema.registry.basic.auth.credentials.source=USER_INFO
schema.registry.basic.auth.user.info=<SCHEMAREG-API-KEY>:<SCHEMAREQ-API-SECRET>
schema.registry.url=<SCHEMA_REG_URL>" > ccloud_kafka-rest.properties
```
## Start your local components
First the REST Proxy and then the KSQL Server.
```BASH
# start REST Proxy local with connection to CCLOUD
kafka-rest-start -daemon ccloud_kafka-rest.properties
# start local ksql-server with connection to CCLOUD
ksql-server-start -daemon ccloud_ksql-server.properties
# or if used the generation tool
ksql-server-start -daemon delta_configs/ksql-server-ccloud.delta
control-center-start -daemon delta_configs/control-center-ccloud.delta 
connect-distributed -daemon delta_configs/connect-ccloud.delta
```
If you are running on AWS (with mentioned terraform script), then please start as followed:
```BASH
# start REST to connect Confluent clouod
sudo software/confluent-5.3.0/bin/kafka-rest-start -daemon ./ccloud_kafka-rest.properties
sudo software/confluent-5.3.0/bin/ksql-server-start -daemon ccloud_ksql-server.properties
# or if used the generation tool
sudo software/confluent-5.3.0/bin/ksql-server-start -daemon delta_configs/ksql-server-ccloud.delta
sudo software/confluent-5.3.0/bin/control-center-start -daemon delta_configs/control-center-ccloud.delta 
sudo software/confluent-5.3.0/bin/connect-distributed -daemon delta_configs/connect-ccloud.delta
```
HINT:
If you start the connect distributed, than some parameters are missing (standalone I did check):
```
echo "# missing values
group.id=connect-cluster
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=false
value.converter.schemas.enable=false
# Connect clusters create three topics to manage offsets, configs, and status
# information. Note that these contribute towards the total partition limit quota.
offset.storage.topic=connect-offsets
offset.storage.replication.factor=3
offset.storage.partitions=3
config.storage.topic=connect-configs
config.storage.replication.factor=3
status.storage.topic=connect-status
status.storage.replication.factor=3
offset.flush.interval.ms=10000
# end missing values" >> delta_configs/connect-ccloud.delta

to access the control center via SSH you have to tunnel:
```
ssh -i hackathon-temp-key.pem -N -L 9022:ip-<Priv IP>.<REGION>.compute.internal ec2-user@<Pub IP>
```
and than goto your browser and enter http://localhos:9022

## Play around with KSQL
start your local ksql interactive tool and enter some code:
```
ksql
ksql> list topics
ksql> SET 'auto.offset.reset' = 'earliest';
ksql> CREATE STREAM myusers (userdata VARCHAR)  WITH (KAFKA_TOPIC='myusers', VALUE_FORMAT='DELIMITED');
ksql> select * from myusers;
```
You can follow this document for more samples [Tumbeling window with KSQL](https://kafka-tutorials.confluent.io/create-tumbling-windows/ksql.html)
## Play around and check REST Proxy
Get a list of topics
```
curl "http://localhost:8082/topics"
```
Get info about myusers topic
```
curl "http://localhost:8082/topics/myusers"
```
create a consumer
```
curl -X POST -H "Content-Type: application/vnd.kafka.v2+json" \
      --data '{"name": "my_consumer", "format": "json", "auto.offset.reset": "earliest"}' \
      http://localhost:8082/consumers/my_consumer
```
Subribe to a topic
```
curl -X POST -H "Content-Type: application/vnd.kafka.v2+json" --data '{"topics":["myusers"]}' \
 http://localhost:8082/consumers/my_consumer/instances/my_consumer/subscription
# No content in response
```
Consume data from Topic via REST API call:
```
curl -X GET -H "Accept: application/vnd.kafka.json.v2+json" \
      http://localhost:8082/consumers/my_consumer/instances/my_consumer/records
```
Close the consumer:
```
curl -X DELETE -H "Accept: application/vnd.kafka.v2+json" \
          http://localhost:8082/consumers/my_consumer/instances/my_consumer
```
This example shows how you can use local components to run against the Confluent Cloud.
