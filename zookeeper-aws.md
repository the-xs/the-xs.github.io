# Zookeeper provisioned in AWS

## The problem

Zookeeper is a service discovery tool used by many distributed applications such as Hadoop, Kafak, and Solr. 

The implementation of zookeeper client has made it very hard to be provisioned automatically in a cloud environment due to the ephemeral nature of the cloud computing, the instances can be terminated and instantiated any time. The challenge is to allow the started zookeeper instance to join the ensemble automatically. Forturnately there is [Exhibitor](https://github.com/soabase/exhibitor/wiki) to help with it. 

## Exhibitor
The purpose of Exhibitor is to manage the zookeeper instance and use a shared storage (either shared drive, S3, or another zookeeper cluster) for propagating the zookeeper configurations to all instance within the ensemble. It is meant to be installed on each of the zookeeper instance.

To provision the zookeeper ensemble in AWS, we will need to create one autoscaling groups with the fixed number of instances desired for the ensemble, but when using code deploy service for deployment, it has to be configured with ```CodeDeployDefault.OneAtATime```. This allows the situation when there is needs to update the stack, the zookeeper instances will be handled one at a time to keep the service uninterruptive. Also there is a solution to use multiple autoscaling groups with fixed one instance in each ASG. When using cloudformation and code deploy, there can be waitOn conditions so that one autoscaling group can be handled at a time while others are waiting.

Before starting up the exhibitor, the exhibitor.properties file should be in place in the shared the storage. To use S3 as the shared storage, one can upload the file to the S3 bucket and configure the exhibitor to start with the S3 bucket and key name.	

Limitation of the exhibitor to use S3 is it does not support the encryption and there is no configuration to support it. Since it is no longer maintained by Netflix, one can modify the source code to support it. 

## Another problem

Once we have the zookeeper ensemble provisioned using exhibitor, we will have a healthy zookeeper cluster, and it can be maintained automatically. However, the applications that depend on zookeeper could still see a big challenge. The instance IP address will change when zookeeper is terminated and created again. Those applications such as Kafak or Solr will fail to respond once all the instance IPs are changed. 

An immediate thought is to use a host name for each zookeeper instance, however the implementation of the zookeeper client makes its impossible to do so, because it implemented ```org.apache.zookeeper.client.StaticHostProvider``` to be instantiated in ```org.apache.zookeeper.ClientCnxn``` [[see code here](https://github.com/apache/zookeeper/blob/70797397f12c8a9cc04895d7ca3459f7c7134f7d/src/java/main/org/apache/zookeeper/ClientCnxn.java#L386)]. When ```StaticHostProvider``` [[see code here](https://github.com/apache/zookeeper/blob/70797397f12c8a9cc04895d7ca3459f7c7134f7d/src/java/main/org/apache/zookeeper/client/StaticHostProvider.java#L58)]  initializes itself, the host name is resolved to the ```InetAddress``` and stored in the ```List```. Once it is instantiated, the solved host IPs are in memory forever.

This issue was report here: [https://issues.apache.org/jira/browse/ZOOKEEPER-2184](https://issues.apache.org/jira/browse/ZOOKEEPER-2184), and a [pull request](https://github.com/apache/zookeeper/pull/150) is created to address the issue and hasn't been merged at this moment when this article is written.

To use this fix, one can merge the code manually and build it to replace the zookeeper jar file in the applications such as Solr.

## Workaround

A workaround is to use the static IP for each of the static host name. The instance can be tagged with proper name and the automation script can assign the ENI as a secondary IP.

@the-xs 9/1/2017