# how to tackle a system design interview question

## key notes

- system design interviews are designed to be **open ended conversations**
  - depending on how senior the position is, you are expected to lead the conversation
  - it's imperative that you ask questions during this process
    - not asking any / not asking enough questions is seen as a pretty big red flag during the process
- there is _usually_ "no right answer"
  - you should be able to hit the basic points and get a general structure
    - e.g. your final system design shouldn't be "client hits web server, calls API, gets data, that's it"
  - oftentimes, the interviewer won't have a super comprehensive structure that they're working with either
    - a lot of interviewers are a bit nervous during interviews too; remember this to ease your nerves a bit

## step 1: outline use cases, constraints, and assumptions

gather requirements and the scope of the problem. ask questions to clarify use cases and constraints. discuss assumptions. it is expected that you ask a lot of questions during this period of time. the system you are expected to build is generally pretty open-ended so you'll be expected to be asking primarily clarifying questions

examples of things to note at this stage:

- who is going to use it?
- how are they going to use it?
- how many users are there?
- what does the system do?
- what are the inputs and the outputs of the system
- how much data do we expect to handle?
- how many requests per second do we expect?
- what is the expected read to write ratio?

## step 2: create a high level design

as gayle laackman mcdowell says: **bruteforce first then optimize**

outline a high level design with all the important components:

- sketch the main components and connections
  - this is typically the "whiteboarding" stage
  - it's important to practice drawing things
    - it's a major red flag if you do not draw anything
    - draw simple components and simple connections (this initial design is your "bruteforce")
    - oftentimes people get too into the details at this stage and try to optimize too early and end up with nothing
- justify your ideas
  - explain the choices you're making and possible tradeoffs / alternatives

## step 3: design core components

this is subjective depending on the type of problem you're presented with:

- implementation detail level components
  - problem-specific challenges that depend on what question is being asked
  - it's good to be more collaborative at this stage
    - if you're stuck, you can outline some of the things you're thinking of and leverage your interviewers as a resource
  - examples:
    - **url shortening service**: hashing algorithms, hash collisions, database options, schemas
    - **twitter**: timeline service, search service, fanout service
- API and object-oriented design
  - some of this can get into some high level algorithms / data structures
  - again, this section is very context dependent so it depends on the problem you're being asked

## step 4: scale the design

common acronym to remember is **BUD**:

- **bottlenecks**
- **unnecessary work**
- **duplicated work**

there are a couple of common solutions that show up all the time:

- load balancers
  - this one can kind of be a double edged sword
    - a lot of people like to slap a load balancer right at the API layer and call it a day as if it's a silver bullet
    - you should understand a bit more about load balancers other than that they "balance the load"
- horizontal vs vertical scaling
- caching
  - caching policies
- database sharding

discuss potential solutions and tradeoffs. everything is a tradeoff

## back of the envelope calculations

i'm going to say something a little controversial

unless you're interviewing at FAANG or Big-N or unless your interviewer brings it up, i don't think it's necessary to do back of the envelope calculations. i think if you've spent a lot of time talking about design you will have most likely filled up all the time in your interview and there won't realistically be any time to do this

a lot of these calculations are based on assumptions and just memorized numbers. again, this is a much more frequent topic for backend, dev ops, or data engineers (less so for frontend)

---

## the leetcode template

templates are a great way of approaching problems. depending on the company you're interviewing at, sometimes getting "somewhere" is good enough and templates allow you to at least build yourself a foundation even if you're totally lost

time for each section is variable depending on the problem and how much time you're allotted for the interview (this time is allocated for roughly under an hour here)

### feature expectations (5 mins)

1. use cases
2. scenarios that will not be covered
3. who will use this application
4. how many people will use this application
5. usage patterns

### estimations (5 mins)

i would do this at a really high level with rough numbers (or no numbers at all) but this depends on how senior of a role you're applying for

1. throughput (queries per second (**qps**) for read / write queries)
2. latency expected from the system (for read and write queries)
3. read / write ratio
4. traffic estimates

- writes (qps, volume of data)
- read (qps, volume of data)

5. storage estimates
6. memory estimates

- if we are using a cache, what kind of data we want to store in the cache
- how much RAM and how many machines do we need for us to achieve this
- amount of data you want to store in the disk / ssd

### design goals (5 mins)

1. latency and throughput requirements
2. consistency vs availability (weak / strong / eventual => **consistency** vs failure / replication => **availability**)

### high level design (5 to 10 mins)

1. APIs for read / write for crucial components
2. database schema
3. basic algorithm
4. high level design for read / write scenario

### deep dive (15 to 20 mins)

1. scaling the algorithm
2. scaling individual components

- availbility, consistency, and scale story for each component
- consistency and availbility patterns

3. think about the following components and how they would fit in / help

- DNS
- CDN
- load balancers
- reverse proxy
- application layer scaling
- database (RDBMS vs NoSQL)
- caching
  - caching patterns
  - eviction policies
- asynchronism
  - message queues
  - task queues
  - back pressure
- communication
  - TCP
  - UDP
  - REST
  - RPC

### justify (5 mins)

1. throughput of each layer
2. latency caused by each layer
3. overall latency justification
