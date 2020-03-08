---
layout: post
title:  "Serializing Spark Dataframes to Avro using KafkaAvroSerializer"
date:   2019-07-04 15:03:48 +0800
tags: [Apache Spark, Apache Kafka, Confluent Schema Registry]
---
I recently worked on a project that used Spark Structured Streaming using Apache Spark, Confluent SchemaRegistry and Apache Kafka. Due to some versioning constraints between the various components, I had to write a custom implementation of the KafkaAvroSerializer class for serializing Spark Dataframes into Avro format. The serialized data was then published to Kafka. This post is based on the examples specified in the Confluent documentation here.

In newer versions of Confluent Schema Registry, lot of the implementations detailed below have been simplified and much easier to use. The standard recommended usage of the Confluent KafkaAvroSerializer is fairly simple in that it requires you to set it as one of the Kafka properties that is used when initializing a KafkaProducer:

{% highlight scala %}
val kafkaProperties = new Properties();
props.put(...)
...
...
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, io.confluent.kafka.serializers.KafkaAvroSerializer.class
val producer = new KafkaProducer(props);
{% endhighlight %}

<!--more-->

This abstracts out many of the implementation specifics and details. The way this works is that when the object to be published to Kafka is sent using the `KafkaProducer`, internally the `KafkaAvroSerializer` does the following:

1. Use the provided schema and initialise an internal cache that is used to store the schema for future use.
2. Register the schema at the Schema Registry URL provided at a path that corresponds to `KAFKA_KEY_NAME-value`.
3. If the publishing in (2) is successful and the serialisation process succeeds, the serialized message is returned as type `Array[Byte]`


#### Implementation using Spark dataframes
Each row in our dataframe corresponds to a single Kafka message. The message key is of type String while the message value would be an array of bytes. A custom implementation requires a few things to be pulled out from under the hood. In this case, we perform three specific tasks:

1. Get an existing schema from a known source
2. Initialise the `KafkaAvroSerializer` object
3. Serialize the data using the object defined in (2) and publish it to Kafka using the standard `KafkaProducer`

The rest of this post covers three areas of the process which are manually implemented.

#### Get an existing schema from a SchemaRegistry URL:

In the implementation detailed below, we first define the required `CachedSchemaRegistry` client and use it to fetched the schema from an endpoint by providing a Subject name.

{% highlight scala %}
df.foreachPartition(currentPartition => {
val client = new CachedSchemaRegistryClient("http://localhost:80/", Integer.MAX_VALUE)
val schema = client.getLatestSchemaMetadata("YOUR_SUBJECT_NAME-value").getSchema
val schemaParser = new Schema.Parser
val parsedSchema = schemaParser.parse(schema)
})
{% endhighlight %}

#### Initialise the serializer 

Using the client defined above and kafka properties defined earlier, we then initialise the `KafkaAvroSerializer`. The serializer registers the schema at the URL specified when it serializes an object. We also initialise the Kafka producer at this stage for each partition so that it can be reused by each row in the partition. By defining these objects within the `foreachPartition`, we can ensure that they can be serialized correctly thereby avoiding any `SerializationException` errors.:

{% highlight scala %}
val serializer = new KafkaAvroSerializer(client)
serializer.configure(kafkaProperties, false)
val producer = new KafkaProducer[String, Array[Byte]](kafkaProperties)
{% endhighlight %}

#### Serialize the data and publish to Kafka 

One of the side effects of using the `KafkaAvroSerializer` is that it attempts to register the schema at the specified SchemaRegistry URL at the path `KAFKA-KEY-value`. The newer versions of SchemaRegistry include the option to disable this, however, the older version I worked with did not. Lastly, as mentioned earlier, each row of our dataframe corresponds to a single Kafka message and needs to be serialized individually. We begin doing this by creating the `GenericData.Record` object and passing the schema to it. We then use it to convert each `Row` object to it. Specific implementation for this is included in the supplementary section below:

{% highlight scala %}
currentPartition.foreach(row > {
val avroRecord = new GenericData.Record(parsedSchema)
val serializedMessage = serializer.serialize(key, avroRecord)
producer.send(new ProducerRecord[String, Array[Byte]](KAFKA_TOPIC_NAME, key, serializedMessage))
})
producer.flush()
producer.close()
{% endhighlight %}

