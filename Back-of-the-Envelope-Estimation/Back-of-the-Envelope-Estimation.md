2. Back of the envelope Estimation

Problem Statement: Created using combination of thought experiments and common performance numbers to get a good feel for which designs will meet the reqiurements.

-- Power of Two
Power   Approximate Value   Fullname    Short name
10      1 Thousand          1 Kilobyte  1KB
20      1 Million           1 Megabyte  1MB
30      1 Billion           1 Gigabyte  1GB
40      1 Trillion          1 Terabyte  1TB
50      1 Quadrillion       1 Petabyte  1PB
Dont make the mistake - 2^10 = 1024 bytes, not 10^3

-- Latency Numbers to know - fastness or slowness of operations
Operation Name                              Time
L1 Cache reference                          0.5ns
Branch mispredict                           5 ns
L2 Cache reference                          7ns
Mutex lock/unlock                           100ns
Main memory reference                       100ns
Compress 1K bytes with Zippy                10,000ns = 10us
Send 2K bytes over 1 Gbps network           20,000ns = 20us
Read 1MB sequentially from memory           250,000ns = 250us
Round trip within the same datacenter       500,000ns = 500us
Disk seek                                   1,000,000ns = 10ms
Read 1MB sequentially from the network      1,000,000ns = 10ms
Read 1MB sequentially from the disk         30,000,000ns = 30ms
Send packet CA -> Netherlands -> CA         150,000,000ns = 150ms

Conclusions - 
Memory is fast but disk is slow
Avoid disk seeks if possible
Simple Compression algorithms are fast
Compress data before sending over the internet if possible
Data centers are usually in different regions, and it takes time to send data between them

-- Availability Numbers - system continuously operational for a desirably long period of time, 100% means 0 downtime, most between 99% and 100%
Availability %                  Downtime per day                    Downtime per year
99%                             14.40 minutes                       3.65 days
99.9%                           1.44 minutes                        8.77 hours
99.99%                          8.64 seconds                        52.60 minutes
99.999%                         864.00 milliseconds                 5.26 minutes
99.9999%                        86.40 milliseconds                  31.56 seconds

Estimate Twitter QPS and storage requirements
Assumptions:
300 million monthly active users
50% of users use Twitter daily
Users post 2 tweets per day on average
10% of tweets contain media
Data is stored for 5 years

Estimations:
Query per second (QPS) estimate:
Daily active users (DAU) = 300 Million * 50% = 1500 million
Tweets QPS = 150 Million * 2 tweets / 24 hour / 3600 seconds = 3500
Peek QPS = 2*QPS = 7000

Media storage
Average tweet size:
tweet_id 64 = 64 bytes
text        = 140 bytes
media       = 1 MB

Media storage: 150 Million * 2 * 10% * 1MB = 30TB per day
5-year media storage: 30TB * 365 * 5 = 55PB

VERY IMPORTANT - Commonly asked QPS, peak QPS, storage, cache, number of servers, etc
