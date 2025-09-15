1. Scaling app to a Million usersr

Problem Statement - Build a system that supports a single user and gradually scale it up to serve millions of users.

Steps:
-- Basic Single Server setup -- 
user --> browser --> dns --> gets IP --> calls the server --> returns JSON or html page --> browser shows to the user

-- Databases 
1. Most commonly used - RDBMS (Relational Database Management Sytstem) - can do joins between tables
2. Non-relational Database systems - 4 categories - key-values stores, graph stores, column stores, and document stores; Joins not supported
Right choice if
a. Your application needs super low latency
b. Data are unstructured or no relational data
c. Only need to serialize or deserialize data
d. Need to store a massive amount of data

-- Vertical Scaling - "Scaling up" - Has a hard limit
-- Horizontal Scaling - "Scaling out"

-- Load Balancer -Web servers are unreachable by the clients. For better security, private IPs are used for communication between the servers which is used by the load balancer.

Notes:Now we have solved the issue of failover as load balancer can handle traffic and can handle more servers being added to the server pool
Data tier right now has 1 db and does not support failover and redundancy.

-- Data Replication - done with usually master/slave relationship between databases.
A master db only supports write operations. A slave db gets the copies of the data from the master db and only supports read ops. All data modifying commands like insert, delte or update must be sent to master db. Most apps require much higher ratio of reads to writes; thus the number of slaves db is usually higher than master dbs. 
Advantages:
1. Better Performance - allow more queries to be executed in parallel - reads across slaves and writes across masters.
2. Reliability - If some dbs are destroyed, data is preserved and it is replicated across multiple locations
3. High Availability - Replicating allows website to remain online even if one db is offline.

What if one db goes down - 
1. if slave goes down and it is the only slave - read ops are redirected to the master db. After the isssue if resolved a new slave can replace the old.
2. if master goes down - a slave db is promoted to master and will handle all the writes. A new slave is spun up in place of this slave. Promoting slaves if challenging as it might be out of date - data will need to be updated using recovery scripts. Also multi masters and circular replication can help[SEE THEM]

Deisgn till now - 
1. A user gets the load balancer IP using a dns service.
2. The user connects to the load balancer using this IP.
3. HTTP request is routed to either Server 1 or Server 2.
4. A web server reads the data from a slave DB.
5. A web server routes any data-modifying ops to the master DB - write, update, delete

Notes: Now we can improve the response time using cache layer and CDN for static content(JS/CSS/Image/Video files)

-- Cache - Temporary storage area that stores the result of expensive responses or frequently accessed data in memory.
Cache Tier - Data store layer much faster than the db. Benefits - better system perf, reduce db workloads, scale cache independently.
After recieving request, web server checks the cache first, if yes sends back to the client; else queries the db, stores the response in the cache and then send it back the client - this is called read through cache.
Considerations:
1. Data is read frequently but modified infrequently - cache is stored in volatile memory so not ideal for persisting data; if cache server restarts all the data is lost.
2. Expiration Policy - not too long (data can become stale), also not too short(reload data too frequently from db).
3. Consistency - Keeping the data store and the cache in sync; inconsistency can happen because data modifying ops on the data store and the cache are not in a single operation; hard to maintain cache consistency across multiple regions.(See scaling memcache at Facebook)
4. Mitigating Failures - cache server represents a single point of failure(SPOF) - if it fails entire system fails; multiple cache servers across diff data centres are recommended - also overprovision required memory by certain percentages - provides a buffer as the memory usage increases.
5. Eviction Policy - Once cache is full, any requests to add items to cache causes existing items to ber removed; LRU is most popular, LFU or FIFO can ber used.

-- Content Delivery Network (CDN) - network of geographically dispersed servers used to deliver static content like images, videos, CSS, JS, etc
User visits a website -> CDN server closest to the user delivers the static content
1. User A tries to get image.png bt using the image URL - provided by the CDN provider - eg. https://mysite.cloudfront.net/logo.jpg, https://mysite.akamai.com/image-manager/img/logo.jpg
2. If the CDN server does not have the image in cache, the CDN requests the image from the origin - web server or S3 bucket.
3. Origin returns image to the CDN, including optional HTTP header TTL - describing how long the image is cached.
4. CDN caches the image and retunrs to the user A - image remains in the cache till TTL
Considerations using a CDN:
1. Run by third party providers - charged for data in and out of CDN, caching infrequently used assets provide no benefits.
2. Seting appropriate cache expiry
3. CDN fallback - how woebsite copes with CDN failure - clients should be able to detect outage and request from source.
4. Invalidating files - before expiry - using APIs provided by CDN vendors, object versioning

-- Stateless webtier  - moving the session data out of the web tier to a persistent storage like a relatioanl DB or NoSQL
For Stateful architecture can use sticky sessions in most load balancers - adds overhead, adding/removing servers is more difficult, challenging to handle server failures.
In Stateless - fetch state data from a shared data store - simpler, more robust, scalable
NoSQL is chosen as it is easier to scale - autoscaling can be achieved much easier as state data is out of the servers.

-- Data Centers - users are geoDNS-routed known as geo-routed to the closest data center. geoDNS is a DNS serveice which allows domain names to be resolved to IP addresses based on the location of the user.
Technical challenges:
1. Traffic redirection - redirect traffic correctly, GeoDNS can be used to direct traffic based on user location
2. Data Synchronization - users from diff regions can use diff local dbs or caches. in failover cases, traffic might be routed to a data center where the data is unavailable. a common strategy is to replicate data across data centers.
3. Test and deployment - with multi data center setup - imp tp test your website at diff locations - automated deployment tools are vital to keep services consistent through all data centers

-- Message Queue - supports async communication - serves as a buffer and distributes async requests. Decoupling helps building scalable and reliable application. When the sze of the queue becomes large - more workers can be added to reduce the processing time. If queue is empty mostly - workers can be reduced.

-- Logging - Monitoring error logs is necessary - can be done per server level or aggragate them to a centralized service for easy search and viewing

-- Metrics - Hostlevel metrics - CPU, Memory, Disk I/O, etc, Aggregated level - performance of entire db tier, cache tier, etc, Key business - DAU, retention, revenue, etc

-- Automation - CI, automating build, test, and deploy

NOTES: Time to scale the data tier

-- Database Scaling
1. Vertical Scaling - adding more power to existing machine - greater risk of single point of failures - powerful servers are more expensive
2. Horizontal Scaling - called as Sharding - practice of adding more servers
Separates larger dbs into smaller, more easily managed parts called shards. Each shard shares the same schema - actual data on each shard is unique to the shard.
eg. sharding db based on userid - hash function to find the corresponding shard. Most important is the choice of the sharding key (partition key) - to evenly distribute the data.
Challenges:
1. Resharding data - 1. single shard cannot hold more data due to rapid growth, 2. Certain shard might experience shard exhaustion faster due to uneven data distribution. - when this happens - it requires upadting the sharding function and moving the data around - Consistent Hashing solves this problem
2. Celebrity Problem - Hotspot key problem - excessive access to a specific shard could cause server overload - shard overwhelmed by read ops.
3. Join and de-normalization - Once a db has been sharded across multiple servers, it is harder to perform join operations across database shards - workaround is to denormalize the database so that queries can be performed in a single table.

-- Nonrelational functionalities are moved to a NoSQL data store to reduce the db load further.

SUMMARY:
1. Keep webtier stateless
2. Build redundancy at every tier
3. Cache data as much as you can.
4. Support multiple data centers
5. Host Static assets in CDN
6. Scale your data tier by sharding 
7. Split tiers into individual services
8. Monitor your systerm and use automation tools
