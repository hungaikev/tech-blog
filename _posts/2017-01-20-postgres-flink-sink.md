---
title: Writing to Postgres from Apache Flink
description: 
tags: Flink, Postgres
layout: article
---

We started to play around with Apache Flink to process some of our event data.

What is Apache Flink?

We use flink to perform a series of transformations on that data, and at certain points we want to persist the results to a database (Postgres).

We’re using this database to serve a REST API.

What are Sinks?

There is no out of the box postgres sink for Flink, but there is a class, JDBCOutputFormat.

To use any jdbc connect-able database as a sink you can use the JDBCOutputFormat.

JDBCOutputFormat is/was part of the flink batch api, however it can also be used as a sink for the data stream api. It seems to be the recommended approach, judging from a few discussions I found on the flink user group.

The JDBCOutputFormat requires a prepared statement, driver and database connection.
It can store instances of Row. Row is basically just a wrapper for the parameters of the prepared statement.

JDBCOutputFormat output = JDBCOutputFormat.buildJDBCOutputFormat()
     .setDrivername("org.postgresql.Driver")
     .setDBUrl("jdbc:postgresql://localhost:1234/test?user=xxx&password=xxx")
     .setQuery(query)
     .finish();
cases.writeUsingOutputFormat(output);

The query is the prepared statement.
In our case:
INSERT INTO public.cases (caseid, events)
"VALUES (?, ?) "

CREATE TABLE cases
(
 caseid VARCHAR(255),
 events TEXT
--   CONSTRAINT cases_unique UNIQUE (caseid)
);


So now we just need to transform our Datastream<Case> into Row objects.


So now everytime our window is evaluated we get a new row in the database, woo!

Ok, but instead of a new row I want to either create a new row if one does not exist, or update an existing row. I.e. do an upsert.


