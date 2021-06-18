# How the Schema Registry Works<a name="schema-registry-works"></a>

This section describes how the serialization and deserialization processes in Schema Registry work\.

1. Register a schema: If the schema doesn’t already exist in the registry, the schema can be registered with a schema name equal to the name of the destination \(e\.g\., test\_topic, test\_stream, prod\_firehose\) or the producer can provide a custom name for the schema\. Producers can also add key\-value pairs to the schema as metadata, such as source: msk\_kafka\_topic\_A, or apply AWS tags to schemas on schema creation\. Once a schema is registered the Schema Registry returns the schema version ID to the serializer\. If the schema exists but the serializer is using a new version that doesn’t exist, the Schema Registry will check the schema reference a compatibility rule to ensure the new version is compatible before registering it as a new version\.

   There are two methods of registering a schema: manual registration and auto\-registration\. You can register a schema manually via the AWS Glue console or CLI/SDK\.

   When auto\-registration is turned on in the serializer settings, automatic registration of the schema will be performed\. If `REGISTRY_NAME` is not provided in the producer configurations, then auto\-registration will register the new schema version under the default registry \(default\-registry\)\. See [Installing SerDe Libraries](schema-registry-gs.md#schema-registry-gs-serde) for information on specifying the auto\-registration property\.

1. Serializer validates data records against the schema: When the application producing data has registered its schema, the Schema Registry serializer validates the record being produced by the application is structured with the fields and data types matching a registered schema\. If the schema of the record does not match a registered schema, the serializer will return an exception and the application will fail to deliver the record to the destination\. 

   If no schema exists and if the schema name is not provided via the producer configurations, then the schema is created with the same name as the topic name \(if Apache Kafka or Amazon MSK\) or stream name \(if Kinesis Data Streams\)\.

   Every record has a schema definition and data\. The schema definition is queried against the existing schemas and versions in the Schema Registry\.

   By default, producers cache schema definitions and schema version IDs of registered schemas\. If a record’s schema version definition does not match what’s available in cache, the producer will attempt to validate the schema with the Schema Registry\. If the schema version is valid, then its version ID and definition will be cached locally on the producer\.

   You can adjust the default cache period \(24 hours\) within the optional producer properties in step \#3 of [Installing SerDe Libraries](schema-registry-gs.md#schema-registry-gs-serde)\.

1. Serialize and deliver records: If the record complies with the schema, the serializer decorates each record with the schema version ID, serializes the record based on the Avro schema format selected \(other formats coming soon\), compresses the record \(optional producer configuration\), and delivers it to the destination\.

1. Consumers deserialize the data: Consumers reading this data use the Schema Registry deserializer library that parses the schema version ID from the record payload\.

1. Deserializer may request the schema from the Schema Registry: If this is the first time the deserializer has seen records with a particular schema version ID, using the schema version ID the deserializer will request the schema from the Schema Registry and cache the schema locally on the consumer\. If the Schema Registry cannot deserialize the record, the consumer can log the data from the record and move on, or halt the application\.

1. The deserializer uses the schema to deserialize the record: When the deserializer retrieves the schema version ID from the Schema Registry, the deserializer decompresses the record \(if record sent by producer is compressed\) and uses the schema to deserialize the record\. The application now processes the record\.

**Note**  
Encryption: Your clients communicate with the Schema Registry via API calls which encrypt data in\-transit using TLS encryption over HTTPS\. Schemas stored in the Schema Registry are always encrypted at rest using a service\-managed AWS Key Management Service \(AWS KMS\) key\.

**Note**  
User Authorization: The Schema Registry supports both resource\-level permissions and identity\-based IAM policies\.