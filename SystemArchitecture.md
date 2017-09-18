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

For applications that have a higher write load, there is no good way to scale up a relational database. However, this is attempted by sharding. In this scenario, there is one router which points to different database buckets. For example, you could have a bucket for users A-C, another bucket for users D-F, and so on. Instead of sharding alphabetically, you could also shard by location. Facebook shards by location because you can assume that most people talk to people who are near them. Sharding also employs master-slave replication.

If you have a high write load across shards, you have reached the point where you need to stop using a relational database.

Sharding can be spread across many data centers. A data center is basically a giant building with a bunch of servers, where some of those servers are your database machines. When you have data centers in different places, you need to ensure that the data between all data centers is consistent.

Think about your data model. Is there a natural aggregation? How can you denormalize to make the most common queries easy and fast? You need to make important design decisions upfront. Think, how am I usually going to be querying this?

Most startups use a relational database because it is easy to change, as opposed to starting with NoSQL which is significantly harder to change.

If you have a non-normalized database, then every time that you write, you need to update it in all places. Ideally, you are controlling this in the application layer.

### CDN

A CDN is a giant data center that is designed to send big files to users as fast as possible. Cloudfront, for example, has data centers everywhere that are very close to end users. A CDN can serve big files to users very quickly, and is significantly faster than serving the files up from your server.

If your frontend is moving slowly, a CDN will help speed up the process. Typically, your browser is making too many requests. If your JavaScript / CSS is loaded from your web server, that is inefficient and very slow. You want to load what you can from a CDN and minimize every request that is made.

In virtually every production application, the JavaScript gets concatenated together and put into one giant file that you feed to a CDN. You feed another giant CSS file to the CDN as well. This way, the user makes far fewer requests: one to your server for the raw HTML, one for the JavaScript, one for the CSS, and maybe another for the media. Ideally, a CDN will store all content that is not raw HTML.

### Caching

Examples: Redis, memcached

Caching is really important to prevent databases from being smashed. Databases are largely slow, and ideally we would have a cache that is optimized for speed. The cache is generally stored in memory and isn't persistent (similar to RAM).

A cache is simply a Key-Value store. What you store in the cache is the output of common queries or static data. Anything that needs to hit the database goes to the caching layer first. All user data, session data, and data related to the most common users is stored in the cache. This way, when a user logs in, you can validate them without hitting the database.

An example of a common query for Twitter, for example, would be a celebrity's tweets. Celebrity tweets live in the cache because so many people try to read them. An example of an expensive query would be trending hashtags. You would need to read every hashtag of every tweet. ASIDE - Elasticsearch is a specialized database optimized for searching.

If something takes two minutes to compute, you don't want the user to have to wait for two minutes. You should cache the result of the value and then run recomputation in the background, which updates the cache every hour or so.

It is your responsibility to tell the cache when things are invalidated / expired. You might invalidate the celebrity tweets in the cache every time you get a write to a celebrity's tweets.

In addition to using a CDN, caching also makes a website faster.

### Application Server

The difference between Heroku and AWS is that Heroku is a PaaS - Platform as a Service, and AWS is IaaS - Infrastructure as a Service.

PaaS figures everything out for you, does bundle install, gems, etc. It is less configurable and more expensive. Heroku does a lot of magic for you, but you would not use it for a large-scale application because it is too expensive per CPU cycle. Heroku is ~2x more expensive than AWS.

AWS provides you VMs that have a certain amount of memory / CPU, and these VMs can run whatever code you want. You have to configure them and do all the "magic" that Heroku takes care of for you. Big companies typically hire DevOps to make sure all of this stuff works, ensure each piece talks to each other in the proper way, and manage your servers.

Before AWS you would have to buy your own server, bring it to your house and plug it in. That server would then run your app. Netflix runs its own data centers, and doesn't use AWS. Very few companies do this. Most just use AWS.

AWS has data center failover.

You will have many application servers, and each server runs the same code, and talks to the cache + database.

------
Services vs. Monolith

There are two different types of architecture when it comes to the application server component: Service-oriented Architecture (SOA; now known as microservices) and Monolith

Monolith - one giant Rails app that does everything. If something breaks, anywhere, the whole thing breaks.

Microservices - You might have a user auth microservice (which is its own app), a tweets microservice, etc. These two talk to each other on an internal network via API. They are two different Rails apps that talk internally. This way, if the tweets app breaks, the user auth app does not. It enables modularity on the application level. User auth has its own API, tweets has its own API, and no one can talk to user auth except tweets.

Example of service breakdown: Uber

Routing service

Dispatch service

Payment service

Reviewing service

User auth service

If you work on routing, you don't need to know how dispatch or payment processing works under the hood. You just need to know how to call the API.

There is a failover for each service. Microservices is similar to the idea of OOP, namely the separation of concerns.

Pros / Cons

Failures can be isolated more easily with SOA. It is extremely difficult to do this with a monolith architecture.

Easy to divide among teams. One team can keep their codebase small and understandable.

Microservices can be written in various languages and they just talk over an API. Monolith is written in only one language.

With microservices, it is easy to do small refactors.

As a downside, there is a bit of overhead in having the services talk to one another.

### Client vs Server

TCP – ex: this data must get to the user. TCP requests to talk to the server; the server obliges. They open a connection. Once the connection is open, I send packets to the server. Server confirms it received the packets, and the process repeats. The packets are all numbered, and the server makes sure it received the packets in the right order.

TCP has a lot of overhead because of the acknowledgement, verification of things going in the right order

HTTP is built on UDP. HTTP uses TCP. When you ask a server for  a get request, you first send a message to the server to talk. The server asks what you want. You say I want this page. Server says I got your packet that says you want this page, here’s the page. You say I got it, server says I’m done, connection closed.

UDP AKA best effort. I send you a packet and you either get it or you don't with no guarantee about ordering. UDP is faster, but TCP is more secure, and you know how things are ordered.

Example: every 5 seconds an Uber driver sends up location details to the Uber sever. Uber records info on its server, then sends you the most updated info. This needs to be really fast. You should use UDP.

You would use TCP if you want to ensure that a packet does not get dropped. HTTP assumes you need TCP.

Is it ok to let the client and server become inconsistent? Possibly yes, if an "instant" feeling is desired. On the contrary, Google Docs cannot allow inconsistency because each user's document is the same.

### Load Balancer

Balances load. You do not want all your users going to one server; you want to equally distribute the load amongst all of your servers. You have multiple load balancers in place to ensure failover, and you also put a load balancer between the caching layer and your databases.
