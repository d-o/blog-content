+++
date = '2025-11-10T11:29:26+11:00'
draft = false
title = 'Time series in graphs'
description = "Can a graph database store time series data? Modelling events and measurements in Neo4j with temporal types, sketched out with the arrows app."
author = 'Dean Sellers'
tags = [
    "neo4j",
    "cypher",
    "arrows",
    "time-series",
    "think-in-graphs",
]
+++

# Time series data as a graph

Nearly all embedded devices report *event* based data. That is at time *t* there was an event, like a change of state or a measurement was generated. We generally want to record this event, *e* against it's time *t*. In my use case particularly time series data is at the core of what I need to store and query. There are really good, robust solutions for this already, for instance [influxDB](https://www.influxdata.com/) or [iotdb](https://iotdb.apache.org/).

Why not just use one of these then? This is a pretty good question, and there might be no reason not to. When I started to model the rest of the system though, it really looked like a graph. The connections between components and their relationships to each other make a lot of sense as a graph, and then a graph database is a natural fit for managing that data.

Is it feasible to use the same graph DBMS to store the measurement/event data? In the [Neo4j](https://neo4j.com/) graph DBMS versions since 3.4 support native temporal types and indexes. Using previous versions time series like this were modelled using structures like [time trees](https://github.com/graphaware/neo4j-timetree) but this approach is now unnecessary, at least for my use case, so I didn't spend too much time exploring it.

In a later article I'll compare using a dedicated time series DBMS to the model tested here.

## The model

Using graph modelling terminology a *node* in a graph is usually a noun, a thing, where an *edge*, *vertex* or *relationship* (from here on I'll just use the term *relationship*) is best thought of as a *verb* or an action. To borrow the common examples from Neo4j, an *Actor* is a thing and a *Movie* is a thing, *ACTED_IN*, *DIRECTED* are relationships. In this case I will model the events as the nouns, and the action of observation as a *relationship*. So the *(:Event {data: xxx})* is *\[:SEEN {at: a_time}\]*. In this way an event has both a magnitude (or some state) and a relationship in time to be fully described.

In this basic model we have a device that *PROVIDES* data, *Event*s that are *SEEN* at discrete times. The data associated with these events will probably also have some other meta data, like it's unit of measurement, or an enumeration, or a scaling factor, et cetera. We could add this data to every *Event* instance, although fairly obviously this is a lot of redundant data. One easy method of deduplicating this is to add an intermediate node which provides the description of each event data. This intermediate node will also be useful to query a data source for particular types of events.

Enough talking, what does this look like? I have used the online [arrows app](https://arrows.app) to sketch out what I am thinking.

![Example data source with two time series data points](/time-series-basic-graph.png "Data source with two time series data points")

This is a contrived example of course, but a quick scan reveals an interesting and useful aspect of this approach. In both cases the same data is duplicated at different times. In the case of the state we observed is as being `ok` in three out of the four observations. For the measurement data it had a magnitude repeated twice. In the database we require storage for 5 nodes, to represent 8 events.

You could also go further in the deduplication and remove *SEEN* relationships that don't reflect a change in state, or data. This is completely application dependent, in this case I am assuming that it is important that the data was *SEEN*, even though it may or may not have changed.

## Populate and query a database

Finally, some code! My trusty `lab-rapi3b` is running the community edition of Neo4j and serves it up onto the local and tailscale networks. I can use the built in browser available over http and the [`CYPHER`](https://neo4j.com/docs/cypher-cheat-sheet/5/all/) query language to transfer the model above from arrows into a database.


Exporting the model from arrows as a *Cypher* `MERGE` gives this transaction;


### Populate a database with the model

```
MERGE (n3:DataDesc {name: "This is a measurement"})-[:SEEN {at: "2025-08-28T00:54:31+00:00"}]->(:Event {mag: 0})<-[:SEEN {at: "2025-08-28T00:54:30+00:00"}]-(n3)<-[:PROVIDES]-(:DataSource {name: "A source of data"})-[:PROVIDES]->(n4:DataDesc {name: "This is a state"})-[:SEEN {at: "2025-08-28T00:54:30+00:00"}]->(n8:Event {state: "ok"})<-[:SEEN {at: "2025-08-28T00:54:31+00:00"}]-(n4)-[:SEEN {at: "2025-08-28T00:54:32+00:00"}]->(n8)
MERGE (:Event {mag: 3})<-[:SEEN {at: "2025-08-28T00:54:33+00:00"}]-(n3)-[:SEEN {at: "2025-08-28T00:54:32+00:00"}]->(:Event {mag: 1})
MERGE (n4)-[:SEEN {at: "2025-08-28T00:54:33+00:00"}]->(:Event {state: "error"})
```

After connecting to the database running this query/transaction the results in;


``` plaintext
Added 8 labels, created 8 nodes, set 16 properties, created 10 relationships, completed after 1757 ms.
```

Ok, so the Rapi is pretty slow, running the same query on an instance local to my laptop results in


``` plaintext
Added 8 labels, created 8 nodes, set 16 properties, created 10 relationships, completed after 177 ms.
```

So the Rapi is fairly resource constrained, I would certainly not use it as a production server. For the purposes of these experiments though it will work fine. It's speed will certainly motivate me to pay attention to query performance. It will also be interesting to see how long the SD card holding the filesystem lasts!

Now I can do some exploration of the data using *Cypher* in the database browser.

### Query all events

``` cypher
MATCH (n)-[r:SEEN]->(x)
RETURN n.name AS Event, r.at as Seen,
CASE x.mag
    WHEN IS NULL THEN x.state
    ELSE x.mag
END AS Data ORDER BY r.at
```

``` plaintext
╒════════════════════╤═══════════════════════════╤═══════╕
│Event               │Seen                       │Data   │
╞════════════════════╪═══════════════════════════╪═══════╡
│"This a measurement"│"2025-08-28T00:54:30+00:00"│0      │
├────────────────────┼───────────────────────────┼───────┤
│"This is a state"   │"2025-08-28T00:54:30+00:00"│"ok"   │
├────────────────────┼───────────────────────────┼───────┤
│"This a measurement"│"2025-08-28T00:54:31+00:00"│0      │
├────────────────────┼───────────────────────────┼───────┤
│"This is a state"   │"2025-08-28T00:54:31+00:00"│"ok"   │
├────────────────────┼───────────────────────────┼───────┤
│"This a measurement"│"2025-08-28T00:54:32+00:00"│1      │
├────────────────────┼───────────────────────────┼───────┤
│"This is a state"   │"2025-08-28T00:54:32+00:00"│"ok"   │
├────────────────────┼───────────────────────────┼───────┤
│"This a measurement"│"2025-08-28T00:54:33+00:00"│3      │
├────────────────────┼───────────────────────────┼───────┤
│"This is a state"   │"2025-08-28T00:54:33+00:00"│"error"│
└────────────────────┴───────────────────────────┴───────┘
```

Great! I can see my events in order and when they happened, *BUT* the quotes around the times are suspicious. They are ordered as I expect, but ISO date/time strings sort lexically so that isn't really a clue and temporal values are implicitly returned as strings if no other format is specified. I wonder how they are being stored?

### Query the *SEEN* relationship's property type

``` cypher
MATCH ()-[r:SEEN]->()
WITH valueType(r.at) as at_type, r
RETURN DISTINCT at_type AS seen_at_type, count(r) as occurrences
```

``` plaintext
╒═════════════════╤═══════════╕
│seen_at_type     │occurrences│
╞═════════════════╪═══════════╡
│"STRING NOT NULL"│8          │
└─────────────────┴───────────┘
```

So it appears that the time stamp property of the *SEEN* relationships has been stored as a string, rather than a [temporal](https://neo4j.com/docs/cypher-manual/current/values-and-types/temporal/) type, which is what I want the model to reflect. This isn't a surprise really as I couldn't find a method using arrows to specify a data type of a property. It is easy to fix though.

### Change the *SEEN*'s *at* property to a *ZONED DATETIME*

``` cypher
MATCH ()-[r:SEEN]->()
WHERE valueType(r.at) STARTS WITH "STRING"
SET r.at = datetime(r.at)
```

``` plaintext
Set 8 properties, completed after 227 ms.
```

Again, real order of magnitude difference in performance, the same query on my development machine.

``` plaintext
Set 8 properties, completed after 11 ms.
```

Check the data type of the *SEEN* timestamp property again.

``` plaintext
╒═════════════════════════╤═══════════╕
│seen_at_type             │occurrences│
╞═════════════════════════╪═══════════╡
│"ZONED DATETIME NOT NULL"│8          │
└─────────────────────────┴───────────┘
```

### More queries

Now I have the database populated as per the model I can run some other queries.

1. Find the last time the device reported an "ok" status
    ``` cypher
    MATCH (d:DataDesc)-[r:SEEN]->(e:Event)
    WHERE d.name CONTAINS "state" AND toLower(e.state) = "ok"
    RETURN r.at AS On, e.state AS Event ORDER BY r.at DESC LIMIT 1
    ```

    ``` plaintext
    ╒══════════════════════╤═════╕
    │On                    │Event│
    ╞══════════════════════╪═════╡
    │"2025-08-28T00:54:32Z"│"ok" │
    └──────────────────────┴─────┘
    ```

 2. Get a time v magnitude table for the measurement data

    ``` cypher
    MATCH (d:DataDesc)-[r:SEEN]->(e:Event)
    WHERE d.name CONTAINS "measurement"
    RETURN r.at AS At, e.mag AS Magnitude ORDER BY r.at DESC
    ```

    ``` plaintext
    ╒══════════════════════╤═════════╕
    │At                    │Magnitude│
    ╞══════════════════════╪═════════╡
    │"2025-08-28T00:54:33Z"│3        │
    ├──────────────────────┼─────────┤
    │"2025-08-28T00:54:32Z"│1        │
    ├──────────────────────┼─────────┤
    │"2025-08-28T00:54:31Z"│0        │
    ├──────────────────────┼─────────┤
    │"2025-08-28T00:54:30Z"│0        │
    └──────────────────────┴─────────┘
    ```

## Conclusion

It looks like I have a workable model for the core of my planned device data storage and consuming API. There are a few things that need to be investigated further though. For instance if we wanted to query a measurement series for `time in the OK state`, how would that work? Also I should compare the data storage requirements with other solutions.

In an upcoming article I'll use the python driver to connect and populate the model with some random data and do some more real world profiling.
