---
title: Writing to PostgreSQL from Apache Flink®
description: 
tags: Apache Flink®, PostgreSQL
layout: article
---

We started to play around with Apache Flink® to process some of our event data.

[Apache Flink®](https://flink.apache.org/ "Apache Flink® at apache.org") is an open-source stream processing framework. It is the latest in streaming technology, providing [high throughput with low-latency and exactly once semantics](http://data-artisans.com/high-throughput-low-latency-and-exactly-once-stream-processing-with-apache-flink/ "dataArtisans article introducing the power of flink")

Something about data artisans, the tutorials and the community?

Link to other blog post I read and used to get started with PostgreSQL sinks?

We use Flink to perform a series of transformations on our data. At certain points we want to persist the results to a database (PostgreSQL), from where we serve a REST API.

In order to persist out results to some outside system, we have to use a data sink. [Data Sinks](https://ci.apache.org/projects/flink/flink-docs-master/dev/datastream_api.html#data-sinks "Flink data sink documentation") are connectors that consume Data Streams and forward them to files, sockets, external systems, or print them.

Flink provides a number of "out of the box" [connectors](https://ci.apache.org/projects/flink/flink-docs-master/dev/connectors/index.html "out of the box connector documentation") with various guarantees. It is also possible to define your own.

There is no out of the box PostgreSQL sink for Flink. This does not mean, however, that you have to start from scratch! The [`JDBCOutputFormat`](https://github.com/apache/flink/blob/4d27f8f2deef9fad845ebc91cef121cf9b35f825/flink-connectors/flink-jdbc/src/main/java/org/apache/flink/api/java/io/jdbc/`JDBCOutputFormat`.java "github for `JDBCOutputFormat`") class can be used to turn any jdbc connect-able database into a sink.

`JDBCOutputFormat` is/was part of the flink batch api, however it can also be used as a sink for the data stream api. It seems to be the recommended approach, judging from a few discussions I found on the flink user group.

The `JDBCOutputFormat` requires a prepared statement, driver and database connection.

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

The `JDBCOutputFormat` can only store instances of Row. A Row is basically just a wrapper for the parameters of the prepared statement. This means need to transform our data stream of cases into rows. We're going to map the id to caseid, and the trace hash to tracehash by implementing a Flink MapFunction.

~~~~
DataStream<Case> cases = ...
		
		DataStream<Row> rows = cases.map((MapFunction<Case, Row>) aCase -> {
			Row row = new Row(2); // our prepared statement has 2 parameters
			row.setField(0, aCase.getId()); //first parameter is caseid
			row.setField(1, aCase.getTraceHash()); //second paramater is tracehash
			return row;
		});
~~~~

And finally we can specify the sink:

`rows.writeUsingOutputFormat(jdbcOutput);`

So now every time our window is evaluated we get a new row in the database, woo!

However, my console is being spammed with:

`"Unknown column type for column %s. Best effort approach to set its value: %s."`

This is because I did not set explicit type values for my columns when I built the `JDBCOutputFormat`. I can do so using the builder and simply passing in an array of java.sql.Types.

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

This means I have a new parameter, and I must specify a value for this. I need to do so in my MapFunction, which now looks like this:

~~~~
DataStream<Case> cases = ...
		
		DataStream<Row> rows = cases.map((MapFunction<Case, Row>) aCase -> {
			Row row = new Row(3); // our prepared statement has 3 parameters
			row.setField(0, aCase.getId()); //first parameter is caseid
			row.setField(1, aCase.getTraceHash()); //second paramater is tracehash
			row.setField(2, aCase.getTraceHash()); //third parameter is also tracehash
			return row;
		});
~~~~

I must also add a type for this parameter when I build the `JDBCOutputFormat`:

~~~~
JDBCOutputFormat jdbcOutput = JDBCOutputFormat.buildJDBCOutputFormat()
     .setDrivername("org.postgresql.Driver")
     .setDBUrl("jdbc:postgresql://localhost:1234/test?user=xxx&password=xxx")
     .setQuery(query)
     .setSqlTypes(new int[] { Types.VARCHAR, Types.VARCHAR, TypEs.VARCHAR }) //set the types
     .finish();
~~~~

I also need to add a constraint to my table:

~~~~
CREATE TABLE cases
(
  caseid VARCHAR(255),
  events VARCHAR(255),
  CONSTRAINT cases_unique UNIQUE (caseid)
);
~~~~

So now when I run my job I do not get any two rows with the same caseid.

Then I want to add some buffering, `JDBCOutputFormat` has some buffering. Take a look at that.

This is just a fixed size, what is I want to buffer with a size and a timeout, or even better still, take the latest checkpoint into consideration?

What is checkpointing?

