# Topic policies & System topic

- Date: Mon Oct 24 2022
- Category: Apache Pulsar

## Preface 

This article attempts to explain the implementation of the Apache Pulsar topic policy function.
As the official documentation for Apache Pulsar is good enough. I will not describe it and how to use it.

If you are interested in learning more about it, please follow this [link](https://pulsar.apache.org/docs/).

## Implementation

Before explain the implementation of topic policy we have to introduce some important components. there are 
Compactor and system topics.

### What is compactor?

![Topic compaction](./topic%20compaction.png)

Apache pulsar supports the compression of topics by specific keys.
This process you can understand from the diagram above.

> In this article, I will not cover the details of message compression.
Details of this component will be covered in other articles.

### What is system topic

The system topic, as its name says, is a topic that will be used internally by pulsar to synchronise and communicate with data within the cluster.
They have special names, in this article we are using a system topic called `__change_events`, if we enable system topic `systemTopicEnabled` and topic policy `topicLevelPoliciesEnabled`,
pulsar will create one for each namespace via the internal client.

### System topic client

pulsar wraps the internal client to create a new component `SystemTopicClient`,
which will provide pulsar with the functionality of a system topic `Reader` and `Writer`,
and the configuration of the reader and writer can be modified for each system topic depending on its needs.
In the implementation of the topic policy feature for `SystemTopicClient` described in this article,
pulsar uses a `Reader` that supports compaction and a `Writer` that needs to carry the message key, so that we can rely on the compaction feature to implement ADD,UPDATE,DELETE operation.


At this point, I think we have a good understanding of how pulsar relies on system topic to implement a simple storage system similar to K-V.
Let's move on to how we rely on this feature to implement a topic policy.

### Topic policies

![Topic Policy](./Topic%20policy.png)

#### Read topic polices

The broker will create a system-topic `Reader` in the background
that will continuously read and compress the data into the current broker's cache.

When a broker needs to fetch a policy for a topic, it will go directly to the cache.

#### Write topic polices

- **ADD**: Sends a message containing the topic policy to the system topic, then the message key is the topic name.

- **UPDATE**: Sends a new message containing the topic policy to the system topic and the message key is the topic name. Dependent on the compression mechanism, the `reader` will only get the new message.

- **DELETE**: Sends a new message including the `null` value, which will be ignored by the compactor in the next round.
