Design A Key Value Store - non-relational database - keys must be unique and the value can be accessed through the key

Requirements - supports put(key, value) and get(key) operations

Step1: Understand the problem and Establish the design scope
1. The size of the key-value pair is small: less than 10KB
2. Ability to store big data
3. High Availability: The system responds quickly, even during failures
4. High Scalability: The system can be scaled to support large dataset
5. Automatic scaling: The addition/deletion of server should be automatic depending on traffic
6. Tunable consistency
7. Low latency

Single server key-value store - store key-value pairs in a hash table which keeps everything in memory - fitting everything in memory may be impossible due to space constraint - optimatizations - data compression and Store only frequently accessed data in memory and rest in disk
But a single server can reach its limit very quickly

*** Distributed Key Value Store - key-value pairs distributed across many servers

CAP THEOREM - impossible for a distributed system to simultaneously provide more than two of the three guarantees: conssitency, availability and partition tolerance
Consistency - all clients see the same data no matter what node they connect to
Availability - any client connecting gets a response even if some nodes are down
Partition Tolerance - a partition indicates a communication break between 2 node - tolerance means the system continues to operate despite this

key-values stores are classified based on which 2 characteristics they provide
CP - supports consistency and partition tolerance
AP - supports availability and partition tolerance
CA - supports consistency and availability - since network failure is unavoidable, distributed system most tolerate partition tolerance - thus CA cannot exist in real world applications

Ideal situation - network partition never occurs - both consistency and avialability are achieved

Real world distributed systems - choose between consistency and availability if one server goes down
Consistency over availability - block all writes on up servers to avoid data inconsistency between the servers making the system unavailable - important for banks to display most up to date info - if inconsistency occurs due to network partition - bank system returns an error before inconsistency is resolved
Availability over consistency - system keeps acepting reads even though it might return stale data - data will be synced to down server once the network partitiion is resolved

VERY VERY IMPORTANT - Partition means that the servers cannot communicate with each other, it DOES NOT mean the servers are down or anything - if the server is down both CP and AP will show unavailbel
CP - has something like a quorum leader - if a partitiion has majority it handles the reads and write - so a separated partitoin does not have a majority wil refuse to read or write - when the partiton is resolved as there are is no history for this partition the history of the leader partition is replayed on this. - that's why banking 
AP - the separated partition will continue to respond to both read and write requests - when the partition resolves then both the histories can be merged
We can choose what we want

System Components - 
1. Data Partition
2. Data Replication
3. Consistency
4. Inconsistency Resolution
5. Handling Failures
6. System Architecture
7. Write Path
8. Read Path 

1. Data Partition - infeasible to fit complete data set in a single server - smaller partitions and store them on multiple servers - 2 challennges - 
1. Dsitribute data across multiple servers evenly
2. Minimize data movement when nodes are added or removed
Consisitent Hashing is used to achieve this

2. Data replication - for high availability and reliability data must be replicated async over N servers - in the consistent hashing ring the data can be replicated over the N consecutive servers in clockwise direction - with virtual nodes in between we can end up choosing the same server - so to avoid this we choose N unique servers - for better relaibility replicas are placed in distinct data centers and data centers are connected using high-speed networks

3. Consistency - data must be synchronised across replicas - quorum consensus can guarantee consistency for both read and write operations
N = number of replicass
W = write quorum of size W - for a write operation to be considered successful it must be acknowledged by W replicas
R = read quorum of size R - for a read operation to be considered successful it must be acknowledged by R replicas

W = 1 does not mean write happened on only 1 server - it means atleast 1 server acknowledged the write
if W = 1 or R = 1, operation is quick becuase only 1 replica needs to acknowledge the operation
So, W, R > 1 ensure better consistency - but slower as the coordinator has the wait for the response from the slowest replica

W + R > N ensures strong consistency - because atleast 1 overlapping replica with latest data - as if W and R are disjoint sets - reads and write happening on diff replicas then R + W will be N - if we want >N then there must be overlap

Possible setups:
1. R = 1, W = N - fast reads
2. R = N, W = 1 - fast writes
3. W + R > N - strong consistency guaranteed - because the overlapping one will have the most recent timestamp used for inconsistency resolution - because it will have acknowledged the most recent write
4. W + R <= N - strong consistency not guaranteed

Consistency models - 
Strong - any read operation returns a value corresponding to the result of most recent updated wrtite item - client never sees out-of-date data
Weak - may not see most updated values
Eventual - specific form of weak - given enough time updates are all propogated and all replicas are consistent

Strong is usually achieved by forcing a replica to not accept read/write until every replica has agreed (CP) - not ideal for highly available systems (AP)
Dynamo and Cassandra have eventual - recommended for our key-value store - allows inconsistent values to enter the system and force the client to read the values to reconcile (AP)

4: Inconsistency resolution: versioning - versioning and vector clocks are used to solve inconsistency problems across replicas (This is only applicable in AP systems as CP systems will never have versioning issues as they have a linear history at all times)

vector clock - [server, version] pair associated with a data item, it can be checked if one version precedes, succeeds or in conflict with others
represented by - D([S1, v1], [S2, v2], ... , [Sn, vn]) where D is the data, v1 is the version counter and s1 is a server number - if data item d is written to a server it must do the following - 
1. Increment vi if [Si, vi] exists 
2. Otherwise, create a new entry [Si, 1]

Scenario - 
1. A client writes a data item D1 to the system, and the write is handled by server Sx, which now has the vector clock D1[(Sx, 1)].
2. Another client reads the latest D1, updates it to D2 and writes it back. D2 decends from D1 so it overwrites D1. Assume tehe write is handled by same server Sx, which now has the vector clock D2([Sx, 2]).
3. Another client reads the latest D2, updates it to D3, andw writes it back. Assume the write is handled by server Sy which now has the vector clock D3([Sx, 2], [Sy, 1]).
4. Another client reads the latest D2, updates it to D4, and writes it back. Assume the write is handled by server Sz, which now has D4([Sx, 2], [Sz, 1])
5. When another client reads D3 and D4, it discovers a conflict, which is caused by D2 being modified by both Sy and Sz. The conflict is resolved by the client and the updated data is sent to the server. Assume the write is handled by Sx, which now has D5([Sx, 3], [Sy, 1], [Sz, 1]).

Using vector clocks, it is easy to tell version X is an ancestor of version Y if each participant(server) in the vector clock of Y is greater than or equal to the ones in X

Similarly, X is a sibling of Y (conflict exists) if there is any participant(server) in the vector clock of Y that is less than in X

Vector clocks - can resolve conflicts but 2 major downsides -
1. Adds complexity because client needs to implement the conflict resolution logic
2. server:version pairs grow rapidly - can set a threshold limit, oldest pairs are removed - but leads to inefficiencies in reconciliation because descendent relationship cannot be determined accurately

5:Handling failures - failures are inevitable

Failure detection - requires atleast two independent sources of information to mark a server down - all-to-all multicasting can be used, but is inefficient when many servers are in the system
Gossip Protocol - 
1. Each node maintains a node membership list, which contains memberIDs and heartbeat counters
2. Each node periodically increments its heartbeat counter
3. Each node periodically sends heartbeats to set of random nodes, which in turn propogate to another set of nodes
4. Once nodes receive heartbeats, membership info is updated with latest info
5. If heartbeat is not incresed for predefined periods, the member is considered offline - once another node confirms it is down - it is propogated that that server is down

Handling temporary failures - after failures are detected, certain mechanisms to ensure availability
strict quorum - read and write blocked (CP)
sloppy quorum - used to improve availability (AP) - system chooses first W healthy servers for writes and R for reads - ignoring offline servers
If a server is unavailble due to network or server failures - another server will processs requests temporarily - when the server comes up, changes will be pushed back to achieve consistency - called as hinted handoff

Hinted handoff is used to handle temporary failures. To handle permanent failures, anti-entropy protocols to keep data in sync - involves comparing each piece of data on replicas and updating each replica to the newest version - a merkel tree is used for inconsistency detection and minimizing the amount of data transferred
Hash tree or merkel tree is a tree in which every non-leaf node is labelled with the hash of the labels or values of its child nodes. Hash trees allow efficient and secure verification of contents of large data structures

Steps:
1. Divide key space into buckets. A bucket is used as the root level node to maintain a limited depth of the tree
2. Once buckets are created, hash each key in a bucket using a uniform hashing method
3. Create a single hash node per bucket
4. Build tree upwards till root by calculating the hashes of children

If root hashes match, both servers have same data. if root hash disagrees, then left child hashes are compared followed by the right hashes.
Merkel trees, the amount of data to be synchronised is proportional to the differences between the 2 replicas and not the amount of data they contain

Handling data center outage - replicate data across multiple data centers

6: System Architecture Diagram - main features
1. Clients communicate with a key-value store using simple APIs: get(key) and put(key, value)
2. A coordinator is a node that acts as a proxy between the client and the key-value store
3. Nodes are distributed on a ring using consistent hashing
4. The system is completely decentralized so adding and moving nodes can be automatic
5. Data is replicated at multiple nodes
6. There is no single point of failure as every node has the same set of responsibilities.

7: Write Path - write request is directed to a specific node
1. Write request is persisted on a commit log file
2. Data is saved in the memory cache
3. When the memory cache is full, data is flushed to SSTable on disk - sorted string table is a sorted list of <key, value> pairs

8: Read Path - read request directed to a specific node - checks if data is in memory, if yes returns to the user.

If data is not in memory, retrieved from disk - bloom filter algorithm is used to efficiently search SSTable for the key-values pair
1. System checks if data is in memory, if yes returns
2. If not, system checks the bloom filter
3. Bloom filter is used to figure out which SSTables might contain the key
4. SSTables return the result of the data set 
5. Returned to the client

Summary
Goal/Problems                               Techniques
Ability to store big data                   Use consistent hashing to spread the load across servers
High availability reads                     Data replication, multi-data center setup
High availability writes                    Versioning and conflict resolution with vector clocks
Dataset partition                           Consistent Hashing
Incremental Scalability                     Consistent Hashing
Heterogeneity                               Consistent Hashing
Tunable consistency                         Quorum consensus
Handling temporary failures                 Sloppy quorum and hinted handoff
Handling permanent failures                 Merkel tree
Handling data center outage                 Cross-data center replication
