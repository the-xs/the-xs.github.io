# Zookeeper provisioned in AWS

## The problem

Zookeeper is a service providing the distributed synchronization, which is used by many distributed applications such as Hadoop, Kafak, and Solr.

The implementation of zookeeper client has made it very hard to be provisioned automatically in a cloud environment due to the ephemeral nature of the cloud computing - the instances can be terminated and instantiated any time. The challenge is to allow the started zookeeper instance to join the ensemble automatically. Forturnately there is [Exhibitor](https://github.com/soabase/exhibitor/wiki) to help with it. 

## Exhibitor
The purpose of Exhibitor is to manage the zookeeper instance and use a shared storage (either shared drive, S3, or another zookeeper cluster) for propagating the zookeeper configurations to all instance within the ensemble. It is meant to be installed on each of the zookeeper instance.

To provision the zookeeper ensemble in AWS, one can create one autoscaling groups with the fixed number of instances desired (1, 3, 5, ...) for the ensemble, but when using code deploy service for deployment, it has to be configured with ```CodeDeployDefault.OneAtATime```. This allows the situation when there is needs to update the stack, the zookeeper instances will be handled one at a time to keep the service running to support the dependent applications. Also another way is to use multiple autoscaling groups with fixed one instance in each ASG. When using cloudformation and code deploy, there can be ```waitOn``` conditions so that one autoscaling group can be handled at a time while others are waiting.

Before starting up the exhibitor, the ```exhibitor.properties``` file should be in place in the shared the storage. To use S3 as the shared storage, one can upload the file to the S3 bucket and configure the exhibitor to start with the S3 bucket and key name.	

Limitation of the exhibitor to use S3 is it does not support the encryption and there is no configuration to support it. Since it is no longer maintained by Netflix, one can modify the source code to support it. 

To make exhibitor as an service, create the ```/etc/init.d/exhibitor```:

```
#!/bin/bash
### BEGIN INIT INFO
# Provides:          Exhibitor/Zookeeper
# Required-Start:    $all
# Required-Stop:     $all
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Start/Stop Exhibitor/Zookeeper
# Description:       Starts/Stops Exhibitor/Zookeeper
### END INIT INFO

. /etc/init.d/functions

if [ -f /etc/exhibitor/exhibitor.conf ]; then
    . /etc/exhibitor/exhibitor.conf  #custom configuraiton
fi
EXHIBITOR_OPTS="-Xms256M -Xmx256M"
EXHIBITOR_ARGS="--configtype s3 --hostname $EXHIBITOR_HOST --port $EXHIBITOR_PORT --s3region $AWS_REGION --s3config $EXHIBITOR_S3_CONFIG --s3configprefix $EXHIBITOR_S3_CONFIG_PREFIX --s3ssealgorithm $EXHIBITOR_S3_SSE_ALGORITHM"

case "$1" in
    start)
        echo -n "Starting Exhibitor..."

        # retrieving pid of the parent process
        /bin/su -l "$EXHIBITOR_USER" --shell=/bin/bash -c "$JAVA_HOME/bin/java $EXHIBITOR_OPTS -jar $EXHIBITOR_HOME/exhibitor.jar $EXHIBITOR_ARGS >> $EXHIBITOR_LOG_FILE 2>&1 &"
        echo $(ps aux | grep java | grep exhibitor  | awk '{print $2}') > "$EXHIBITOR_PID"
        if [ $? == "0" ]; then
            success
        else
            failure
        fi
        echo
        ;;
    status)
        status -p "$EXHIBITOR_PID" exhibitor
        ;;
    stop)
        echo -n "Killing Exhibitor ..."
        killproc -p "$EXHIBITOR_PID" exhibitor
        echo
        ;;
    restart)
        $0 stop
        $0 start
        ;;
    *)
        echo "Usage: $0 {start|stop|status|restart}"
        exit 1
esac
```

Notice in the above script, the ```--s3ssealgorithm $EXHIBITOR_S3_SSE_ALGORITHM``` is customized code to add the support for S3 encryption.

## Another problem

Once we have the zookeeper ensemble provisioned using exhibitor, we will have a healthy zookeeper cluster, and it can be maintained automatically. However, the applications that depend on zookeeper could still see a big challenge. The instance IP address will change when zookeeper is terminated and created again. Those applications such as Kafak or Solr will fail to respond once all the instance IPs are refreshed. 

An immediate thought is to use a host name for each zookeeper instance, however the implementation of the zookeeper client makes its impossible to do so, because it implemented ```org.apache.zookeeper.client.StaticHostProvider``` to be instantiated in ```org.apache.zookeeper.ClientCnxn``` [[see code here](https://github.com/apache/zookeeper/blob/70797397f12c8a9cc04895d7ca3459f7c7134f7d/src/java/main/org/apache/zookeeper/ClientCnxn.java#L386)]. When ```StaticHostProvider``` [[see code here](https://github.com/apache/zookeeper/blob/70797397f12c8a9cc04895d7ca3459f7c7134f7d/src/java/main/org/apache/zookeeper/client/StaticHostProvider.java#L58)]  initializes itself, the host name is resolved to the ```InetAddress``` and stored in the ```List```. Once it is instantiated, the solved host IPs are in memory forever.

This issue was report here: [https://issues.apache.org/jira/browse/ZOOKEEPER-2184](https://issues.apache.org/jira/browse/ZOOKEEPER-2184), and a [pull request](https://github.com/apache/zookeeper/pull/150) is created to address the issue and hasn't been merged at this moment when this article is written.

To use this fix, one can merge the code manually and build it to replace the zookeeper-x.y.z.jar file in the application such as Solr.

## Workaround

A workaround is to use the static IP for each of the static host name. The instance can be tagged with proper name and the automation script can assign the ENI as a secondary IP.

To use the bash script to handle this task:

* Find ENI

```
# find a ENI with the name tag that matches the instance's name tag.
eni_id=`aws ec2 describe-network-interfaces --filters Name=tag:Name,Values=$instancetag | grep "NetworkInterfaceId" | awk -F'"' '{print $4}'`
```

* Attach it

```
aws ec2 attach-network-interface --network-interface-id $eni_id  --instance-id $instanceid --device-index 1
```

@the-xs 9/1/2017