3. Framework For System Design Interviews

4 step process for effective system design interviews

Step1: Understand the problem and establish the design scope
Ask questions to clarify requirements and assumptions - 
1. What specific features are we going to build
2. How many users does the product have
3. How fast does the company anticipate to scale up? What are the anticipated sales in 3 months, 6months and a year?
4. What is the company's technology stack? What existing services you might leverage to simplify the design

Step2: Propose high level design and get a buy-in
1. Initial blueprint of a design. Ask for feedback
2. Draw box diagrams - clients(mobile/web), APIs, servers, data stores, caches, CDNs, message queues, etc
3. Back of the envelope estimation - think out loud
Go through concrete use cases, discover edge cases,
Including API endpoints and db schema depends -  if the problem is too big - don't do it, else do it 

Step3: Design deep dive
Identify and prioritze components in the architecture
Time management is essential

Step4: Wrap up
1. Identify system bottlenecks and discuss potential improvements - never say design is perfect
2. Give a recap of your design
3. Error cases
4. Operation issues - how to monitor metrics and error logs? How to roll out the system?
5. Handle next scale curve - from 1 Million to 10 Million DAU

Dos
1. Ask for clarification - never assume assumption is correct
2. Understand the requirements of the problem
3. No right or the best answer - solution using the requirements
4. Communicate with the interviewer
5. Suggest multiple approaches if possible
6. Once agreed on a blueprint go into details of the components - critical components first
7. Bounce ideas of the interviewer
8. Never give up

Don'ts
1. Unprepared for typical interview questions
2. Jump to a solution without clarifying the requirements
3. Go into too much detail on a single component - give high level design first
4. If stuck - ask for hints
5. Don't think in silence
6. Not done after the design - not done until the interviewer says you're done. Ask for feedback early and often

Time allocation on each step
Step1 Understand the problem and establish the design scope- 3-10 minutes
Step2 Propose a high level design and get a buy-in - 10-15 minuts
Step3 Design deepdive - 10-25 minutes
Step4 Wrap up - 3-5 minutes
