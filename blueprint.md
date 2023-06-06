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
