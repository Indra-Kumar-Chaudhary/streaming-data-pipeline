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
Do the same for orders. For this stream, specifiy that the timestamp of the event is derived from the data itself. Specifically, it's extracted and parsed from the ts field.
</p>
<pre>
    CREATE STREAM orders WITH (
    kafka_topic = 'my-replica-set.logistics.orders',
    value_format = 'avro',
    timestamp = 'ts',
    timestamp_format = 'yyyy-MM-dd''T''HH:mm:ss'
);

</pre>
<pre>
    CREATE STREAM shipments WITH (
    kafka_topic = 'my-replica-set.logistics.shipments',
    value_format = 'avro',
    timestamp = 'ts',
    timestamp_format = 'yyyy-MM-dd''T''HH:mm:ss'
);

</pre>
<h3> Join the streams together </h3>
<p> The goal is to create aunified view of the activity of shipped orders. To do this, we want to include as much customer information on each shipment as possible. The orders collection that we created in MongoDB only had an identifier for each customer, but not their name. Use this identifier to look up the rest of the information by using a stream/table join. To do this, you must re-key the stream into a table by the id field:
</p>

<pre>
    CREATE TABLE customers_by_key AS 
        SELECT id,
                latest_by_offset(name) AS name,
                latest_by_offset(age) AS age
        FROM customers
        GROUP BY id
        EMIT CHANGES;
</pre>
<p>
Now you can enrich the orders with more customer information. The following stream/table join creates a new stream that lifts the customer information into the order event.
</p>

<pre>
CREATE STREAM enriched_orders AS 
    SELECT o.order_id,
            o.price,
            o.currency,
            c.id AS customer_id,
            c.name AS customer_name,
            c.age AS customer_age
    FROM orders AS o
    LEFT JOIN customers_by_key c
    ON o.customer_id = c.id
    EMIT CHANGES;

</pre>

<p>
You can take this further by enriching all shipments with more information about the order and customer. Use a stream/stream join to find orders in the relevant window of time. This creates a new stream called shiped_orders that unifies the shipment, order, and customer information:
</p>

<pre>
    CREATE STREAM shipped_orders WITH (
        kafka_topic = 'shipped_orders'
    ) AS
        SELECT o.order_id,
                s.shipment_id,
                o.customer_id,
                o.customer_name,
                o.customer_age,
                s.origin,
                o.price,
                o.currency
        FROM enriched_orders AS o
        INNER JOIN shipments s
        WITHIN 7 DAYS
        ON s.order_id = o.order_id
        EMIT CHANGES;
</pre>

<h3> Start the Elasticsearch sink connector</h3>
<p>The application must perform searches and analytics over this unified stream of information. To make this easy, spill the information out to Elasticsearch and run the following connector to sink the topic:
</p>

<pre>
    CREATE SINK CONNECTOR enriched_writer WITH (
        'connector.class' = 'io.confluent.connect.elasticsearch.ElasticsearchSinkConnector',
        'connection.url' = 'http://elastic:9200',
        'type.name' = 'kafka-connect',
        'topics' = 'shipped_orders'
    );
</pre>