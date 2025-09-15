4. Design a Rate Limiter - limits the number of client requests allowed to be sent over a specified period, all excess calls are blocked.
eg - no more than 2 posts per second per user, max 10 accounts per IP addr, claim rewards no more than 5 times per week from the same device

Advantages - 
prevent Ddos attacks eg - Twitter limits number of tweets to 300 per 3 hours, Google APIs 300 per user per 60 seconds of read requests
Reduce cost - fewer servers and allocating more resources to high priority APIs, extremely important for companies that use third party APIs as you are charged per-call basis for external APIs - rate limiting is essential to save costs
prevent servers from getting overloaded - to filter out excess requests caused by bots or users' misbehaviours

Step1: Understand the design and establish the scope

Questions asked for rate limiter - 
1. Client Side or server side?
2. Rate limiting based on IP, user ID or other properties?
3. Scale of the system?
4. Will the system work in a distributed environment?
5. Separate service or application code itself?
5. Inform the users if they are throttled?

Problem Statement: 
1. Accurately limit excessive requests
2. Low latency - should not slow down the HTTP response time
3. Use as little memory as possibe;
4. Distributed rate limiting - shared across multiple servers or processes
5. Exception handling - show users when their requests are limited
6. High fault tolerance - if problem with the rate limiter, the system does not go offline

Step2: Propose a high level design and get a buy-in 

1. Where to put the rate limiter?
- Client-side implementation - unreliable place as client requests can be easily forged, also we might not have control over the client implementation
- Server-side implementation -Rate limiter along with the server code on the server itself
- Middleware implementation - requests forwareded the middleware first, if possible forward to the server, else middleware throttles the requests and sends back HTTP 429 - too many requests

In the cloud architecture - rate limiting is implemented within a scomponent called a API gateway - supports rate limiting, SSL termination, authentication, IP whitelisting, servicing static component, etc

Rate limiter on the server or the gateway? - depends:
1. Current tech stack, programming language, caching service, etc
2. Rate limiting algorithm - on server side: full control over the algorithm, third party api gateway: limited choices
3. Already have a API gateway? - can add rate limiting to it easily
4. Building own rate limiting service takes time - not enough team? - use a commercial API gateway

Algorithms for Rate Limiting:

1. Token Bucket - container that has s prefefined capacity - tokens are put in the bucket at preset rates periodically - once the bucket is full, no more tokens are added,
Each request consumes a token, when a request arrives we check if there are enough tokens
- if enough tokens - we take one token out for each request and the request goes through
- not enough tokens - the request is dropped

This algorithms takes 2 params:
1. Bucket Size: the maximum number of tokens allowed in the bucket
2. Refill rate: number of tokens put into the bucket every seconds(interval)

How many buckets? - 
1. Usually 1 bucket per API endpoint
2. If need to throttle based on IP addr, need another bucket
3. If the system as a whole allows 10000 requests per second, global bucket shared by all APIs

Pros:
1. Easy to implement
2. Memory Efficient
3. Allows a burst of traffic for short periods. Requests can go through as long there are tokens left

Cons:
1. Two parms bucket size and refill rate - challenging to tune them properly

2. Leaking Bucket Algorithm - similar to token bucket but the requests are processed at a fixed rate usually with a FIFO queues
When request arrives, check if the queue is full; if yes drop the request; else add to queue

Leaking bucket takes 2 params:
1. Bucket size - equal to the queue size; queue holds the requests to be processed
2. Outflow rate - how many requests can be processed at a fixes rate, usually in seconds

Pros:
1. Memory efficient - given the limited queue size
2. Fixed rate so suitable for uses cases that a stable outflow rate is needed

Cons:
1. Burst of traffic fills up the queue with old requests, new requests rate limited
2. 2 params - not easy to tune them properly

3. Fixed window counter algorithm
Divides the timeline into fixed sized windows and a counter for each window
Each request increments the counter by 1
Once the counter reaches the pre-defined threshold, new requests are dropped until new window starts

A major problem is that a burst of traffic at the edges of time windows could cause more requests than allowed quota to go through

Pros:
Memory efficient
Easy to understand
Resetting available quota at the end of a unit time window fits certain use cases

Cons:
Spike in traffic would allow more requests than the allowed quota to go through

4. Sliding Window log algorithm
With fixed window algorithm the issue is it allows more requests at the edge of the windows,
Sliding window does
1. Keeps track of recent timestamps, timestamp data is usually kept in cache, such as sorted sets of Redis
2. When a new request comes in, remove all the outdated timestamps - the timestamps older than the start of the current time window
3. Add timestamp to the request to the log
4. If the log size is same or lower than the allowed count, a request is accepted, else rejected

Pros:
Rate limiting implemented by this algorithm is very accurate. In any rolling window, requests will never exceed the rate limit

Cons:
This algorithm consumes a lot of memory - even if request is rejected, its timestamp still might be stored in the memory

5. Sliding Window Counter Algorithm - hybrid approach that combines fixed window counter and sliding window log algorithms

Number of requests in the rolling window = Requests in current window + requests in the previous window * overlap percentage of the rolling window and the previous window
- the number can be rounded up or rounded down

Pros:
It smoothes out traffic because rate is based on the average rate of the previous window, memory efficient

Cons:
Works only for not-s-strict look back window - approximation of the actual rate because it assumes requests in the previous window are evenly distributed - 0.003% of requests are wrongly allowed or rate limited using this out of 400 million requests

High Level Architecture
We need to store the counters - disk are too slow, in-memory cache is chosen as it is fast and supports time-based expiration strategy - Redis is commonly used - INCR (increase counter by 1) and EXPIRE(sets a timeout for the counter)

So, now - 
Client sends a request to a rate limiting middleware
Rate limiting middleware fetches the counter from the corresponding bucket in Redis and checks if limit is reached or not -
if yes, request is rejected
else, request is sent to API servers and the Redis counter is incremented by 1  

Step3: Design deep dive
How are rate limiting rules created? Where are the rules stored?
How to handle requests that are rate limited?

Exceeding Rate Limit - return http response 429 (too many requests) to the client - we may enqueue rate limited requests to process later like if some orders are rate limited we may store them to process later

Rate Limiter headers - to let the user know they are being throttled
X-Ratelimit-Remaining - remaining number of requests allowed within the window
X-Ratelimit-Limit - how many calls the client can make per window
X-Ratelimit-Retry-After - number of seconds to wait until you can make a request again without being throttled

Detailed Design - 
Rules are stored on the disk. Workers frequently pull rules from the disk and store them in the cache
When a client sends a request to the server, the request is sent to the rate limiter middleware first
Rate limiter middleware loads rules from the cache. It fetches the count and last request timestamp from the Redis cache.
- if the request is not rate limited, it is forwarded to the API servers
- if yes, returns 429 too many requests - request is either dropped or forwarded to the queue

Rate limiter in a distributed environment
-  2 issues - race conditions and synchronisation issues

Race Conditions - if two requests concurrently read the value from the cache, they update the wrong value
- Can be solved using locks - but locks are slow
- Can be solved using Lua script and sorted sets data structures

Synchronisation Issue - one limiter is not enough to serve millions of users, have to have multiple
Now its possible that the first rate limiter does not have the data about the previous request it handled as the web tier is stateless - this can be solved by
- sticky sessions - not ideal, scalable or flexible
- Contralized store like Redis

Performance Optimization
Multi data center setup - high latency for users far away, cloud providers build many edge server locations around the world - traffic automatically routed to the nearest edge server
Synchronise data with an eventual consistency model

Monitoring - gather analytics data to check if the rate limiter is effective - algortihm and rules
- rules can be relaxed if too many requests are dropped.
If rate limiter becomes ineffective when there is a sudden burst of traffic - token bucket can be used

Step 4: Wrap Up

Algorithms for Rate Limiting:
1. Token Bucket
2. Leaking bucket
3. Fixed Window
4. Sliding window log
5. Sliding window counter

Additional talking points:
- Hard vs Soft rate limiting
    - Hard - The number of requests cannot exceed the threshold
    - Soft - Requests can exceed for a short period of time

    This was Layer 7 rate limiting - can be implemented on all levels of the OSI model - Application (7), Presentation (6), Session (5), Transport (4), Network (3), Data Link (2), Physical (1)
    To avoid being rate limited - 
    - Use client cache to avoid making frequent API calls
    - Understand the limit and avoid sending too many requests in a short time frame
    - Include code to catch exceptions or errors so your client can gracefully recover from exceptions
    - Add sufficient back off time to retry logic
