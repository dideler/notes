tags: #aws #aws-dynamodb #nosql #document-storage
sources: https://www.youtube.com/watch?v=HaEPXoXVf2k

db: wide column key-value store

### history of data processing (why nosql?)
- peaks and valleys when it comes to handling data pressure
- relational db is good at handling low data pressure
    - optimised for storage, not cpu
    - increases cpu costs (e.g. joining tables)
    - cpu is now more expensive than storage
    - nosql optimised for cpu, more cost effective
    
![[dynamodb-talk-2.png]]
- early adoption faces issues due to not understanding how to use
	- relational design patterns, multi tables, normalised data models, etc do not work well with nosql

![[dynamodb-talk-3.png]]
- nosql tradeoffs solve different problems
- nosql better at scaling horizontally than sql
- nosql built for OLTP at scale, bad for OLAP
- nosql optimised for compute, not storage
- nosql models data as denormalised/hierarchical
- nosql not good at "reshaping the data", likes simple queries
- nosql is good when access patterns are repeatable (OLTP)


### overview of dynamodb

![[dynamodb-talk-4.png]]
- dynamodb is a fully managed nosql soln at scale (50+ nodes)
- document or key-value data
- works well with event-driven programming (incl serverless)
- the busier it gets, the more consistent the latency becomes
	- because more stuff gets cached
	- so it improves with load

### nosql data modeling

![[dynamodb-talk-5.png]]

- nosql table is more like a catalog, with many items in a table
- items in table don't have to have same attributes
- item needs one attr to uniquely identify the item (partition key)
- partition key (mandatory) determines data distribution
- sort key (optional) models 1:* relationships (richer queries)
	- enables complex range queries against items in partition
	- can think of partition as bucket now
	- sort key allows operators against it
        - `==, <, >, >=, <=`
        - `begins with`, `between`, `contains`, `in`
        - sorted results, counts, top/bottom n values
    - e.g. partition key = customer id, sort key = order date
        - "give me all the customer's orders in last 24 hours"
        - query: pkey=<customer_id>, skey > 24h ago
    - model the table to support primary access paterns

![[dynamodb-talk-6.png]]
- in addition to uniquely representing an item, partition keys are also used to distribute the items across a key space
- pk used for building an unordered hash index
- all nosql dbs work this way

![[dynamodb-talk-7.png]]
- when db is scaled, we chop up key space and distribute across multiple nodes (physical devices)
- that's why pk has to be provided in query, so system knows where to fetch the data from (which storage node)
- this makes all nosql dbs fast and consistent at any scale
- automatic routing to exact storage node where data lives

![[dynamodb-talk-8.png]]
- when we include sort (aka range) key, system goes into pk partition, collectively fetches items that are sorted on that node
- that's how nosql dbs maintain fast consistent behaviour

![[dynamodb-talk-9.png]]
- partitions are replicated (in dynamo, and many others, it's 3-way)
- when you write, you get an ack when majority has received write
- when you read, have a choice between
    - eventually consistent (default)
    - strongly consistent
- aws recommends "eventually consistent read" because they are half the cost an a "strongly consistent read" because we have more nodes to choose from (random replication node)
- whereas a "strongly consistent read" always uses the primary node (depends on data being on all replicas?) 
    - primary node always accepts the write

![[dynamodb-talk-10.png]]
![[dynamodb-talk-11.png]]
- two types of indexes
    - local secondary index (LSI)
    - global secondary index (GSI)
- LSI allows resorting of data in a partition
    - allows for different access patterns
    - must always use the same partition key as table
    - it's a way to resort the data but not regroup the data
    - strongly consistent
- GSI allows creating a new aggregation of the data
    - e.g. group orders by warehouse instead of by customer
    - eventually consistent
    - table updates GSIs async (online processing)
    - GSI replication speed (write capacity) can throttle table writes
        - provisioning of GSIs must match throughput of tables
- using indexes, we can regroup and resort the data
    - to support secondary access patterns

![[dynamodb-talk-12.png]]

### common nosql design patterns

![[dynamodb-talk-13.png]]
![[dynamodb-talk-14.png]]
- can look at cloudwatch...
    - you're throttling and way below provisioned capacity
    - probably means you have a "hot key"
- this graph shows all access hitting a single storage node
    - high velocity access pattern to a small number of keys
- we want to distribute the access patterns in nosql
	- use partition key with large number of distinct values
	- e.g. don't use binary partition keys, would aggregate a lot of data into very few partitions. uuids are good
	- space: access is evenly spread over the key-space
	- time: requests arrive evenly spaced in time
- in the second (blue) picture, you may have to deprovision the table because it's not doing much - under utilised
- we want heat charts to look like pepperoni pizzas

![[dynamodb-talk-15.png]]
- big advantage of dynamo is its elasticity
- another db (e.g. mongo or cassandra) has to provisioned to handle peak load, which doesn't go away when it's not needed
- dynamodb autoscaling deals with the demand of your system
- good way to manage your db costs

### modeling real applications w/ nosql

![[dynamodb-talk-16.png]]
- designed to maximise efficiency of your access patterns
- not designed for data modeling
- hard because data is relational
- sql db is hard to scale because it spends a lot of time computing a denormalised view, as data is mostly modeled normalised
- sql db data lives in multiple places (as it's normalised), so when it needs to be updated, we need ACID transactions
    - the need for ACID is because of how we model our data with relational dbs

![[dynamodb-talk-17.png]]
- what if we store our data denormalised
- access patterns / queries are simplified (one query vs three)

![[dynamodb-talk-18.png]]
- important for nosql is choosing good partition and sort keys
- for sort key we want to query across entities
    - with a single trip to the db, get all the items we need
    - focus on having a simple and efficient access pattern

![[dynamodb-talk-19.png]]
- with nosql, understand every access pattern *beforehand*
    - if you don't, then you can't model the data efficiently
- what's the use case / nature of your application?
- OLAP does not fit with nosql model
- define the access patterns: read/write workloads
    - identify data sources
    - define query aggregations
    - document all workflows
- nosql is NOT flexible, it's efficient
    - data model is designed for specific access models
    - which couples tightly to the service
- avoid relational design patterns
- use one table (1 application service = 1 table)
    - reduce round trips
    - simplify access patterns
    - identify primary keys (how will items be inserted/read)
    - overload items into partitions
    - define indexes (sort keys) for secondary access patterns

![[dynamodb-talk-20.png]]
- complex queries
- dynamo has streams
    - with a lambda, can be thought of as stored procedures
    - completely disconnected from the table space
    - stored procedures on a shared data service (multiple teams) leads to a lot of complexity and increased risk
        - e.g. bad processing on the db server affects all consumers
    - streams + lambda has an isolated process space
        - don't worry about impacting the dynamodb table
    - stream is the changelog for the nosql table
    - all write operations appear in the stream
    - data in the stream can invoke a lambda function
    - lambda has two IAM roles:
        1. invocation => what it can read (e.g. from the stream)
        2. execution => what it can do (e.g. access to other services)

![[dynamodb-talk-23.png]]
- pattern: triggers for computed aggregations
- one of the most common use cases for streams + lambdas
- e.g. averages, counts, sums, ...
- aggregated data is written back into the table as metadata item
- e.g. timeseries data computes aggregations "offline"
- lambda can also push data into other services
- for high velocity workflows, lambda may not be good enough
    - could instead have a static stream reader on an EC2 instance

![[dynamodb-talk-24.png]]
![[dynamodb-talk-25.png]]
- we want to store hierarchical data in our table - but how?
- composite keys
- sort condition applies before read
- filter condition applies after read
    - get knocked out after read, but still paying to read them
    - has the same read cost as whatever the sort key dictates
- approach 1 is inefficient if you're filtering out 99% of data and reading thousands of records/documents

![[dynamodb-talk-26.png]]
![[dynamodb-talk-27.png]]
![[dynamodb-talk-28.png]]
- with a composite key we can avoid those wasted reads
- use ckeys to create hierarchies on top of the sort key structure
- like a faceted search

![[dynamodb-talk-29.png]]
- OLTP apps use data in a certain way
- primary driver for ACID in nosql dbs
- might still need transactions to
    - maintain version history (audit trail)
    - create multiple items in one pass

![[dynamodb-talk-30.png]]
- resolver service for configuration items at amazon
	- 1. create resolver groups
	- 2. associate config items to groups
	- 3. when new config items come in, notify contacts
- data model has many-to-many relationships
- want to add all config items to resolver group, or none at all

![[dynamodb-talk-31.png]]
- dynamodb supports transactions
- good for committing changes across items
- good for conditional batch inserts/updates
- bad for maintaining normalized data models

![[dynamodb-talk-34.png]]
- example use case for transact api:
    - cancel configuration across all resolver groups as long as none of them are in progress
    - update contact across resolver groups

![[dynamodb-talk-35.png]]
- reverse lookup to maintain many-many relationship

![[dynamodb-talk-36.png]]
- internal amazon service to get office information
- data has a linear hierarchy
- access patterns: give me everything in country/state/city/office
- composite sort key defines a hierarchy inside one single table
- don't need expensive joins to access the data

![[dynamodb-talk-37.png]]
![[dynamodb-talk-38.png]]
![[dynamodb-talk-39.png]]
![[dynamodb-talk-40.png]]
![[dynamodb-talk-41.png]]
![[dynamodb-talk-42.png]]
![[dynamodb-talk-43.png]]
- fictional service: GetMeThat
- complicated entity model
- has many access patterns (10-12)
- straightforward to model normalised data in relational db
- but all those joins will be expensive when scaling
- nosql is for OLTP at scale
    - "if you're not dealing with big data, look at other technologies"
    - but many things evolve to become a "big data" app
- nosql data model
    - driver status at 5 min incements
        - get driver starts with "gps" (i.e. "gps-05", "gps-10")
        - driver indexed by sector (GSIs) to help with scheduling
    - one query gets data across multiple entities
        - customers, orders, drivers
- we support 12 access patterns with just one table and two GSIs

![[dynamodb-talk-44.png]]
![[dynamodb-talk-45.png]]
![[dynamodb-talk-46.png]]
![[dynamodb-talk-47.png]]
![[dynamodb-talk-48.png]]
- real world example: audible sync service
- lots of downstream and upstream consumers
- 20 access patterns
- can we do it all with one table and five GSIs?
- difficult, start considering relational table approach
- proposed nosql primary table:
    - supports audit trail via version history design pattern
        - current item is always v0
    - two partitions on the table: abook, ebook
    - item inserted into abookacr partition associates it to ebookacr
    - 3 GSIs on the table
        - they don't always contain the same data type
        - because we have many access patterns
    - single tables exist for up to 40 access patterns

![[dynamodb-talk-49.png]]
- cheap data center infrastructure
- (but very complex and locked in)
- shown application deploys for pennies a month
- can autoscale to millions of users if needed
- fail cheaply

![[dynamodb-talk-50.png]]