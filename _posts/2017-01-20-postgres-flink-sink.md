---
title: Writing to Postgres from Apache Flink®
description: 
tags: Apache Flink®, PostgreSQL
layout: article
---

We started to play around with Apache Flink® to process some of our event data.

What is Apache Flink®? [Apache Flink®](https://flink.apache.org/ "Apache Flink® at apache.org") is an open-source stream processing framework. It is the latest in streaming technology, providing [high throughput with low-latency and exactly once semantics](http://data-artisans.com/high-throughput-low-latency-and-exactly-once-stream-processing-with-apache-flink/ "dataArtisans article introducing the power of flink")

Something about data artisans, the tutorials and the community?

Link to other blog post I read and used to get started with postgres sinks?

We use Flink to perform a series of transformations on our data. At certain points we want to persist the results to a database (PostgreSQL), from where we serve a REST API.

What are Sinks?
[Data Sinks](https://ci.apache.org/projects/flink/flink-docs-master/dev/datastream_api.html#data-sinks "Flink data sink documentation") are connectors that consume Data Streams and forward them to files, sockets, external systems, or print them.

Flink provides a number of "out of the box" [connectors](https://ci.apache.org/projects/flink/flink-docs-master/dev/connectors/index.html "out of the box connector documentation") with various guarantees. It is also possible to define your own.

There is no out of the box postgres sink for Flink. This does not mean, however, that you have to start from scratch! There is a class, JDBCOutputFormat, that can be used to turn any jdbc connect-able database into a sink.

JDBCOutputFormat is/was part of the flink batch api, however it can also be used as a sink for the data stream api. It seems to be the recommended approach, judging from a few discussions I found on the flink user group.

The JDBCOutputFormat requires a prepared statement, driver and database connection.

~~~~
JDBCOutputFormat jdbcOutput = JDBCOutputFormat.buildJDBCOutputFormat()
     .setDrivername("org.postgresql.Driver")
     .setDBUrl("jdbc:postgresql://localhost:1234/test?user=xxx&password=xxx")
     .setQuery(query)
     .finish();
~~~~

The query is the prepared statement, in our case this is sufficient:

`String query = "INSERT INTO public.cases (caseid, tracehash) VALUES (?, ?)";`

Where our table looks like:

~~~~
CREATE TABLE cases
(
 caseid VARCHAR(255),
 tracehash VARCHAR(255)
);
~~~~

It can store instances of Row. Row is basically just a wrapper for the parameters of the prepared statement.

So now we just need to transform our data stream of cases into rows. We're going to map the id to caseid, and the trace hash to tracehash.

~~~~
DataStream<Case> cases = ...
		
		DataStream<Row> rows = cases.map((MapFunction<Case, Row>) aCase -> {
			Row row = new Row(2); // our table has 2 columns
			row.setField(0, aCase.getId()); //first column is caseid
			row.setField(1, aCase.getTraceHash()); //second column is tracehash
			return row;
		});
~~~~

And finally we can specify the sink:

`rows.writeUsingOutputFormat(jdbcOutput);`

So now every time our window is evaluated we get a new row in the database, woo!

However, my console is being spammed with:

`"Unknown column type for column %s. Best effort approach to set its value: %s."`

This is because I didn't set explicit type values for my columns whenI build the JDBCOutputFormat

~~~~
JDBCOutputFormat jdbcOutput = JDBCOutputFormat.buildJDBCOutputFormat()
     .setDrivername("org.postgresql.Driver")
     .setDBUrl("jdbc:postgresql://localhost:1234/test?user=xxx&password=xxx")
     .setQuery(query)
     .setSqlTypes(new int[] { Types.VARCHAR, Types.VARCHAR }) //set the types
     .finish();
~~~~

And now I don't get spammed with warnings.

Ok, but instead of a new row I want to either create a new row if one does not exist, or update an existing row. I.e. do an upsert.

I'm using PostgreSQL so I will just modify my query to include an ON CONFLICT statement:

`String query = "INSERT INTO public.cases (caseid, tracehash) VALUES (?, ?) ON CONFLICT (caseid) DO UPDATE SET events=?";`

I am also going to add a constraint to my table:

~~~~
CREATE TABLE cases
(
  caseid VARCHAR(255),
  events VARCHAR(255),
  CONSTRAINT cases_unique UNIQUE (caseid)
);
~~~~


Then I want to add some buffering, JDBCOutputFormat has some buffering. Take a look at that.

This is just a fixed size, what is I want to buffer with a size and a timeout, or even better still, take the latest checkpoint into consideration?

What is checkpointing?


