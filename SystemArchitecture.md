# Architecture

How big can this thing get?


## Components
Load Balancer, Application Servers, Caching Layer, CDN, Databases

## Cycle
Client makes request to DNS server. With the IP address back, a request then gets sent to the load balancer. Load balancer then connects with all application servers. The servers then connect with a caching layer and CDN. Finally, the cache connects with the database(s).

---

### Database
+ Relational (Postgres)
+ Non-relational / NoSQL (MongoDB)

Relational databases are comprised of many tables with relationships that are expressed through joins.
