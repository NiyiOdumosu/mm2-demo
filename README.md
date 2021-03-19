# mm2-demo
This repository is meant to demo Mirror Maker 2 for Kafka

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