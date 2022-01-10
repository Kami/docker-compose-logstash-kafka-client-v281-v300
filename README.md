# Logstash 7.16 with JDK 17 and Kafka Input with Kafka Client Library 2.8.x / 3.0.x

This repository contains docker compose setup which demonstrates how to run Logstash under JDK 17
using Logstash kafka input plugin with a more recent Kafka client library Java dependency (2.8.x).

Current version of Logstash bundles very old version of the Kafka client library (2.1.0) which
contains known bugs in the consensus protocol (https://issues.apache.org/jira/browse/KAFKA-12890,
https://issues.apache.org/jira/browse/KAFKA-12983, https://issues.apache.org/jira/browse/KAFKA-12984,
https://www.mail-archive.com/users@kafka.apache.org/msg39617.html, etc.( Thos bugs have only been
fixed in Kafka 2.8.1 and >= 3.0.0.

## Docker Compose Setup

Docker compose setup is loosely based on https://github.com/sermilrod/kafka-elk-docker-compose (MIT
license).

This setup starts the following services:

* Apache ZooKeeper
* Apache Kafka
* Filebeat
* Apache httpd

Apache httpd is used to produce access logs which are written to a file on disk. Those files on
disk are ingested by filebeat into Kafka and Logstash has Kafka input configured which consumes
those records and prints them to stdout.

It includes the following changes to the original setup:

* For simplicity and ease of troubleshooting we only run a single instance of ZooKeeper and Kafka
  broker.
* Kibana and Elasticsearch are not used anymore, Logstash is configured to only output events to
  stdout.
* ZooKeeper, Kafka and Filebeats have been updated to utilize more recent versions. To test
  compatibility of the new Kafka Client Library with older Kafka broker version, we use Kafka 
  Broker 2.6.x.
* Various config options have been updated.

## Needed Changes

To get everything working, various changes needed to be made in multiple places.

1. logstash-input-kafka - Various changes were made to the input plugin so it bundles new Kafka
   client library and also builds under newer versions of JDK (JDK >= 14) and Gradle >= 7.3.
2. Logstash Dockerfile - We build a custom version of Logstash which uses JDK 17 and install our
   updated Lafka input plugin. To get everything to work under JDK 17, we also need to use patched
   ruby-maven version and copy Kafka Client jars which is bundled with the plugin into the global
   Logstash search path so it's loaded by Logstash (for some reason, Logstash doesn't load jar file
   which is local / vendored with the gem).

## Usage

Change ``KAFKA_ADVERTISED_HOST_NAME`` value in ``docker-compose.yml`` to your local host IP and then
run:

```bash
docker-compose up -d
```

Initial startup may take a while, after that, subsequent start up should take 30-40 seconds or so.

If you want to destroy all the state (e.g. offsets and other data stored in ZooKeeper, you can do
that by running ``docker-compose down -v``).

You should also do that if there is some left over state from previous runs and you see messages
like this in Kafka logs and events are not being ingested:

```
kafka_1  | [2022-01-10 12:18:49,114] ERROR Error while creating ephemeral at /brokers/ids/1, node already exists and owner '72058176868384769' does not match current session '72058190436433921' (kafka.zk.KafkaZkClient$CheckedEphemeral)
```

Then generate some requests to Apache so some logs are generated and ingested by filebeat into
Kafka.

```bash
for((i=1;i<=10;i+=1)); do curl http://localhost:8888/; done
```

After a while, those logs should be ingested by filebeat into Kafka and you should see Logstash
output ingested events to stdout (``docker compose logs -f logstash``).

## Backward Incompatible Kafka Input Plugin Config Changes

There were some backward incompatible changes in terms of consumer config options which need to
be handled when configuring logstash kafka input:

* ``max_poll_records`` - This value now needs to be a string, otherwise it will incorrectly be cast
  to long which will fail.
* ``decorate_events`` - This value now needs to be a beolean (e.g. ``true``).
* ``partition_assignment_strategy`` - This value now needs to contain a full clase name (e.g.
  ``org.apache.kafka.clients.consumer.CooperativeStickyAssignor``)

Example old config:

```
input {
  kafka {
    bootstrap_servers => "kafka:9092"
    client_id => "logstash"
    group_id => "logstash"
    consumer_threads => 3
    topics => ["log"]
    codec => "json"
    tags => ["log", "kafka_source"]
    type => "log"
    add_field => { "[tag]" => "kafka" }
    heartbeat_interval_ms => "1000"
    poll_timeout_ms => 10000
    session_timeout_ms => "120000"
    request_timeout_ms => "130000"
    max_poll_records => 500
    decorate_events => "basic"
    partition_assignment_strategy => "cooperative_sticky"
  }
}

```

Example new config:

```
input {
  kafka {
    bootstrap_servers => "kafka:9092"
    client_id => "logstash"
    group_id => "logstash"
    consumer_threads => 3
    topics => ["log"]
    codec => "json"
    tags => ["log", "kafka_source"]
    type => "log"
    add_field => { "[tag]" => "kafka" }
    heartbeat_interval_ms => "1000"
    poll_timeout_ms => 10000
    session_timeout_ms => "120000"
    request_timeout_ms => "130000"
    max_poll_records => "500"
    decorate_events => "true"
    partition_assignment_strategy => "org.apache.kafka.clients.consumer.CooperativeStickyAssignor"
  }
}
```
