# streaming-data-pipeline
This project is about the to build streaming data pipeline using Apache Kafka

A streaming data pipeline means ingesting the data from some source system into Kafka as the data is created.

And then streaming that data from kafka to one or more target systems.
<h1>
Downloading confluent-hub
</h1>
<h1>
Get the connectors
<h1>
<h3>Create a directory for your components:</h3>
<pre>
<p>mkdir confluent-hub-components
</p></pre>

<h3>Download the Postgres Debezium connector</h3>

<h1>Up and run the docker-compose</h1>
<pre> docker-compose up -d </pre>

<h3>Create the customers table in Postgres</h3>

<h3> Start the Postgres and MongoDB Debezium source Connectors</h3>7
<pre> docker exec -it ksqldb-cli ksql http://ksqldb-server:8088</pre>

<p> Before you issue more cmmands, tell ksqlDB to start all queries from earliest point in each topic:</p>

<pre>
    SET 'auto.offset.rest' = 'earliest';
</pre>

<p>
    Now you can aks Debezium to stream the Postgres changelogs into Kafka. Invoke the following command in ksqlDB, which creates a Debezium source connector and writes all of its changes to Kafka topics:
</p>
<pre>
    CREATE SOURCE CONNECTOR customers_reader WITH (
        'connector.class' = 'io.debezium.connector.postgresql.PostgresConnector',
        'database.hostname' = 'postgres',
        'database.port' = '5432',
        'database.user' = 'postgres-user',
        'database.password' = 'postgres-pw',
        'database.dbname' = 'customers',
        'database.server.name' = 'customers',
        'table.whitelist' = 'public.customers',
        'transforms' = 'unwrap',
        'transforms.unwrap.type' = 'io.debezium.transforms.ExtractNewRecordState',
        'transforms.unwrap.drop.tombstones' = 'false',
        'transforms.unwrap.delete.handling.mode' = 'rewrite'
    );
</pre>

<p>
Run another source connector to ingest the changes from MongoDB. Specify the same bhaivior for discarding the Debezium envelope:
</p>
<pre>
CREATE SOURCE CONNECTOR logistics_reader WITH (
    'connector.class' = 'io.debezium.connector.mongodb.MongoDbConnector',
    'mongodb.hosts' = 'mongo:27017',
    'mongodb.name' = 'my-replica-set',
    'mongodb.authsource' = 'admin',
    'mongodb.user' = 'dbz-user',
    'mongodb.password' = 'dbz-pw',
    'collection.whitelist' = 'logistics.*',
    'transforms' = 'unwrap',
    'transforms.unwrap.type' = 'io.debezium.connector.mongodb.transforms.ExtractNewDocumentState',
    'transforms.unwrap.drop.tombstones' = 'false',
    'transforms.unwrap.delete.handling.mode' = 'drop',
    'transforms.unwrap.operation.header' = 'true'

);
</pre>

<h3>Create the ksqlDB source Stream </h3>
<pre>
    CREATE STREAM customers WITH (
        kafka_topic = 'customers.public.customers',
        value_format = 'avro'
    );
</pre>

<p>
Do the same for orders. For this stream, specifiy that the timestamp of the event is derived from the data itself. Specifically, it's extracted adn parsed from the ts field.
</p>
<pre>
    CREATE STREAM orders WITH (
    kafka_topic = 'my-replica-set.logistics.orders',
    value_format = 'avro',
    timestamp = 'ts',
    timestamp_format = 'yyyy-MM-dd''T''HH:mm:ss'
);

</pre>