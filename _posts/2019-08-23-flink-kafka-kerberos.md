---
layout: post
title:  "Configuring Apache Flink to connect to a Kerberos secured Kafka cluster"
date:   2019-08-23 19:03:00 +0800
---

The aim of this post is to describe the configuration required for a Flink application, deployed on a Kerberos secured Hadoop/Yarn cluster, to connect to a Kerberos-secured Apache Kafka cluster using two different keytabs. The following steps worked for me. Depending on your environment setup, the specific steps may vary even though the general idea might just be similar.

This post assumes that you are already able to connect to the Hadoop/Yarn cluster. Connecting to a secured Hadoop cluster is fairly straightforward and the [official documentation](https://ci.apache.org/projects/flink/flink-docs-stable/ops/deployment/yarn_setup.html) explains it well. 
<!--more-->
### Steps Involved

#### Step 1.
Move the contents of the Kafka JAAS file and include it in the Kafka properties used by the Flink application. Namely, this means setting the Kafka properties as follows:

``` bash
bootstrap.servers: "server_address:port_1, server_address:port_2"
group.id: "..."
...
...
security.protocol: "SASL_PLAINTEXT"
sasl.mechanism: "GSSAPI"
sasl.kerberos.service.name: "kafka"
sasl.jaas.config="""com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    refreshKrb5Config=true
    storeKey=true
    serviceName="kafka"
    keyTab="<KAFKA_CONFIG_DIR>/filename.keytab"
    principal="your@assigned.princpal" """;
```
**Note:** The **KAFKA_CONFIG_DIR** above refers to a directory in your local environment where the keytab and other Kafka configuration files are stored. The path specified should be relative. Keep note of this folder name in the `keyTab="<KAFKA_CONFIG_DIR>/filename.keytab"` line because you're going to need it later. 

#### Step 2. 
Add the Kafka `krb5.conf` to the same folder above so that `KAFKA_CONFIG_DIR` now contains two files: 

- The filename.keytab file
- The kafka_krb5.conf file

#### Step 3.
In the flink config, instead of `env.java.opts` set the following JVM properties:
``` bash
env.java.opts.jobmanager: "-Djava.security.krb5.conf=KAFKA_CONFIG_DIR/kafka-krb5.conf"
env.java.opts.taskmanager: "-Djava.security.krb5.conf=KAFKA_CONFIG_DIR/kafka-krb5.conf"
```
#### Step 4.
For the Flink task managers to find the keytab, you'll need to include it in the `flink run` command via the `--yarnship (-yt)` flag. Unfortunately, `yarnship` only supports folders or JARs (atleast in Flink v1.8) so you'll have to include the folder that contains the required files. In this case **KAFKA_CONFIG_DIR/**.
 
So the `flink run` command roughly looks like this:
``` bash
flink run \
    -d  \
    -m yarn-cluster \
    -ynm $STREAM_NAME \
    -yt <ABSOLUTE_PATH_TO_KAFKA_CONFIG_DIR> \
    the_application.jar
```

As the job is being submitted to the cluster, you will notice in the debug logs that the contents of the KAFKA_CONF_DIR directory is being copied from the local environment to each of the yarn containers. As a result, for the containers, the keytab files are accessible locally at the path `KAFKA_CONF_DIR/filename.keytab`. This is the reason for setting the keytab value to the relative path in Step 1 above.

```bash
2019-08-23 02:19:26,175 DEBUG org.apache.flink.yarn.Utils   - Copying from file:<ABSOLUTE_PATH_TO_KAFKA_CONFIG_DIR>/kafka_client_jaas.conf to hdfs://nameservice1/user/tmp_user/.flink/application_1559073145191_998368/<KAFKA_CONFIG_DIR>/kafka_client_jaas.conf
2019-08-23 02:19:26,219 DEBUG org.apache.flink.yarn.Utils   - Copying from file:<ABSOLUTE_PATH_TO_KAFKA_CONFIG_DIR>/tmp_user.keytab to hdfs://nameservice1/user/tmp_user/.flink/application_1559073145191_998368/<KAFKA_CONFIG_DIR>/tmp_user.keytab
2019-08-23 02:19:26,266 DEBUG org.apache.flink.yarn.Utils   - Copying from file:<ABSOLUTE_PATH_TO_KAFKA_CONFIG_DIR>/kafka-krb5.conf to hdfs://nameservice1/user/tmp_user/.flink/application_1559073145191_998368/<KAFKA_CONFIG_DIR>/kafka-krb5.conf
2019-08-23 02:19:26,313 DEBUG org.apache.flink.yarn.Utils   - Copying from file:/some/local/path/flink-1.8.1/conf/log4j.properties to hdfs://nameservice1/user/tmp_user/.flink/application_1559073145191_998368/log4j.properties
2019-08-23 02:19:26,357 DEBUG org.apache.flink.yarn.Utils   - Copying from file:/some/local/path/flink-1.8.1/lib/slf4j-log4j12-1.7.15.jar to hdfs://nameservice1/user/tmp_user/.flink/application_1559073145191_998368/lib/slf4j-log4j12-1.7.15.jar
2019-08-23 02:19:26,404 DEBUG org.apache.flink.yarn.Utils   - Copying from file:/some/local/path/flink-1.8.1/lib/log4j-1.2.17.jar to hdfs://nameservice1/user/tmp_user/.flink/application_1559073145191_998368/lib/log4j-1.2.17.jar
2019-08-23 02:19:26,449 DEBUG org.apache.flink.yarn.Utils   - Copying from file:/some/local/path/my_application.jar to hdfs://nameservice1/user/tmp_user/.flink/application_1559073145191_998368/my_application.jar
2019-08-23 02:19:26,493 DEBUG org.apache.flink.yarn.Utils   - Copying from file:/some/local/path/flink-1.8.1/lib/flink-dist_2.11-1.8.1.jar to hdfs://nameservice1/user/tmp_user/.flink/application_1559073145191_998368/flink-dist_2.11-1.8.1.jar
```

### Commonly Faced Errors:

#### 1. Hadoop Connectivity
``` bash
Caused by: KrbException: Cannot locate default realm
    at sun.security.krb5.Config.getDefaultRealm(Config.java:1029)
    ... 19 more
```
The above error most likely means that you are unable to connect to the Hadoop cluster itself. This is probably because Flink can't find the Hadoop `krb5.conf` file. Set the `JVM_ARGS` param as follows. Note that Flink also adds some JVM args at runtime depending on how you run it, so when you add params you should also include the existing `JVM_ARGS` if any as follows:
``` bash
export JVM_ARGS="$JVM_ARGS -Djava.security.krb5.conf=/path/to/krb5.conf
```

#### 2. Kafka JAAS config

``` bash
Caused by: org.apache.kafka.common.KafkaException: java.lang.IllegalArgumentException: Could not find a 'KafkaClient' entry in the JAAS configuration. System property 'java.security.auth.login.config' is /tmp/jaas-8659858782382158091.conf
```
This is probably because the Flink task managers cannot find the Kafka JAAS config file. See steps 2 - 4 above. 

