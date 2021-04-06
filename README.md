# mm2-demo
This repository provides the configs for Mirror Maker 2 for Kafka.

### Pre-requisites
- Install Kafka Connect instance
- Install `jq` to parse and print pretty json

### Mirror Maker 2

This repository uses the [Amazon Mirror Maker 2](https://amazonmsk-labs.workshop.aws/en/migration/mirrormaker2/setupmm2.html#custpol).
 documentation as its reference.
 
There are three MirrorMaker connectors: MirrorSourceConnector, MirrorHeartbeatConnector, MirrorCheckpointConnector

In this repository we have three json configs that is meant to be run on a connect instance. 
These MM2 connectors work together to replicate data from the source cluster to the target cluster.
It also uses heartbeats and checkpoints to enable offset translation, which unlike Replicator, applies to both java and non-java clients. 

### Running Connect
You can spin up a Connect instance in multiple ways:
- [Docker Connect Deployment](https://docs.confluent.io/5.0.0/installation/docker/docs/installation/connect-avro-jdbc.html#kafka-connect-tutorial-on-docker)
- Kubernetes Connect Deployment
- [Ansible Connect Deployment for CCloud](https://github.com/confluentinc/cp-ansible/blob/6.0.1-post/sample_inventories/ccloud.yml)


### Replication Policy

There are two replication policies: 
- [CustomReplicationPolicy](https://amazonmsk-labs.workshop.aws/en/migration/mirrormaker2/setupmm2.html#custpol)
- [DefaultReplicationPolicy](https://amazonmsk-labs.workshop.aws/en/migration/mirrormaker2/setupmm2.html#defpol)

A ReplicationPolicy defines what a "remote topic" is and how to interpret it. This should generally be consistent across an organization.
The DefaultReplicationPolicy specifies the <source>.<topic> convention for its topics to be replicated.
The CustomReplicationPolicy allows you to set your own topic naming convention.

In order to to use the CustomReplicationPolicy, you have to add the class to the json payload of your mirror maker connector.
```
{
  "replication.policy.class": "com.amazonaws.kafka.samples.CustomMM2ReplicationPolicy",
}
```

## Mirror Maker Connectors

### *MirrorSourceConnector*

Once the connect instance is running, you can send an API request to the connector to start the MM2 connectors.
We start with the MirrorSourceConnector which actually does the replication of the topics between the clusters.
```

curl -X PUT -H "Content-Type: application/json" --data @mm2-src.json http://localhost:8083/connectors/mm2-src/config | jq .

```

You can verify the status of this connector by running:
```
curl -s localhost:8083/connectors/mm2-src/status | jq .
```

### *MirrorCheckpointConnector*
This periodically emit checkpoints in the destination cluster, containing offsets for each consumer group in the source cluster. 
The connector will periodically query the source cluster for all committed offsets from all consumer groups, filter for those topics being replicated, and emit a message to a topic like us-west.checkpoints.internal in the destination cluster. 
The message will include the following fields:

- consumer group id (String)
- topic (String) – includes source cluster prefix
- partition (int)
- upstream offset (int): latest committed offset in source cluster
- downstream offset (int): latest committed offset translated to target cluster
- metadata (String)
- timestamp

As with  `__consumer_offsets`, the checkpoint topic is log-compacted to reflect only the latest offsets across consumer groups, based on a topic-partition-group composite key. The topic will be created automatically by the connector if it doesn't exist.

Finally, an offset sync topic encodes cluster-to-cluster offset mappings for each topic-partition being replicated.

- topic (String): remote topic name
- partition (int)
- upstream offset (int): an offset in the source cluster
- downstream offset (int): an equivalent offset in the target cluster
- The above fields are what MirrorMaker 2 uses to enable offset translation.

Please look at the confluent page for additional information on how [checkpoint](https://cwiki.apache.org/confluence/display/KAFKA/KIP-382%3A+MirrorMaker+2.0#KIP382:MirrorMaker2.0-InternalTopics) works.

Afterwards run the MirrorCheckpointConnector by using the command.

```
curl -X PUT -H "Content-Type: application/json" --data @mm2-chk.json http://localhost:8083/connectors/mm2-chk/config | jq .
```

Again verify the status by running:
```
curl -s localhost:8083/connectors/mm2-chk/status | jq .
```

### *MirrorHeartbeatConnector*

MM2 emits a heartbeat topic in each source cluster, which is replicated to demonstrate connectivity through the connectors. 
Downstream consumers can use this topic to verify that a) the connector is running and b) the corresponding source cluster is available. 
Heartbeats will get propagated by source and sink connectors s.t. chains like backup.us-west.us-east.heartbeat are possible. 
Cycle detection prevents infinite recursion.

More information can be found [here](https://cwiki.apache.org/confluence/display/KAFKA/KIP-382%3A+MirrorMaker+2.0#KIP382:MirrorMaker2.0-InternalTopics).

To run the Heartbeat connector run the following command:

```
curl -X PUT -H "Content-Type: application/json" --data @mm2-chk.json http://localhost:8083/connectors/mm2-chk/config | jq .
```

Again verify the status by running:
```
curl -s localhost:8083/connectors/mm2-hbt/status | jq .
```

Please keep in mind that the json files provided are sample MirrorMaker 2 connector configurations and that they need to ebe updated accordingly.

### Consumer Group Offset Sync
NOTE: Do this step only if you’re using the consumer migration methodology with a background process to sync MM2 checkpointed offsets to the __consumer_offsets internal topic at the destination.

You can view and download the source code of the Consumer Group Offset Sync Application here: https://github.com/aws-samples/mirrormaker2-msk-migration/tree/master/MM2GroupOffsetSync

### Default Replication Policy
Using Default Replication Policy and a background process to sync MM2 checkpointed offsets to the __consumer_offsets internal topic at the destination
Note: This requires NO change to the consumer code except the topic pattern it subscribes to. This does require a separate application that syncs offsets, runs in the background and has a dependency on connect-mirror-client and kafka-clients available with Apache Kafka 2.5.0 and above.

Steps:

* Start MM2.
This will detect the topic and create it in the destination and start replicating messages. As mentioned above, the replicated topic at the destination will be created in the format `<source-cluster-alias>.<topic>` as it uses the DefaultReplicationPolicy. By default, it will use an `auto.offset.reset = earliest` setting and start reading messages from the beginning of the retention period for the topic in the source.

* Start the MM2GroupOffsetSync application.
This application periodically syncs the MM2 checkpointed offsets to the __consumer_offsets internal topic at the destination. In order to start the app do the following:

  * Go to the /tmp/kafka dir. Make a copy of the /tmp/kafka/consumer.properties file and update BOOTSTRAP_SERVERS_CONFIG to point to the destination Kafka cluster.

    *  ``` cd /tmp/kafka
           cp /tmp/kafka/consumer.properties /tmp/kafka/consumer.properties_sync_dest
           vim /tmp/kafka/consumer.properties_sync_dest
       ```

   *  The below command runs the help command to show the usage of the application. 

        * ```
            java -jar /tmp/kafka/MM2GroupOffsetSync-1.0-SNAPSHOT.jar -h
          ```
   * Run the MM2 Consumer Group Offset Sync Application. To kill the application process, note down the pid (process id) of the application process. Use kill <pid> to kill the process.
        * ```
           java -jar /tmp/kafka/MM2GroupOffsetSync-1.0-SNAPSHOT.jar -cgi mm2TestConsumer1 -src msksource  -pfp /tmp/kafka/consumer.properties_sync_dest 
          ```

* Stop the consumer.
The consumer should be designed to do a commitsync before shutting down cleanly, in which case the last consumed offset will be committed to the source __consumer_offsets topic and then checkpointed by MM2 to the destination in the msksource.checkpoints.internal topic. If the consumer does not shutdown cleanly, or the consumer is re-started pointing to the destination within the configured checkpoint interval, there could be some duplicate records when it resumes. In order to deal with that, the consumer should be idempotent. In our case, the consumer does do a commitsync when it shuts down.

* Wait for the MM2GroupOffsetSync application to sync the last committed offset, then stop the application.
The application checks to make sure the consumer group in the destination is empty before syncing offsets so as not to overwrite the consumer after it has failed over. However, shutting it down would be safer.

* Start the consumer against the destination Apache Kafka cluster by changing the bootstrap brokers configuration.

* In this case, MM2 utilizes the DefaultReplicationPolicy implementation. As mentioned above, this creates topics in the destination cluster in the <source-cluster-alias>.<topic> format. The consumer, when it starts up will subscribe to the replicated topic based on the topic pattern specified which should account for both the source topic and the replicated topic names.

* It will find the offsets for its consumer group in the destination, and it will start consuming from the last synced offset.

* At this point, the producer is still producing messages to the source which are getting replicated to the destination, and the consumer is reading the replicated messages.

* Stop the producer.
This could be at any convenient time after the consumer has moved over.

(Optional) Create the source topic at the destination.
The consumer can either continue to read messages from the replicated topic (<source-cluster-alias>.<topic>) and the producer is modified to point to the replicated topic which will require a producer code change or the source topic can be created in the destination which will require no producer code change.

Start the producer against the destination Apache Kafka Amazon MSK cluster by changing the bootstrap brokers configuration.
If the producer is stopped and re-started against the destination before all the messages in the replication pipeline are consumed by the consumer, there could be issues with ordering. If ordering is important, wait for all the messages to replicate before starting the producer.

Stop MM2.

### Custom Replication Policy

Using a custom Replication Policy and a background process to sync MM2 checkpointed offsets to the __consumer_offsets internal topic at the destination
Note: This requires NO change to the consumer code. This does require a separate application that syncs offsets, runs in the background and has a dependency on connect-mirror-client and kafka-clients available with Apache Kafka 2.5.0 and above.

Steps:

* Start MM2.
This will detect the topic and create it in the destination and start replicating messages. The MirrorSourceConnector configuration will include a custom ReplicationPolicy class which will enable the replicated topic at the destination to be created with the same name as the source instead of the default <source-cluster-alias>.<topic> format. By default, it will use an `auto.offset.reset = earliest` setting and start reading messages from the beginning of the retention period for the topic in the source.

* Start the MM2GroupOffsetSync application.
This application periodically syncs the MM2 checkpointed offsets to the __consumer_offsets internal topic at the destination.

    * Go to the /tmp/kafka dir. Make a copy of the /tmp/kafka/consumer.properties file and update BOOTSTRAP_SERVERS_CONFIG to point to the destination Kafka cluster.
    
        *  ``` cd /tmp/kafka
               cp /tmp/kafka/consumer.properties /tmp/kafka/consumer.properties_sync_dest
               vim /tmp/kafka/consumer.properties_sync_dest
           ```
    
       *  The below command runs the help command to show the usage of the application. 
    
            * ```
                java -jar /tmp/kafka/MM2GroupOffsetSync-1.0-SNAPSHOT.jar -h
              ```
       * Run the MM2 Consumer Group Offset Sync Application. To kill the application process, note down the pid (process id) of the application process. Use kill <pid> to kill the process.
            * ```
               java -jar /tmp/kafka/MM2GroupOffsetSync-1.0-SNAPSHOT.jar -cgi mm2TestConsumer1 -src msksource  -pfp /tmp/kafka/consumer.properties_sync_dest -rpc com.amazonaws.kafka.samples.CustomMM2ReplicationPolicy
              ```

* Stop the consumer.
The consumer should be designed to do a commitsync before shutting down cleanly, in which case the last consumed offset will be committed to the source __consumer_offsets topic and then checkpointed by MM2 to the destination in the msksource.checkpoints.internal topic. If the consumer does not shutdown cleanly or the consumer is re-started pointing to the destination within the configured checkpoint interval, there could be some duplicate records when it resumes. In order to deal with that, the consumer should be idempotent. In our case, the consumer does do a commitsync when it shuts down.

* Wait for the MM2GroupOffsetSync application to sync the last committed offset, then stop the application.
The application checks to make sure the consumer group in the destination is empty before syncing offsets so as not to overwrite the consumer after it has failed over. However, shutting it down would be safer.

* Start the consumer against the destination Apache Kafka cluster by changing the bootstrap brokers configuration.

    * The consumer, when it starts up will subscribe to the replicated topic based on the topic pattern specified and pick up the replicated topic which would be the same as the source topic name.
It will find the offsets for its consumer group in the destination, and it will start consuming from the last synced offset.

    * At this point, the producer is still producing messages to the source which are getting replicated to the destination, and the consumer is reading the replicated messages.

* Stop the producer.
This could be at any convenient time after the consumer has moved over.

* Start the producer against the destination Apache Kafka Amazon MSK cluster by changing the bootstrap brokers configuration.
If the producer is stopped and re-started against the destination before all the messages in the replication pipeline are consumed by the consumer, there could be issues with ordering. If ordering is important, wait for all the messages to replicate before starting the producer.

* Stop MM2.

