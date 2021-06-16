## KSQL

You need linux to use all the below commands, open command prompt and run below command for at least 4 different windows

```
wsl.exe -u root
```


Now run `ksql client` which is console based client, will connect to ksql server.

```
ksql
```


Basic commands, to list streams and tables.

```
SHOW STREAMS;

SHOW TABLES;

```

confluent kafka has datagen utility, which can generate data for learning purpose..

clickstream like example

1. users stream
    generate users who has userId, gender, region data
2. pageview stream
     generate pageviews which has userId, pageId, time
     
join can be done with userId with both the stream

## Preparation

Launch Linux Shell 1 to produce users data

below produce the records, write to topic users

```
ksql-datagen quickstart=users format=json topic=users maxInterval=60000 iterations=5000000
```

Launch Linux  Shell 2

below produce the pageview records, write to topic pageviews

```
ksql-datagen quickstart=pageviews format=json topic=pageviews maxInterval=60000 iterations=5000000
```

Launch Linux  Shell 3 for kSQL interactive queries

```
ksql
```

```
SHOW STREAMS;

SHOW TABLES;

CREATE STREAM users_stream (userid varchar, regionid varchar, gender varchar) WITH (kafka_topic='users', value_format='JSON');

SHOW STREAMS;

DESCRIBE users_stream;
```

NON_PERSISTED QUERIES [Means, the output/result is not stored into KAfka Brokers]

To stop the non persisted query, use Ctrl + C

```
select userid, regionid, gender from users_stream EMIT CHANGES;
```

Now generate users records on shell 1, if datagen stop, run again.. 

```
select userid, regionid, gender from users_stream where gender='FEMALE'  EMIT CHANGES;
```

Now generate users records on shell 1, if datagen stop, run again..

```
select userid, regionid, gender from users_stream where gender='MALE'  EMIT CHANGES;
```

Now generate users records on shell 1, if datagen stop, run again..


PERSISTED QUERIES [CREATE STREAM AS ] results written to Kafka
Will be runnign automatically, need to use TERMINATE command to stop them

persisted queries will create topics like users_female, users_male kafka topics, and results shall be published to kafka topics..

```
CREATE STREAM users_female AS SELECT userid AS userid, regionid FROM users_stream where gender='FEMALE';

CREATE STREAM users_male AS SELECT userid AS userid, regionid FROM users_stream where gender='MALE';
```

now check whether new topics created or not

```
SHOW STREAMS;

SHOW TOPICS;
```

Listen for from newly created streams..

```
select * from users_female  EMIT CHANGES;
select * from users_male  EMIT CHANGES;
```

Now create pageviews_stream from pageviews data

```
 CREATE STREAM pageviews_stream (userid varchar, pageid varchar) WITH (kafka_topic='pageviews', value_format='JSON');
 
 select * from pageviews_stream  EMIT CHANGES;
```


now generate pageviews usign datagen



JOIN pages view and users stream

```
CREATE STREAM user_pageviews_enriched_stream AS SELECT users_stream.userid AS userid, pageid, regionid, gender FROM pageviews_stream LEFT JOIN users_stream WITHIN 1 HOURS ON pageviews_stream.userid = users_stream.userid;

select * from user_pageviews_enriched_stream  EMIT CHANGES;
```

Ctrl +C to exit

use window, time slicing, group by, aggregation

```
CREATE TABLE pageviews_region_table WITH (VALUE_FORMAT='JSON') AS SELECT gender, regionid, COUNT() AS numusers FROM user_pageviews_enriched_stream WINDOW TUMBLING (size 60 second) GROUP BY gender, regionid HAVING COUNT() >= 1;

select * from pageviews_region_table  EMIT CHANGES;


```


List the persisted queries

```
SHOW QUERIES;
```

explain the query with query id



C***** - QUERY ID

```
EXPLAIN CTAS_PAGEVIEWS_REGION_TABLE_3; 

```

To stop the query / once stopped, cannot be restarted, need to run fresh query
`Query ID may vary, use the right one from show queries`

```
TERMINATE  CTAS_PAGEVIEWS_REGION_TABLE_3;

```

DROP STREAM and TABLE

```
DROP STREAM  users_male; 


DROP TABLE  pageviews_region;
```
 


# launch Linxu shell 4

for the consumer...

```
kafka-console-consumer --bootstrap-server localhost:9092 --topic USERS_FEMALE --from-beginning "

kafka-console-consumer --bootstrap-server localhost:9092 --topic PAGEVIEWS_REGION_TABLE --from-beginning "
```

