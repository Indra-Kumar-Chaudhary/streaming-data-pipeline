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
<pre>

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

