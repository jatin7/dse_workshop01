# DSE-Workshop

![icon](http://i.imgur.com/FoIOBlt.png)

In this session, you'll learn all about DataStax Enterprise. It's a mix between presentation and hands-on. This is **obviously** your reference for the hands-on content. Feel free to bookmark this page for future reference! 

----------


Hands On Setup
-------------

We have an 5 node cluster for you to play with! The cluster is currently running in both **search** and **analytics** mode so you can take advantage of both Spark and Solr on your Cassandra data. 

```

// You can login to any of these nodes 

ssh root@54.215.239.149
ssh root@52.53.237.20
ssh root@54.215.239.161
ssh root@52.53.208.182
ssh root@54.67.72.249

Password: 
        
```

#### UI's you'll want to play around with
 
 - OpsCenter: https://54.215.239.149/opscenter/index.html
 - Spark Master: http://52.53.237.20:7080/
 - Solr Admin UI: http://54.215.239.149:8983/solr/#/

#### Connecting to the cluster using Studio



Hands On DSE Cassandra 
-------------------

Cassandra is the brains of DSE. It's an awesome storage engine that handles replication, availability, structuring, and of course, storing the data at lightning speeds. It's important to get yourself acquainted with the Cassandra to fully utilize the power of the DSE Stack. 

#### Creating a Keyspace, Table, and Queries 

Try the following CQL commands in DevCenter. In addition to DevCenter, you can also use **CQLSH** as an interactive command line tool for CQL access to Cassandra. Start CQLSH like this:

```cqlsh 127.0.0.1``` 
> Make sure to replace 127.0.0.1 with the IP of the respective node 

Let's make our first Cassandra Keyspace! If you are using uppercase letters, use double quotes around the keyspace.

```
CREATE KEYSPACE <Enter your name> WITH replication = {'class': 'NetworkTopologyStrategy', 'DC1': 3 };
```

And just like that, any data within any table you create under your keyspace will automatically be replicated 3 times. 

Note: if you only have 1 node in 1 data center, use SimpleStrategy: 

```
CREATE KEYSPACE <Enter your name> WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 1 };
```

Let's keep going and create ourselves a table. You can follow my example or be a rebel and roll your own. 

```
CREATE TABLE <yourkeyspace>.sales (
	name text,
	time int,
	item text,
	price double,
	PRIMARY KEY (name, time)
) WITH CLUSTERING ORDER BY ( time DESC );
```

> Yup. This table is very simple but don't worry, we'll play with some more interesting tables in just a minute.

Let's get some data into your table! Cut and paste these inserts into DevCenter or CQLSH. Feel free to insert your own data values, as well. 

```
INSERT INTO <yourkeyspace>.sales (name, time, item, price) VALUES ('marc', 20150205, 'Apple Watch', 299.00);
INSERT INTO <yourkeyspace>.sales (name, time, item, price) VALUES ('marc', 20150204, 'Apple iPad', 999.00);
INSERT INTO <yourkeyspace>.sales (name, time, item, price) VALUES ('rich', 20150206, 'Music Man Stingray Bass', 1499.00);
INSERT INTO <yourkeyspace>.sales (name, time, item, price) VALUES ('marc', 20150207, 'Jimi Hendrix Stratocaster', 899.00);
INSERT INTO <yourkeyspace>.sales (name, time, item, price) VALUES ('rich', 20150208, 'Santa Cruz Tallboy 29er', 4599.00);
```

And to retrieve it:

```
SELECT * FROM <keyspace>.sales where name='marc' AND time >=20150205 ;
```
>See what I did there? You can do range scans on clustering keys! Give it a try.

----------


Hands On Cassandra Primary Keys 
-------------------

#### The secret sauce of the Cassandra data model: Primary Key

There are just a few key concepts you need to know when beginning to data model in Cassandra. But if you want to know the real secret sauce to solving your use cases and getting great performance, then you need to understand how Primary Keys work in Cassandra. 

Let's dive in! Check out [this exercise for understanding how primary keys work](https://github.com/robotoil/Cassandra-Primary-Key-Exercise/blob/master/README.md) and the types of queries enabled by different primary keys.

----------


Fleet Example 
-------------------

#### Data Model


Create the following tables that will be used in the fleet management example

```
CREATE TABLE <yourkeyspace>.metrics (
    fleet_id text PRIMARY KEY,
    idle_time double,
    vehicles int
);

CREATE TABLE <yourkeyspace>.vehicle_hist (
    fleet_id text,
    vin text,
    day int,
    time timestamp,
    hour int,
    idle_time double,
    PRIMARY KEY ((fleet_id, vin, day), time)
) WITH CLUSTERING ORDER BY (time DESC);

CREATE TABLE <yourkeyspace>.fleet_daily_rollup (
    fleet_id text,
    day int,
    idle_time double,
    PRIMARY KEY (fleet_id, day)
) WITH CLUSTERING ORDER BY (day ASC);

CREATE TABLE <yourkeyspace>.vehicle_daily_rollup (
    fleet_id text,
    day int,
    idle_time double,
    vin text,
    PRIMARY KEY ((fleet_id, day), idle_time, vin)
) WITH CLUSTERING ORDER BY (idle_time ASC, vin ASC);
```


Once you have created the tables, load the data from sample file [vehicle_hist_small.csv](https://github.com/carolinerg1/DSE-Workshop/blob/master/vehicle_hist_small.csv) using [CQL COPY](http://docs.datastax.com/en/cql/3.1/cql/cql_reference/copy_r.html)

```
COPY <yourkeyspace>.vehicle_hist from 'vehicle_hist_small.csv' with header=true;
```

Another way to load data is by using [Cassandra Loader](https://github.com/brianmhess/cassandra-loader) or by writing a simple app


#### Queries

Try selecting some data:

```
select * from vehicle_hist where fleet_id = '985' and vin = '214876152631' and day = 20160322;
```

What if you want to retrieve all the history for a fleet?

```
select * from vehicle_hist where fleet_id = '985'
```

Why didn't this work? (how to do it: skip to DSE Search section below!)


----------


Hands On Cassandra Consistency 
-------------------

#### Let's play with consistency!

Consistency in Cassandra refers to the number of acknowledgements replica nodes need to send to the coordinator for an operation to be successful while also providing good data (avoiding dirty reads). 

We recommend a ** default replication factor of 3 and consistency level of LOCAL_QUORUM as a starting point**. You will almost always get the performance you need with these default settings.

In some cases, developers find Cassandra's replication fast enough to warrant lower consistency for even better latency SLA's. For cases where very strong global consistency is required, possibly across data centers in real time, a developer can trade latency for a higher consistency level. 

Let's give it a shot. 

>During this exercise, I'll be taking down nodes so you can see the CAP theorem in action. We'll be using CQLSH for this one. 

**In CQLSH**:

```tracing on```
```consistency all```

>Any query will now be traced. **Consistency** of all means all 3 replicas need to respond to a given request (read OR write) to be successful. Let's do a **SELECT** statement to see the effects.

```
SELECT * FROM <yourkeyspace>.sales where name='<enter name>';
```

How did we do? 

**Let's compare a lower consistency level:**
```consistency local_quorum```
>Quorum means majority: RF/2 + 1. In our case, 3/2 = 1 + 1 = 2. At least 2 nodes need to acknowledge the request. 

Let's try the **SELECT** statement again. Any changes in latency? 
>Keep in mind that our dataset is so small, it's sitting in memory on all nodes. With larger datasets that spill to disk, the latency cost become much more drastic. 

**Let's try this again** but this time, let's pay attention to what's happening in the trace
```
consistency local_all
```
```
SELECT * FROM <yourkeyspace>.sales where name='<enter name>';
```

Take a look at the trace output. Look at all queries and contact points. What you're witnessing is both the beauty and challenge of distributed systems. 

```
consistency local_quorum
```
```
SELECT * FROM <yourkeyspace>.sales where name='<enter name>';
```

>This looks much better now doesn't it? **LOCAL_QUORUM** is the most commonly used consistency level among developers. It provides a good level of performance and a moderate amount of consistency. That being said, many use cases can warrant  **CL=LOCAL_ONE**. 

For more detailed classed on data modeling, consistency, and Cassandra 101, check out the free classes at the [DataStax Academy] https://academy.datastax.com website. 

----------


Hands On DSE Search.
-------------
DSE Search is awesome. You can configure which columns of which Cassandra tables you'd like indexed in **lucene** format to make extended searches more efficient while enabling features such as text search and geospatial search. 
  
  
   

Let's start off by indexing the tables we've already made. Here's where the dsetool really comes in handy:

```
dsetool create_core <yourkeyspace>.sales generateResources=true reindex=true
```

>If you've ever created your own Solr cluster, you know you need to create the core and upload a schema and config.xml. That **generateResources** tag does that for you. For production use, you'll want to take the resources and edit them to your needs but it does save you a few steps. 

This by default will map Cassandra types to Solr types for you. Anyone familiar with Solr knows that there's a REST API for querying data. In DSE Search, we embed that into CQL so you can take advantage of all the goodness CQL brings. Let's give it a shot. 

```
SELECT * FROM <keyspace>.<table> WHERE solr_query='{"q":"<column>:*"}';

SELECT * FROM <keyspace>.sales WHERE solr_query='{"q":"name:marc", "fq":"item:*pple*"}'; 
```
> For your reference, [here's the doc](http://docs.datastax.com/en/datastax_enterprise/4.8/datastax_enterprise/srch/srchCql.html?scroll=srchCQL__srchSolrTokenExp) that shows some of things you can do

OK! Time to work with some more interesting data. Meet Amazon book sales data:
>Note: This data is already in the DB, if you want to try it at home, [CLICK ME](https://github.com/Marcinthecloud/Solr-Amazon-Book-Demo). 

Click stream data:
```
CREATE TABLE amazon.clicks (
    asin text,
    seq timeuuid,
    user uuid,
    area_code text,
    city text,
    country text,
    ip text,
    loc_id text,
    location text,
    location_0_coordinate double,
    location_1_coordinate double,
    metro_code text,
    postal_code text,
    region text,
    solr_query text,
    PRIMARY KEY (asin, seq, user)
) WITH CLUSTERING ORDER BY (seq DESC, user ASC);
```
And book metadata: 

```
CREATE TABLE amazon.metadata (
    asin text PRIMARY KEY,
    also_bought set<text>,
    buy_after_viewing set<text>,
    categories set<text>,
    imurl text,
    price double,
    solr_query text,
    title text
);
```

> Example page of what's in the DB http://www.amazon.com/Science-Closer-Look-Grade-6/dp/0022841393/ref=sr_1_1?ie=UTF8&qid=1454964627&sr=8-1&keywords=0022841393

So what are things you can do? 
>Filter queries: These are awesome because the result set gets cached in memory. 
```
SELECT * FROM amazon.metadata WHERE solr_query='{"q":"title:Noir~", "fq":"categories:Books", "sort":"title asc"}' limit 10; 
```
> Faceting: Get counts of fields 
```
SELECT * FROM amazon.metadata WHERE solr_query='{"q":"title:Noir~", "facet":{"field":"categories"}}' limit 10; 
```
> Geospatial Searches: Supports box and radius
```
SELECT * FROM amazon.clicks WHERE solr_query='{"q":"asin:*", "fq":"+{!geofilt pt=\"37.7484,-122.4156\" sfield=location d=1}"}' limit 10; 
```
> Joins: Not your relational joins. These queries 'borrow' indexes from other tables to add filter logic. These are fast! 
```
SELECT * FROM amazon.metadata WHERE solr_query='{"q":"*:*", "fq":"{!join from=asin to=asin force=true fromIndex=amazon.clicks}area_code:415"}' limit 5; 
```
> Fun all in one. 
```
SELECT * FROM amazon.metadata WHERE solr_query='{"q":"*:*", "facet":{"field":"categories"}, "fq":"{!join from=asin to=asin force=true fromIndex=amazon.clicks}area_code:415"}' limit 5;
```
Want to see a really cool example of a live DSE Search app? Check out [KillrVideo](http://www.killrvideo.com/) and its [Git](https://github.com/luketillman/killrvideo-csharp) to see it in action. 


Fleet Example - Advanced Queries
-------------
Going back to our query from before, get all the history for a fleet, here's how you would do it with DSE Search

#### Create Solr Core

```
dsetool create_core <yourkeyspace>.vehicle_hist generateResources=true reindex=true
```

You can view the core via [Solr Admin UI](http://54.215.239.149:8983/solr/#/<yourkeyspace>.vehicle_hist)


#### Sample Queries

Lets try again to get all the history for fleet ID 985:

```
select * from vehicle_hist where solr_query = '{"q":"fleet_id:985"}';
```

Return all the activity on March 23, 2016

```
select * from vehicle_hist where solr_query = '{"q": "time:[2016-03-21T00:00:00Z TO 2016-03-23T23:59:59Z] "}';
```

What about vehicles that are idle for more than 11.999 sec, sorted by by most idle:

```
select * from vehicle_hist where solr_query = '{"q":"idle_time:[11.999 TO *]","sort":"idle_time desc"}' ;
```

Top 10 days most idle:

```
select day, idle_time from vehicle_hist where solr_query = '{"q":"*:*","sort":"idle_time desc"}' limit 10 ;
```

Count of history per day for Fleet ID 985

```
select * from vehicle_hist where  solr_query='{"q":"fleet_id:985","facet":{"range":"time", "f.time.range.start":"2016-03-19T00:00:00Z", "f.time.range.end":"2016-03-29T23:59:59Z", "f.time.range.gap":"+1DAY", "method":"fcs"}}'
```
   
   
   
#### ** Looking for additional DSE Search examples, check out this [ tutorial](https://github.com/carolinerg1/DSE-Workshop/blob/master/DSE%20Search/README.md) ** 
   
   
   
----------



Hands On DSE Analytics
--------------------

Spark is general cluster compute engine. You can think of it in two pieces: **Streaming** and **Batch**. **Streaming** is the processing of incoming data (in micro batches) before it gets written to Cassandra (or any database). **Batch** includes both data crunching code and **SparkSQL**, a hive compliant SQL abstraction for **Batch** jobs. 

It's a little tricky to have an entire class run streaming operations on a single cluster, so if you're interested in dissecting a full scale streaming app, check out [THIS git](https://github.com/retroryan/SparkAtScale).  

>Spark has a REPL we can play in. To make things easy, we'll use the SQL REPL:

```dse spark-sql --conf spark.ui.port=<Pick a random 4 digit number> --conf spark.cores.max=1```

>Notice the spark.ui.port flag - Because we are on a shared cluster, we need to specify a radom port so we don't clash with other users. We're also setting max cores = 1 or else one job will hog all the resources. 

Try some CQL commands

```use <your keyspace>;```
```SELECT * FROM <your table> WHERE...;```

And something not too familiar in CQL...
```SELECT sum(price) FROM <your table>...;```

Let's try having some fun on that Amazon data:

```
SELECT sum(price) FROM metadata;
```
```
SELECT m.title, c.city FROM metadata m JOIN clicks c ON m.asin=c.asin;
```
```
SELECT asin, sum(price) AS max_price FROM metadata GROUP BY asin ORDER BY max_price DESC limit 1;
```
----------


DSE Streaming Demo
--------------------
**Spark Notebook**

[Spark Notebook](http://spark-notebook.io/) is an awesome tool for exploring Spark and making simple visualizations. 

>Have fun with it! See what you come up with :)

----------


Getting Started With DSE Ops
--------------------

Most of us love to have tools to monitor and automate database operations. For Cassandra, that tool is DataStax OpsCenter. If you prefer to roll with the command line, then two core utilities you'll need to understand are nodetool and dsetool.

**Utilities you'll want to know:**
```
nodetool  //Cassandra's main utility tool
dsetool   //DSE's main utility tool
```
**nodetool Examples:**
```
nodetool status  //shows current status of the cluster 

nodetool tpstats //shows thread pool status - critical for ops
```

**dsetool Examples:**
```
dsetool status //shows current status of cluster, including DSE features

dsetool create_core //will create a Solr schema on Cassandra data for Search
```

**The main log you'll be taking a look at for troubleshooting outside of OpsCenter:**
```
/var/log/cassandra/system.log
```


