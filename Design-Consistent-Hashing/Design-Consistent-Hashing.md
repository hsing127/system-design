The objective is to distribute the data/requests efficiently across servers.

The rehashing problem - hashing can be done with serverIndex = hash(key) % N, where N is the number of servers - works perfectly if the number of servers is fixed
The problem arises when the servers are added or removed - then we get cache misses as the N changes consistently - all the keys need to be remapped everytime any server is added or removed from the pool

Consistent Hashing - hash table is re-sized and only k/n keys need to be remapped on average, where k is the number of keys and n is the number of servers.

Hash Space and Hash ring - x1, x2 ... xn servers arranged like a ring

Hash servers - we map the servers on the hash ring based on server IP or name

Hash keys - the hash keys are different from rehashing - no modular operation - the keys are also hashed onto the ring
Now, to determine which key is mapped to which server we start from the key and go to the first server in clockwise direction - key0 stored on server0 and so on.

Add a server - If a server is added between 2 existing servers - we remap the keys from server1 till it encounters the new server to the new server - other keys are not distributed

Remove a server - we start from the removed server - go anticlockwise and remap all the keys to the next server till we encounter existing server

Two basic steps in this algorithm - 
1. Map servers and keys on to the ring is a uniformly distributed hash functions
2. To find which key is stored in which server - go clockwise from the key position until the first server on the ring is found

Two problems - 
1. Impossible to keep the same size of partitions on the ring for all servers considering a server can be added or removed - It is possible that the size of the partitions on the ring is fairly small or fairly large
2. It is possible to have a non-uniform key distribution on the ring

Virtual nodes or replicas are used to solve these problems - refers to a real node, and each server is represented by multiple virtual nodes on the ring - all servers have multiple virtual nodes - with virtual nodes each server is responsible for multiple partitions - to find which server the key is stored on go clockwise find the first virtual node connected on the ring

As the number of virtual nodes is increased, the distibution becomes more balanced - but more space is needed to maintain data about the virtual nodes - this is a tradeoff and we can tune the number of virtual nodes to fit our system requirements

Benefits of consistent hashing - 
1. Minimized keys are reditributed when servers are added or removed
2. It is easy to scale horizontally because data are more evenly distributed
3. Mitigate hotspot problem - excessive access to a specific shard can cause server overload

Consistent Hashing is widely used in the real world - 
1. Partitioning component of Amazon's Dynamo database
2. Data partitioning across the cluster in Apache cassandra
3. Discord chat application
4. Akamai content delivery network
5. Maglev network load balancer
