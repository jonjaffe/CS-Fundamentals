# Architecture

How big can this thing get?


## Components
Load Balancer, Application Servers, Caching Layer, CDN, Databases

## Cycle
Client makes request to DNS server. With the IP address back, a request then gets sent to the load balancer. Load balancer then connects with all application servers. The servers then connect with a caching layer and CDN. Finally, the cache connects with the database(s).

---

### Database
1. Relational (Postgres)
2. Non-relational / NoSQL
+ Document-based (XML/JSON)
+ Key-Value store

Relational databases are comprised of many tables with relationships that are expressed through joins.

In a relational database, you have a schema for each table with constraints (not null, etc).

Relational databases are very bad at scaling because of joins. Joins require you to look at giant tables and smash them all together. This is very costly. Joins can be improved with indexes, but they are still not very good.

Denormalizing a database can make it faster. Normalization is the idea of not duplicating data, and this is a core tenet of relational databases: the data should live in one place because if it lives in two, things can easily get out of sync. Denormalization is done very often to scale when the tables get so big that joins become problematic.

An example of denormalizing is that, instead of joining a users table with a followers table, the users table has a followers key which contains an array of user ids. Any time that someone follows someone else, I now have to update in 2 places: the users table and the followers table.

Non-relational databases do not have an enforced schema. MongoDB, for example, is a giant hashmap with keys and values. If you want to find something in a MongoDB database, you use the key to retrieve the JSON. Non-relational databases are significantly faster than relational databases. MongoDB still gives you the power to do indexes and joins.

Similarly, a Key-Value store (like Redis) is a giant hashmap. Instead of being document-based and with more power/capabilities like MongoDB, a Key-Value store is simply that. You can store sets, queues, etc.

---

A distributed database is not just one database, it is comprised of many databases. These databases can be located all over the world. Every read+write doesn't just talk to one node; it talks to multiple nodes. A distributed database allows failures. In the case of a failure, other nodes will pick up the slack.

An example of a distributed database is Cassandra. Cassandra is very complex, and requires a significant time investment. Using Cassandra will slow you down until the point that it is actually necessary. 1MM users is typically where it makes sense to switch from Postgres/MongoDB to something like Cassandra.

# Failover (such an important concept in architecture it gets an H1)

The idea of a failover is knowing that, inevitably things will break in your system, and you need to have safeguards to protect against that. When things break, a website should not go down. You want points of redundancy at every step to ensure that you do not have a single point of failure.

### How to scale a relational database
One database is bad.

Leader-follower implementation. Follower becomes the leader if leader fails. This is also known as master-slave replication.

Leader-follower is done for applications that have a higher read load. The leader has a bunch of followers. When the leader gets a request, it bumps the request down to the followers that are designed for reading. For followers, keeping updated with the leader is easy. If you want to write something to the database, do so via the leader. This allows consistency. If followers are allowed to write, how do you keep everything in sync? You would not be able to ensure that followers don't have conflicting writes. So, writing gets done by the leader, and all reads are processed by the followers.

A clear line of succession is needed. In the case of a leader failure, you need to have another leader waiting or a plan to promote one of your followers to be the leader. In this scenario, that follower moves up and now all of the other followers follow this new leader.
