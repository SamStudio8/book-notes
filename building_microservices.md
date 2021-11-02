# Building microservices, Sam Newman, 2nd ed


## What are microservices


* independent, releasable services, modelled around a business domain
* tend to avoid sharing databases and information with other systems, encapsulating its own data
* information hiding - hiding as much as possible and exposing as little as possible, as implementation is hidden it is free to change so long as the endpoints work in a backwards compatible fashion
* related to the Hexagonal Architecture (Alistair Cockburn), splitting internals from externals allowing for internals to be interacted in different ways
* lack of consensus on how to do SOA (service oriented architecture) well, somewhat muddied by vendors, microservices are a form of SOA


### key concepts


* independent deployability (if you take only one thing from this book it should be this)
   * loosely coupled services mean changes in one can be made and deployed without interacting with anything else
   * requires explicit and stable contracts between services
   * hard to get right but comes with many benefits
* modelled around a business domain
   * DDD (domain driven design) helps structure code to better represent the real world the software operates in
   * think more about offering "end to end slices of business functionality" rather than a layered architecture (eg. presentation, logic and storage)
   * changes only affect one business domain rather than the presentation, logic and storage aspects of a traditional monolith
* owning their own state
   * if a service wants data held by another service, it should ask that service for it, rather than sharing a database
   * services get to decide what is shared and what is hidden
   * developers know what is free to change (hidden) and what isn't (contract with external consumers)
   * high cohesion for a particular business function
* size
   * James Lewis, "a microservice should be as big as my head", that is, it should be big enough to understand and no more
   * Chris Richardson, "aim to have the smallest interface possible"
   * more important to get the number of services right, rather than their size, and get their boundaries right so you get the most out of them
* flexibility
   * James Lewis "microservices buy you options"
   * need to decide whether the cost is worth the options you get, herein lies the art of microservices - balance of keeping options open and maintaining the cost of services
* alignment of architecture and organisation
   * Conway's Law: organisations which design systems are constrained to produce designs which are copies of the communication structures of their organisation
   * traditional "three tier" (presentation, logic, storage) reflects the organisation of IT teams more than anything else: high cohesion for technology but low business cohesion
   * IT assets aligned to core competencies
   * "stream aligned teams" (Team Topologies) are aligned to a single stream of value and can work without handing off to another team


### enabling technologies


* log aggregation and distributed tracing
   * need to be cautious about taking on too much new technology when moving to microservices, but log aggregation is pretty much essential
   * central place for storing and analysing log files and generating alerts
   * specifying correlation IDs can group separate distributed requests related to the same user interaction
   * Humio, Jaeger, Lightstep, Honeycomb
* containers and kubernetes
   * virtualization creates isolated execution environments for a service but can be overkill
   * containers are a somewhat lightweight alternative
   * significant advantages over traditional techniques for deployment but can be a headache and often not worth the trouble for small numbers of services
   * running your own kubernetes cluster is a lot of work
* streaming
   * moves away from monolithic databases and batch processing
   * Kafka is de facto standard
      * message permanence, compaction, scales well
      * Flink, Debezium
* serverless
   * public cloud computing that hide the underlying machines allowing one to work at a higher level of abstraction
   * message queues, storage containers, databases
   * function as a service platforms (AWS lambda)


### advantages of microservices


* tends to offer more advantages than other distributed architectures as the service boundaries are aggressively small
* technology heterogeneity
   * different technologies and techniques can be used inside each service, pick the right tool for the job
   * avoids one size fits all, lowest common denominator
   * can change entire technology stack if required to improve performance or scaling
   * allows for experimenting
* robustness
   * avoid cascading failure, degrade functionality instead of full failure
   * not a given, need to get it right
* scaling
   * can choose to scale only the parts of the system that need it (as opposed to a monolith)
   * cost savings from on demand dynamic scaling on public cloud
* ease of deployment
   * low impact, low risk deployment compared to monolith, can deploy more often without fear
* organisation alignment
   * minimise the number of people and teams working on a single codebase
   * better ownership
* composability
   * functionality can be consumed in different ways for different purposes


### pain points for microservices


obviously bring a lot of complexity
many issues not specifically limited to microservices but any distributed system
* developer experience
   * harder to run and develop locally
   * what happens when you can't simulate all the services you need on one machine
   * may need to limit the scope of what a developer can work on
* technology overload
   * wealth of new toys can be overwhelming
   * inescapable fundamental challenges
      * data consistency, latency, service modelling
   * will reduce time for shipping new features to start with
   * gradually increase complexity and introduce new technologies as you need them
* cost
   * increase in cost to begin with, more processes, more network, more storage
   * takes time to get to grips with new ways of doing things which will impact productivity
   * poor choice of architecture for an organisation with a cost cutting mentality
* reporting
   * stakeholders can easily analyse all data in a monolithic system
   * more difficult to do large joins on data with multiple logically isolated schemas
   * requires streaming or publishing to a separate central database
* monitoring and troubleshooting
   * monoliths have simplistic approach to monitoring
   * operating mode of a monolith somewhat binary (it's up or its down), harder to understand the status of a microservices architecture
   * Distributed Systems Observability, Cindy Sridharan
* security
   * information flows harder to understand and secure
   * data flows over the network more, leaving it open to interception and MITM
   * need to configure authorisation on all endpoints that need it
* testing
   * end to end tests offer most confidence but are extremely difficult in microservices architecture
   * can generate lots of false negatives
   * diminishing returns as testing costs more and offers less confidence
   * contract driven testing and testing in production, canary releases are useful alternatives
* latency
   * processing that would have been on one process may now span multiple system
   * incurs serialisation, network transmission and deserialising
   * important to gradually move to microservices to avoid an explosion of latency
   * slower operations are acceptable if they are still fast enough
* consistency
   * data safety cannot be provided in a distributed system
   * distributed transactions are highly problematic
   * eventual consistency helps to reason about state


### the monolith


* consider as a unit of deployment, a monolith isn't about size, but just a system that has to be deployed altogether
* monoliths aren't bad, some people confuse the term monolith with "legacy"
* easier to deploy, test and promote code reuse 
* really, you need a good reason to move to microservices over a monolith!
* types of monolith
   * single process monolith
      * code packed into one process
      * might run in multiple places for robustness or scale
   * modular monolith
      * contains modules that can be developed independently but the application must still be deployed as a whole
      * not a new idea but still not really engaged with very well
      * can allow for lots of parallel work if done correctly, with the added benefit of not having to look after multiple services (trivial deployment topology)
      * main challenge is the database is often not decomposed to the same granularity as the application which can couple teams unnecessarily
      * Deconstructing the Monolith, Kirsten Westeinde, Shopify
   * distributed monolith
      * Leslie Lamport "A distributed system is one in which the failure of a computer you didn't even know existed can render your own computer unusable"
      * consists of multiple services but must still be deployed together
      * has the complexity of a distributed system with none of the benefits
* delivery contention
   * different people working on the same code, wanting to push changes at different times
   * can be confusing around who owns and makes decisions on what
   * microservices can reduce this contention with more clear boundaries and service owners


### adopting microservices


* many challenges, requires a lot of thought
* often a bad choice for new products and start ups as the business domain is not pinned down, changes across service boundaries is very expensive (which is why it's important to get them right)
* lots of work to handle and manage deployment of microservices (microservices tax) can be a strain on smaller teams
* products that are built to be managed and deployed by the end user will unacceptably push all the operational difficulty to the users
* best reason to move to microservices is it allows more developers to work on the same system without getting in each others way
* SaaS products are expected to work 24/7 which can be easier to manage with independent releasable code
* technology agnostic nature of microservices allows you to get the most out of cloud platforms


## How to Model Microservices


### What makes a good boundary?


* information hiding - David Parnas, hide as many details as possible behind a module (or service boundary)
   * improved development time as more work can be done in parallel by more developers
   * comprehension, easier to understand what the service and wider system does
   * flexibility, modules can be changed from one another without impact, modules can be combined for additional functionality
   * reducing assumptions modules make about each other
* cohesion
   * code that changes together, stays together
   * related behaviour should sit in the same service
* coupling




### types of coupling


* a degree of coupling is inevitable, the goal is to reduce coupling as much as possible
* concepts overlap with structured programming
* spectrum of looser to tighter coupling (domain, passthrough, common, content)


* domain coupling
   * a microservice needs to interact with another to make use of its functionality
   * unavoidable in microservices as services need collaborate and work together
   * though one service depended on by many other services can imply its doing too much work
   * share only what you have to and send only what is needed
* aside: temporal coupling
   * an order service may send a message to a payment service that must complete for the order to be successful, requiring both services to be up and a response to be returned
   * can cause trouble with scale 
   * can be overcome in some cases by moving to async communication (message queues)
* pass through coupling
   * a microservice passes data through to another
   * means the caller has to know where the data is going and what other services are involved
   * also means changes to one service will affect multiple downstream services that don't look related on the surface
   * calling service can just bypass the intermediary and make a request to the appropriate downstream services itself, increases domain coupling but that's a looser form of coupling
   * alternatively just hide low level details (eg. particular data models etc.) from other services
   * or make them proper passthrough and the intermediary service cannot look at or use the contents at all
* common coupling
   * occurs when services use the same data set
   * a shared database, memory or filesystem
   * changes to the data will affect multiple services immediately
   * also a source of resource contention
   * less problematic in the case of fixed, read only data
   * particularly bad when writing to the same data (eg a shared database or table)
   * a state machine would allow only valid transitions to be written to a shared resource
   * potential solution is to nominate or create a new service that is solely responsible for setting the state of the shared resource (but prepare for the possibility that this service may reject your request if the transition is not valid), however this can reduce module cohesion (as now code to manage state is taken out of one service into another)
* content coupling
   * one service reaches into the internals of another to change state, eg, changing a database table under the management of another service, bypassing that service
   * similar to common coupling, but the delineation of the shared resource is clear in common coupling
   * at best, it's a duplication of logic but at worst it can allow for incorrect transitions to be made
   * also violates information hiding as the internal state is exposed to another service
   * don't do this


### ddd and microservices


* ubiquitous language
   * use the same terms as the users
   * hard to build general systems
* aggregate
   * a somewhat confusing concept with many definitions
   * typically have a life cycle (good for state machines)
   * encapsulation of objects that have meaning in a specific context (eg. order lines only make sense in the context of an order)
   * one aggregate should be managed by one service
   * want to ensure that illegal state transitions cannot happen and this is controlled by the aggregate
   * can be useful to use URIs as IDs to help services identity who "owns" records in a foreign key
      * SoundCloud pseudo URIs
* bounded context
   * larger building blocks, eg "finance activities" and "warehouse activities"
   * a group of aggregates in which some aggregates are exposed and some are internal
   * hide implementation details with explicit interface to other contexts
   * shared models between contexts will be minimal and specific
   * different information might be stored about the same model in different bounded context (eg. a customer in a warehouse and customer in a finance system will have different data fields) and those may still need to link to a global customer
   * a good place to start a microservice
   * you can even split a context again in future (and hide that implementation from the outside!)
* event storming
   * stakeholders discuss development of domain model in one large room
   * map out domain events, then commands (a decision), aggregates, contexts
   * Event Storming, Alberto Brandolini


### alternatives to DDD


* volatility
   * more frequently changing parts of system grouped together
   * similar to bimodal IT model 
   * has a tendency to dump things that are hard to work with in a box and forget about them
* data
   * some boundaries may be delineated purely based on the data they hold
   * you may want to keep PII inside as few services as possible or have a service purely for PII to reduce the auditing surface area
* technology
   * may be essential to take advantage of different technology stacks
   * the classic three tier architecture is an example of technology based delineation and is exactly what we are trying to move away from
* organisational
   * as per Conway's Law, how you end up organising yourself will affect your design
   * have to consider how teams are organised when designing a system they will develop, maintain and use
   * a service that cuts across many teams will likely cause trouble
   * aim to slice vertically not horizontally


## Splitting the monolith


* microservices are not a goal
* have a clear idea of what you want to achieve and use microservices only if it can't be done with your current architecture
* "microservices aren't easy, try the simple stuff first"
* Martin Fowler: "if you do a big bang rewrite the only thing you're guaranteed of is a big bang"
* "you won't appreciate the true horror, pain and suffering that a microservice architecture can bring until you are running in production"
* the monolith is not your enemy and remains a valid architecture choice
* removing the monolith entirely only tends to work in situations where the monolith is based on dead technology, infrastructure that needs to be retired or expensive third party systems
* premature decomposition is costly
* splitting services is a balancing act between the benefit of making the service and the pain of extracting it from the monolith
* important to consider decomposition of the UI, usually lags the decomposition of the backend but make sure it doesn't lag too much
* often splits are made code first (not data first) as it's usually easier and delivers value more quickly
* Monolith to Microservices, Sam Newman


* useful patterns
   * strangler fig (Martin Fowler)
      * intercept calls to existing system and route to the existing monolith or new services as appropriate
      * allows for a new system to slowly replace an old one
      * monolith unaware it is wrapped
   * parallel running
      * run the monolith and microservice and feed them both the same requests and compare the results
   * feature toggle
      * Pete Hodgson Feature Flags
      * a cross between strangler fig and parallel running
* possible concerns
   * performance
      * moves joins from efficient database joins to application joins
      * can be reduced by allowing bulk requests for data between services, or caching needed data from the other service so it is locally available
   * integrity
      * we lose enforcement of key integrity, no automatic deleting cascade, no uniqueness checks
      * get over it, basically
      * soft delete (mark as deleted but don't remove) is one "coping mechanism"
   * transactions
      * we lose ACID safety
      * get over it, again
   * tools
      * can be hard to make changes when data is concerned, data has state
      * database migrations
      * Flybase, Liquibase
   * reporting
      * services share as little as possible which makes it difficult to serve analytical needs
      * dedicated database for external access 
      * means the service gets to keep hiding internal business so long as it can push the data to another database for analysis


## Microservice communication styles


* can't just map a call to another service to a function call
* performance
   * no compiler or runtime optimisation
   * significant overhead compared to intraprocess calls
   * need to be mindful of the sizes of parameters as they are passed across a network instead of passed around as memory pointers
   * may need to pass less data, use compression or better serialising, or a filesystem
   * code that makes a network request should be clear and not abstracted away
* interface changes
   * in a single process application code using and calling an interface are packaged in the same process, IDEs can even handle changes automatically
   * lockstep deployment must be done for all microservices that are affected by backward incompatible changes
* error handling
   * nature of errors are different, no longer split between easy to handle or catastrophic
   * Distributed Systems (Andrew Tanenbaum, Maarten Steen) offer five categories for distributed system errors
      * Crash -- server crashed, need reboot
      * Omissions -- sent something but didn't get a reply
      * Timing -- something happened too late or too early, or was not received in time
      * Response -- response is wrong
      * Arbitrary -- something is wrong and nobody knows what, including Byzantine faults
   * errors often transient in distributed systems
   * http requests have response codes for a reason
      * rich semantics around errors
      * clients can make decisions whether to retry


* easy to be overwhelmed by choice and stick to only what you know, often stuck with constraints that might not be suitable
* best to find the type of interprocess communication that fits best before shopping for solutions


### collaboration styles


* usually a mix and match, some processes work best as request and response, others more flexible
* blocking vs non blocking
   * blocking is simple and easy to understand
   * introduces temporal coupling which can lead to cascading errors
   * becomes more problematic when many services are chained together
   * async calls will remove temporal coupling, any future instance of the other service can pick up messages
   * async request response still possible
   * async useful for longer processing time, or anything requiring intervention (eg. waiting for a reply from a warehouse)
   * more complex to reason with as a developer
   * async/await is a sort of blocking asynchronous pattern, can be useful for parallel requests
* request - response (can be blocking or async)
   * sometimes you just have to make sure that something gets done
   * sync blocking will usually involve opening a connection and keeping it open
   * async non blocking systems need to be able to match requests to responses, whenever they appear
   * need to think about timeouts, for async methods too
   * pretty much the only option where a result is needed before further processing can continue
* event driven (limited to async non blocking)
   * inherently async
   * services emit events that may or may not cause some processing to happen in one or more other services
   * a statement about something occurring with no knowledge about how that event may get used
   * loose coupling between services, domain coupling reduced as sender doesn't need to know what receivers do with the messages
   * Inversion of responsibility compared to request response, pushing work to other services
   * events are facts, messages are an implementation detail
   * two considerations: how to send and how to receive
      * message brokers try to handle both
      * "keep your smarts in the endpoints", vendors try and add a lot of magic to message brokers
      * atom is a rest compliant spec for sending events over http, but generally brokers have more features for free (eg. support for competing clients)
      * "fully detailed events" generally better to avoid downstream services calling other services for linked data (eg. user registration could send all info about a new user), also helps with auditing and reconstruction of state (event sourcing)
      * fine balance between sharing enough to make the event useful but not so much you'd break information hiding - share only what you want to share downstream
   * non obvious problems to deal with, dead letters, how to replace messages, retries, reporting
   * correlation IDs are useful to link messages
* common data
   * one service shares data in a common location with other services
   * data lake and data warehouse form a spectrum of coupling (from raw to structured)
   * good for lots of data volume
   * not particularly suited for low latency situations without a way to trigger downstream updates
   * potential source of coupling (of the worst kind)
   * only as reliable as the underlying storage


## Implementing microservice communication


* make backwards compatibility easy
   * new fields shouldn't break clients
* explicit interfaces
   * both to users and developers
   * schemas
* technology agnostic
   * avoid choices that impose constraints on your tech stack
* simple for consumers
* hide implementation
   * avoid technologies that require exposing of internal details


### choices


* RPC
   * SOAP, gRPC
   * local methods to be called on remote processes
   * usually needs explicit schema SOAP has WSDL, RPC has IDL
   * explicit schemas make it easier to automatically generate some of the client code, Avro RPC can send the whole schema to allow clients to dynamically react to payload
   * protocol essentially concerns itself with (de) serialisation
   * some implementations such as java RMI are tightly coupled to a technology stack (needs the JVM), gRPC SOAP Thrift are more interoperable
   * ultimately local calls are not remote calls and RPC can be a pitfall if used unwisely
   * can be brittle to changes (new fields) as client code stubs need changing
   * SOAP is heavyweight, RMI is brittle, gRPC a more modern choice, good for choices where you have a lot of control over the client and server
   * gRPC works with http/2
   * in cases where you expect many clients with lots of various use cases, REST might be a better fit
* REST
   * expose resources through verbs
   * how resources are shown are decoupled from how they are stored (eg. JSON replies but SQL store)
   * Richardson maturity model compares styles of REST
   * typically implemented over http, using http verbs
   * large ecosystem based on http, caching with Varnish, mod_proxy load balancing, monitoring, http auth
   * hypermedia as the engine of application state (HATEOAS), clients should perform interactions leading to state transitions by following links
   * allows for big changes to responses so long as the hypermedia controls to perform actions can still be found, URIs can change and everything
   * HATEOAS seems like a good idea that didn't catch on, but can be a chatty protocol as clients need to find the controls they are looking for
   * can be tightly coupled with repetitive code in clients for accessing resources on server, alleviated somewhat by OpenAPI (but client code generation still not used often), very complex schema compared to alternatives like protobuf
   * less performant than binary protocols like RPC Thrift
   * http overhead, though http3 will use QUIC which will offer the same guarantee as TCP with less overhead (could use internally)
   * obvious choice for synchronous request response to support a variety of clients
   * good for request caching and scaling but less efficient
* GraphQL
   * newer protocol for defining queries, excels specifically because clients can define their query rather than making several requests
   * GraphQL endpoint is the entry for all requests and exposes a schema for clients
   * can cause high server load if queries are complex, harder to track down performance problems compared to more traditional SQL
   * hard to cache responses, may need to keep track of all replies to help
   * can often see some mix of GraphQL and REST in applications to cope with caching, or using REST for writes
   * careful not to treat a GraphQL service as nothing more than a wrapper around the data, services should do stuff and not be too coupled to the data storage
   * sweet spot is system perimeter, good for GUI and mobile devices exposing smart search, can be useful in making complex APIs with multiple resources more user friendly
   * aggregates calls to other services and filters results, deals only with this one aspect
* Message brokers
   * middleware, sits between processes to manage their communication
   * point to point queues or multi client topics
   * competing consumers pattern - groups of consumers that read the same queue without repetition of messages
   * topic style messaging somewhat more decoupled as you have some idea of who is reading a particular queue, but topics are more arbitrary
   * broker exists to provide delivery guarantees, ensuring messages are held durably until delivery (the guarantee is only as strong as the deployment of the broker)
   * exactly once delivery somewhat controversial as its implementation usually has a drawback, better to just build clients that can handle a repeated message
   * some brokers make guarantees on message order
   * RabbitMQ ActiveMQ Kafka are popular
   * SQS was AWS second product, launched in 2006
   * Kafka particular popular
      * high throughput, streaming
      * support thousands of consumers and producers
      * message permanence (rather than deleting read messages)
      * KSQL can be used for materialised views on the fly
      * see: Designing Event Driven Systems (Ben Stepford) and Kafka the Definitive Guide (Neha Narkhede et al)




### serialisation and schemas


* some technology limits your choices, gRPC uses some sort of binary protocol buffer
* JSON has usurped XML but it's simplicity comes at a cost (XML encouraged well defined schemas)
* Avro sends the schema as part of the payload
* if you're worried about payload size of efficiency, binary formats are the way to go
* often better to send less data or not send it at all before worrying about optimisation
* schemas useful for catching breakages of endpoints as well as documentation
* endpoints contracts can be broken by 
   * structural change
      * fields or methods added or removed such than a consumer is now incompatible
      * addressed by schemas and/or testing
   * semantic change
      * endpoint behaviour changes
      * harder to identify and more problematic
      * addressed by testing
* a microservice has a schema, it's up to you whether it's explicit or implicit
   * explicit schema makes it clear what is shared and what is hidden


### avoid breaking changes


* expansion changes -- add new things, don't remove old ones
* tolerant reader -- be flexible in your inputs
   * use what you need, ignore everything else
* right technology -- pick tech that makes backwards compatible changes easier
   * avoid java RMI seems to be the message here
   * protobuf field numbering
   * Avro schemas
* explicit interface -- to help understand what can be changed freely or not
   * OpenAPI and JSON Schema catching on
   * hard to get this right for message brokers
      * AsyncAPI and CloudEvents
* catch breaking changes early
   * Protolock, json-schema-diff-validator, openapi-diff-tool (Azure)
   * Confluent Schema Registry for Kafka
   * consumer driven contract testing, Pact tool


### managing breaking changes


* owner and consumers need to be aware of change, what does the change look like, how did it affect consumers and what do consumers need to do, by when
* good to track usage and client IDs to find clients that have not updated (user-agent)
* code reuse can encourage breaking changes - consider a shared library used by multiple services, can you cope with multiple consumers using different versions of this shared library?
* clients should be in charge of their own releases and should be independent enough from the service
* avoid leaking logic from the service into the client
* lockstep deployment
   * interface and consumers updated at the same time
   * garbage solution that implies the system is too coupled
   * reasonable if you control all the clients or have very few users, but is a bad habit
* coexist
   * run both versions side by side
   * route traffic from old clients to old service
   * expensive, use sparingly
   * bugs need to be backported
   * needs work to manage the routing and can make it hard for you to reason about how requests are served
   * may be a good way to test new version for minutes or hours before a full deploy
* emulate
   * new version emulates old version too
   * can eventually remove the old code when clients have migrated
   * internally transform old requests to look like new ones if easier -- expand and contract pattern
   * REST clients can put the version number in the URI or header




### service discovery


* DNS
   * service-environment.domain.com or different nameservers for each environment
   * well understood but not designed for highly disposable hosts, AWS Route 53 tries to overcome this
   * DNS spec TTL and caching can be problematic
   * DNS could just point to load balancer and let the balancer handle trafficking
   * DNS fine for small scale
* service registry
   * zookeeper
      * developed for Hadoop
      * bewildering array of use cases -- config management, data sync, leader elections, message queues and naming services
      * needs at least 3 nodes to be deployed properly and robustly
      * hierarchical namespace
      * clients can watch nodes in the namespace and be alerted to changes
      * somewhat de facto but author argues better options exist for service registry
   * consul
      * richer features than zookeeper for service registry
      * has a DNS server in the box
      * endpoints health monitoring
      * everything done through rest API
      * consul-template can update text files to trigger configuration changes
      * integrated nicely with Vault for secret management
   * etcd and kubernetes
      * etcd bundled with kubernetes
      * pattern matching on metadata about pods to assemble a service directory
* DIY
   * tags in aws, use aws APIs to lookup tags and get metadata such as host
   * reinventing a bad wheel
* don't forget to make service registry available to humans


### service mesh and API gateways


* surrounded by hype and confusion
* API gateway particularly prone to misuse, API economy more of a bust than a boon, plenty of bad VC backed companies fighting for space that doesn't exist
* "East west traffic" inside a data center, "north south traffic" in and out to the outside world
* API gateway sits on a network perimeter (DC, VLAN, kube cluster), and handles north south traffic
* Service mesh handles east west traffic inside, dealing with communications between microservices
* very simply they act as proxies between services and can provide logging or discovery functions that might otherwise be deployed as code
* gateways can also handle east west traffic, adding to confusion
* important that these systems are generic and have no relationship to the services
* API gateway largely acts as reverse proxy and handle API keys, rate limiting and logging
* if you just want to expose services in a kube cluster just use Ambassador
* bad smell to use an API gateway for aggregation of calls to multiple services, use GraphQL or BFF (backend for frontend)
* protocol rewriting another bad smell, eg. converting SOAP to REST
* keep the pipes dumb and the endpoints smart
* adds a barrier to change, API gateways are tightly controlled
* service mesh merge common functions such as TLS, correlation IDs, discovery and balancing, helps provide consistency to multiple services
* service mesh can help unify services with different technical stacks or kube clusters and teams
* service mesh has to take into account that there is probably much more east west than north south traffic, they try and limit the impact of those horizontal calls
* mesh proxy often local to the machine for efficiency, service thinks the call is remote. control plane controls the proxies.
* Envoy proxy is popular (also used in Ambassador)
* aren't these just smart pipes? key is business logic isn't leaking into the pipes, it's boring generic stuff
* think seriously if you need one, then think seriously about which one, many options geared to kube
* Monzo painfully moved from Linkerd v1 to its own Envoy based system
* these products only deal with http (covering REST, SOAP, GRPC) but what about brokers like kakfa? not much in that space at the moment


### documenting services


decomposition of systems aims to expose "seams" that can be used for cool things, but nobody will know what they can do without documentation


* explicit schema
   * show the structure but not the intent
   * without an explicit schema the documentation has to work harder
* Martin Fowler "humane registry"
   * wiki basic and will get out of date
   * use the APIs and power of the systems to generate dashboards and docs
* fancy systems
   * BizOps from FT 
   * Spotify Backstage
   * Ambassador service catalogue


## Workflow


how do multiple services collaborate for a business transaction?


ACID recap
* atomicity, things happen or fail together
* consistency, after changes the database is in a valid state
* isolation, transactions do not interfere with eachother
* durability, once the change has happened it will stay


* transactions that span multiple service lose atomicity by default, need to consider the various states that could happen and how to handle them
* distributed transactions are tricky and don't really work
   * two phase commit 2PC can violate isolation as there may be a window in which parts of the same transaction are committed or not
   * 2PC is distributed locking, big queries or long processing will keep the resource locked for longer
* if you really want ACID, you should keep those parts together in the same service


### sagas (hector garcia-molina, Kenneth Salem)


* models the discrete steps needed to complete a transaction
* originally designed to handle long lived transactions (LLT)
* attempts to reduce scope and therefore locking and contention into many small steps
* be clear this does not offer acidity automatically, it's up to the developer to reason about the saga state and atomicity
* failure of a saga, one or both:
   * backwards
      * revert and clean up
      * requires compensating actions to undo committed steps
   * forwards
      * allows picking up an aborted transaction and continuing
      * system must persist enough information to facilitate retry
* sagas are designed for business failures not technical ones (eg. a failed payment should be retried, but if the payment service is down you'll need to handle that outside the saga)
* imagine an item has been paid for and now can't be found, we need compensating transaction to do backwards recovery
   * note these transactions did happen, not as simple as a rollback -- "semantic rollback"
   * you'll want to keep information on the failed transaction as well as the compensation transactions
* consider reordering the chain of events to make rollbacks easier (eg. less important state updates can go after the hard part has successfully happened rather than before)
   * put the business steps more likely to fail earlier


### implementing sagas


* orchestrated sagas
   * central coordinator, command and control approach
   * knows what services and steps are involved in the saga
   * heavy use of request response
   * couples the service that does the orchestration to the other services involved, hard to avoid domain coupling here
   * risk of logic being pushed into the orchestrator
   * Business Process Modeling (BPM)
      * Zeebe and Camunda
* choreographed sagas
   * distributed responsibility, trust but verify
   * event driven
   * reduced domain coupling (and coupling generally)
   * harder to reason with and hard to understand how the saga works
   * need to be able to work out the saga state from the event stream (correlation ID)
* garcia-molina and Salem also talk about nested sagas
* author argues the benefit of reduced coupling outweighs the ease of understanding offered by orchestration, especially if multiple teams are involved in developing different parts of the application
* sagas force you to explicitly model business transactions which is very useful
* Enterprise Integration Patterns (Hohpe and Woolf), Practical Process Automation (Ruecker)


## build


### continuous integration


* keep everyone in sync, daily minimum
* checks out new code, integrates it and runs tests
* CI will often deploy "artifacts" to run tests against
* embracing infrastructure as code means all the code to configure the services is version controlled too
* Jez Humble three questions
   * do you check into main at least once a day
   * does a test suite validate changes
   * when the build is broken, is fixing it the #1 priority
* source code branching models full of controversy these days
   * author argues that feature branches delays integration
   * trunk based development with feature flags allows integration to be tested and the unfinished features can be disabled
   * if you do use branching, keep the life cycle short
* may have short, fast tests and long large scoped tests
   * build pipeline
   * dedicated stages
   * creates deployable artifacts (ie. the service) used throughout the pipeline
* continuous delivery (Jez Humble and Dave Farley)
   * model how to get software from check-in to prod
   * as tests get larger and slower, you want the environment to be increasingly production like
   * fail as early as possible but want to have confidence this will work in prod
* continuous deployment builds on continuous delivery, any passing artifact is automatically deployed
   * often human interaction post check-in is a bottleneck that needs addressing
* the artifact you build should be the exact artifact you deploy, otherwise how do you know it works
   * store artifacts in Artifactory, Nexus or a container registry
   * store environment configuration outside the artifact
* patterns for storing code -- another source of contention
   * multirepo
      * each service in its own repo
      * strong ownership model
      * cannot do atomic commits across multiple repos, nor atomic deployment -- cross cutting changes should not be the norm however so this might imply your service boundaries are wrong
      * can be slightly harder to reuse code libraries across repos as they may not deploy at the same time (need to be ok with different versions)
   * monorepo
      * all services in single repo
      * allows atomic changes to multiple services
      * easier to reuse code and keep libraries in sync
      * build system usually maps parts of the repo to trigger partial builds of the full system when changed occur
      * Bazel built by Google to make their monorepo possible
      * weaker ownership model
         * git CODEOWNERS can strengthen this
      * may have multiple monorepos
         * some orgs have a monorepo per team with collective ownership
      * often requires investing in tooling, sweet spot is giant tech orgs or small teams
   * megamonorepo
      * every single change builds the entire system
      * don't do this


## deployment


* much harder to deploy microservices than monoliths
* kubernetes more popular than v1 of the book
* function as a service is a new way to think about shipping software
* big cloud providers don't tend to offer an SLA on a single machine or even an availability zone
* logical view hides a lot of the physical complexity of the system
   * multiple instances, load balancing, physical location (DC, geo region)
   * databases will be shared by multiple instances of the service, may have multiple read replicas
      * two databases may even share hardware and even an engine, but will be treated as separate -- more likely on prem than cloud
   * environment -- prod, testing, development, will also have different topologies (too expensive for test environment to reflect prod perfectly), topology should be configurable and automated whenever possible


### principles of microservice deployment


* isolated execution
   * services run independently and cannot impact other services
   * might be tempted to put services on the same machine to save host management
      * harder to monitor
      * one service can degrade everything else on the same machine
      * inhibits team autonomy as deploys must be coordinated
      * violates independent deployability
   * improvements in containers have made it easier to deploy into isolated execution environments
* focus on automation
   * make it easy for developers to self provision services to keep them productive
* infrastructure as code
   * automatic, repeatable environments
   * chef, puppet, ansible, even just bash
   * for cloud, Terraform, Pulumi, CloudFormation for AWS
   * system can be brought to a known state by code
   * Infrastructure as Code (Kief Morris)
* zero downtime deployment
   * no need for communication with consumers to warn of downtime
   * gives confidence to release more often and during the day
   * "trivial" if your service is async as the messages will wait while you deploy
      * rolling upgrades with kube can overcome otherwise
      * blue green deployment
   * helps to build with this in mind 
* desired state management
   * have a means to ensure the system runs with a given state, eg. new machines are spun up to replace failed ones
   * developers can focus on what the right state is rather than how to do it
   * kubernetes, nomad is even more flexible and can handle other workloads


### deployment options


* physical machine
   * increasingly rare
   * low utilisation of hardware, low cost effectiveness
   * can encourage packing of services onto machines in the same env, violating isolated execution
* virtual machine
   * reduced host management, increased utilisation
   * each VM has its own OS and resources, permitting isolation between VMs
   * diminishing returns as virtualization has some overhead
      * container based
      * type 2 virtualization
         * hypervisor maps resources from VM to host and controls the VMs
         * hypervisor needs its own resources
         * each VM OS needs its own resources too
      * type 1 virtualization
         * VMs run directly on hardware, uncommon
   * needs to be automated, but many orgs keep this under lock and key in a central ops team
* container
   * popularised by docker and allied with orchestration tools like kubernetes
   * kernel assigns resources to processes
   * container is essentially an abstraction over a subtree of processes
   * no hypervisor, no kernel
   * container can have an OS but makes use of shared kernel
   * much faster to provision (seconds rather than minutes)
   * can pack more onto one host as they are more lightweight
   * hypervisor does a lot for you that you have to handle yourself, docker made this much easier
   * much weaker isolation (common to find exploits to break out of a container and interact with the host or other containers) -- good for trusted software
   * windows has to rewrite their OS for containers because it was big in space and resources, size disparity still exists between Linux and windows containers
   * docker desktop allows developers to run a bunch of services on their machine for testing
* application container
   * applications sit inside a shared container on a single host to provide cluster management, monitoring tools etc.
   * useful for java where all services can share the JVM
   * constrains the tech stack
   * slow and blurs monitoring of applications inside the container
   * violates isolation
   * usually commercial and costly
* platform as a service
   * higher level of abstraction than a host
   * can offer transparent scaling and other useful tools like hosting databases
   * work well if they exactly suit your needs but otherwise hard to make them do what you want
   * usually try to be too smart, eg. heuristic scaling
   * Heroku gold standard but reducing in popularity
   * Google App Engine, AWS Beanstalk
   * not so much growth in this space as "serverless" seems to be taking off instead -- turnkey solutions of mix and match parts
* function as a service
   * "serverless" is the only tech to come anywhere near the hype of kubernetes
      * umbrella term for services where the underlying computer doesn't exist
      * Ken Fromm: doesn't mean servers aren't involved, just developers don't need to think about them
      * FaaS is indistinguishable from serverless for some but overlooks databases, queues, storage
   * AWS Lambda in 2014
      * deploy code that is dormant until it is triggered
      * isn't costing you money until it runs
      * good for low and unpredictable load
   * reduces operational overhead
   * limitations on what can be run exactly and how it can be run
      * limitations on language, exec environment, CPU and IO, time limits
      * if you find you need to do tuning, or long runtimes, probably not the right option
      * stateless (except for Azure Durable Functions)
   * cold start times not so much an issue anymore
   * can scale to overwhelming numbers of concurrent processes that cripple other services that cannot scale as high or as fast
   * mapping to a service
      * single function as a microservice, will need a way to dispatch to different functions
      * function for each domain aggregate, need to think about the external interface (keep it simple in case you want to merge things back together) and shared data
   * care must be taken to not break things so small that you create inconsistencies in state transition
   * can be much easier than kubernetes


### which option? Sam's basic rules of thumb for working out where to deploy stuff


* if it ain't broke don't fix it
* give up as much control as you are happy with (and then a little more)
* containers are not pain free but offers a compromise around isolation and local development


### kubernetes


* the rise of the container has changed things, puppet and chef no longer the standard as the complexity of deploying to the same machine over and over doesn't exist in containers
* despite many systems to manage container deployment, kubernetes has emerged on top
* started at Google, inspired by omega and Borg
   * more developer friendly and less focussed on scale
   * Cloud Native Computing Federation CNCF has done a lot of work to unfragment the kubernetes landscape
* "container scheduler" or "container orchestration platform"
* docker defines the container on a single machine, orchestrator works out how and where to run workloads
* kubernetes needs a book in its own right
* kube cluster topology
   * control plane
      * nodes (can be physical or virtual)
         * one or more pods
            * one or more containers
* kube cluster service
   * stable routing endpoint
   * maps pods to a network interface
   * pods exposed
   * services live forever, pods are ephemeral
   * this is how your pods are accessed
   * you deploy pods, pods map to a service
* kube deployment
   * handles replica sets "I want four of these pods"
   * can handle rolling updates, rollbacks
* define a pod, a service, a deployment
* commonly a pod has one container but may contain another container running a proxy like Envoy (see earlier)
* difficulties arise with multi tenancy as kube was not designed for this
   * Red Hat OpenShift
      * expensive and vendor specific
   * federation model
      * multiple kube clusters with some management software
   * generally challenge of scale that you won't have to worry about
* applications on kube may be built for portability in theory but not always true in practice
* yet more tools exist to manage life cycle of third party systems (eg. kakfa)
   * Operator and Helm
      * common utilities packaged in a black box manner
      * Helm "the missing package manager"
      * Operator seems to help with ongoing life cycle stuff
   * Custom Resource Definitions CRDS
      * extend kube API
      * seamless integration to CLI/ACL
      * can add your own abstractions for configuration, or adding services etc
   * somewhat a trend in the kubernetes ecosystem that nobody has settled on a good way to do anything
* Knative
   * tries to provide FaaS with kube
   * tries to solve the problem of kube not being friendly
   * needs the Istio service mesh
   * not really production ready
   * notably not under the wing of CNCF
* author expects kube will stay but the trend will move to more abstraction of it, you'll be using it without knowing it
* don't use kube because everyone else is using it


### progressive delivery


* separating deployment from release
   * Jez Humble: deployment is installing to an environment and release is making it available to users
   * check software works in production environment before users can access it
   * blue green deployment
      * deploy new version alongside existing version and switch customers if there are no problems
* James Governor: "continuous delivery with fine grained control over the blast radius"
* feature toggles or feature flags
   * hide functionality behind a switch
   * useful for trunk based development where unfinished work can be turned off
   * can turn off problematic features
   * coordinating releases
   * not all or nothing, can have flags based on users or some other status
   * Pete Hodgson Feature Flags
* canary release
   * mistakes are unavoidable
   * limit new functionality to a subset of users
   * reduce impact of errors
   * tools like Spinnaker can handle this on metrics
* parallel running
   * run new and old system and compare responses
   * dispatch calls to both systems and keep one
   * careful not to perform a transaction twice
   * GitHub Scientist tool


## testing


increasingly testing is done in production, blurring the lines between development and production


### types of test


* acceptance -- did we build the right thing
* exploratory -- how can we break the system
   * visual assertions and other automatic tests can help get rid of manual checking, freeing time to use the system as an end user and try to break it
* unit -- did we build the pieces right
* property -- are the response times, scale, performance and security up to scratch




### test scope


* Mike Cohn test pyramid
   * end to end, service, unit
   * all mean different things to different people but generally test scope and confidence increases toward top but speed and isolation decreases
   * test snow cone is an anti pattern where you have lots of big tests and few unit tests
* unit testing
   * test single functions
   * TDD tests will fall into unit testing
   * no services, external files or network generally needed
   * usually fast and offer fast feedback
   * mostly for developers and testing technology facing stuff (Marick terminology)
   * useful for trusting refactoring
* service testing
   * bypass UI and tests an individual service
   * in a monolith this might test a class or some classes that provide a sort of service
   * might test with a real database or with stubbed services over a network
   * harder to work out what is broken when it fails as the scope is bigger than a unit test
* end to end tests
   * entire system in scope
   * might drive a browser client or mimicking user actions like file uploads
   * high degree of confidence when they pass


### writing service tests


* stub basic replies from collaborators
* mocks go a step further and will check the calls are correctly made, needs more smarts than just a stub provides
* Growing Object Orientated Software Guided by Tests (Freeman and Pryce)
* Brandon Byars Mounteback tool for stub/mock services
   * can be used to stub multiple services
   * send it commands about imposters and what it should do with messages on a given port
   * supports TCP HTTPS SMTP


### end to end tests


* build pipeline should fan in successful builds of related services to make a single end to end test environment
* temporary network issues can make tests brittle
* threading tests can lead to timeouts and race conditions, making tests flaky
* need to remove brittle and flaky tests to keep confidence in the test suite -- Diane Vaughan "normalisation of deviance"
* often changing the software being tested in the easiest solution
* designate particular end to end tests to a particular team to ensure they are written in the first place and that someone knows it's their job to fix it when they fail (Emily Bache)
* team that writes the tests must be close as possible to the code under test
* good to run tests in parallel where possible
   * selenium grid
* removing tests is a risk / reward trade off
* slow end to end tests increase the chance of pile up and increase cycle time
* avoid metaversioning the entire system, you'll slowly lose independent deployability
* end to end testing can explode with many services
   * explicit schemas between services make it easy to find structural breakages with unit tests but semantic behavioural changes need deeper tests
   * contract tests and consumer driven contracts CDC
   * a consumer writes tests that describe how an external service behaves, not testing your own service but verifying the behaviour of another
   * consumer team share these with the producer team and the producer team can use these tests to check it's service meets the expectations of all it's consumers
   * CDCs can help make the lines of communication between teams that should already exist more explicit, pair programming can be helpful for writing these contract tests
   * can detect breaking changes without expensive end to end tests
   * Pact is a tool for this, you can even use it to stub your collaborators (replacing a need for something like Mountebank), Pact Broker can store the version history of the specification
   * CDCs work best if you have good lines of communication between both sides, harder if one end is external
* end to end tests usually a blocker to fast testing and swapped with something else like CDCs, end to end tests almost like training wheels


### in production testing


* eventually adding more tests yields diminished return
* infeasible to catch all problems in a distributed system with tests alone
* tests are about giving confidence there is "sufficient quality"
* ping tests, smoke tests, canary releases, injecting bad data
* trade off between mean time between failure and mean time to repair
   * fast rollbacks and good monitoring reduces time to repair


### testing cross functional (nonfunctional) requirements CFR


* tracked at a service level (might be fine to have a lower uptime on a recommendation engine than the payment service)
* service level objectives SLO
* think about CFR tests earlier rather than later
* performance tests
   * performance testing particularly important in microservices where one request might involve multiple chains of synchronous actions
   * check core journeys through the system if nothing else
   * run scenarios and increase the number of clients to show how latency increases with load
   * might need production data (or similar volume of test data)
   * could run daily or weekly if not feasible
   * need to have targets to know what a good (or bad) result is
   * or at least check performance isn't worse than the last test
* robustness tests
   * tests to recreate failures to ensure services keep operating 
   * tricky to implement
   * artificial network time outs etc
* accessibility testing


## from monitoring to observability


* you thought monitoring a monolith in production was hard? prepare yourself
* tools can help but may need a change in mindset
* monitor the small things and aggregate the big picture
* "we replaced our monolith with microservices so that every outage could be more like a murder mystery" @honest_update
* aggregate and drill down on host metrics
   * is CPU high because of a service problem or a rogue OS process on one node
* ssh multiplexing for checking logs on multiple hosts
   * not very scalable but will work for small systems
* load balancing metrics
* observability
* understand the internal system state from external output
* the challenge is having the understanding to create and interpret these outputs
* monitoring requires advance thought and alerts which works for monoliths with anticipated errors but not so suitable for distributed systems
* observability is a property of the system, monitoring is a thing we do to the system
* be careful not to reduce the problem of observability to implementation details (eg. a blind focus on metrics, events, logs and traces), it's about understanding the system not adding lots of tools
* it's about events and knowing if something is happening that will make users unhappy
* monitoring a passive activity
* need to understand tools for monitoring and observing are also mission critical production systems
   * also a vector for risk and attack
* try to standardise across services
   * log line template
   * metric names
* tools should be
   * democratic -- consider the users who will be actually using the logs and data
   * not prohibitively expensive that you can't give enough licenses or use it in the test environment
   * easy to integrate -- OpenTelemetry
   * provide context
      * temporal (how does this event compare to another time period)
      * relative (how does it compare to other systems)
      * relational (what parents or children are there)
      * proportional (how bad it this)
   * real-time
   * suitable for your scale
      * Google dapper has essentially no features, allowing it to scale to 5B/rseq, you don't need this!
      * cost effectiveness, can you afford to scale it
* humans are still the experts of the machine for now
   * no magic AI or ML is solving understanding anomalies yet (identifying them maybe)
* Observability Engineering (Majors et al), Site Reliability Engineering (Beyer et al), SR Workbook (Beyer et al)






### building blocks for observability


* log aggregation
   * collection of logs across services
   * fetching logs and SSH multiplexing won't scale
   * vital for understanding, need to be accessible
      * author strongly suggests log aggregation is a prerequisite of microservices
      * implementation isn't that hard but can be a good canary project to work out if your org can actually handle microservices
   * most systems for log aggregation involve a local daemon reading local logs and forwarding logs to an aggregator
      * service unaware, needs no code or config changes
   * service, datetimestamp, log level and so on needs to be consistent across all logs to allow for common queries
      * some daemons can reformat logs but this adds stress to a system where it isn't needed, just make the log right (unless you literally can't - eg. third party or shit software)
      * no real common standard has emerged
      * just pick a format and standardise
   * use a correlation id in each log line as a standard field
      * generate on perimeter
      * can be hard to retrofit, design the system with them in mind!
   * be mindful of clock skew when trying to determine chronological order of log lines (they'll be wrong)
      * can't understand causality
      * can't get an idea of timing information for a flow of calls
      * logical clocks can help to an extend
      * distributed tracing tools much more powerful
   * implementation
      * FluentD Daemon, ElasticSearch, kibana
         * ElasticSearch has big overhead
         * ElasticSearch marketed as a database but it's just a search index
         * if you can't afford to use your logs, make sure they are backed up outside ElasticSearch in case you need to reindex
         * ElasticSearch has also changed to a closed license to give AWS the middle finger and upset a lot of people, except AWS has its own open fork
      * Splunk
         * eye wateringly expensive
      * Humio
         * focus on fast ingest and clever queries rather than indexing
      * Data dog
      * AWS cloudwatch, Azure app insight
   * logs can be VERY big across hundreds of services
      * hardware cost
      * scaling
   * logs can potentially contain sensitive information
      * access control
      * malicious parties
* metrics aggregation
   * collection of raw numbers for problem detection and capacity planning
   * secret to knowing when to panic is gathering metrics over a long period to understand patterns
   * want to store and report at different resolutions (think to those classic rotating host graphs)
      * aggregation at each resolution will lose some data
      * decide in advance what information is ok to lose
   * high cardinality data
      * service name, customer, request, etc
      * newer tools support queries but some don't
         * Prometheus open about this design decision
         * difficulties with time series databases
      * Charity Majors -- "a metric is a dot of data", "storing a metric is dirt cheap, but tags are expensive"
      * collecting more information allows you to ask more questions, questions you didn't even know you needed to ask
   * implementations
      * Prometheus a good replacement for Graphite for low cardinality data
      * Honeycomb and Lightstep for higher cardinality
* distributed tracing
   * tracking a flow of related requests
   * understand relationships between services
   * correlation IDs in log files is not that sophisticated
   * local activity in a thread is collected in a "span"
      * spans are collected
      * related spans form a trace
   * OpenTracing API, OpenTelemetry API
      * third party tools support these 
      * make sure to pick something that supports this standard
   * detailed example shows timing can go right down into mysql query time
   * collection has an impact on the system and needs to be sampled
      * challenge to collect just enough
      * Google Dapper -- random sampling
   * Honeycomb and Lightstep more clever, can sample more for a particular type of event, commercial toold
   * Jaeger emerged as popular open source tool
* are you doing ok
   * think of bees, hive health in a holistic fashion, one sick bee is not necessary a sick give
   * monitoring and alerts are a step away from the question of whether a system is working
   * need to define acceptable behaviour
   * "acronym city"
      * SLA -- service level agreement, bare minimum spec of what users can expect and what happens if the system fails to meet those expectations, usually a system exceeds it's SLA (eg. AWS EC2)
      * SLO -- service level objectives, SLA is often broad and cross cutting, teams sign up to objectives to provide an SLA, SLOs can also involve things that don't impact the SLA (internal goals)
      * SLI -- service level indicators, measuring whether the service is meeting objectives (eg. response times etc.)
      * error budgets -- clarity on what errors and error levels are accepted, giving room for some instability to make changes (eg. if your uptime SLO is 99.9pc, you get a few hours of downtime a quarter for free)
* alerting
   * need to get good at prioritising errors as the sources and frequency increases
   * alert fatigue
      * too many errors at once
      * errors that appear all the time
      * insufficient information to prioritize (eg. cascading failure)
   * Steven Shorrock "alarm design"
      * direct users to a significant aspect of a system that needs timely attention
   * EEMUA Engineering and Materials Users Association
      * relevant
      * unique (don't duplicate!)
      * timely (useless if delayed)
      * priority (operator should be able to triage)
      * understandable
      * diagnostic and advisory (needs to say what's wrong and help fix the error)
      * focusing
   * reduce which errors should be sent to operators in the first place ...
* ...semantic monitoring
   * what should wake someone up at 3am, what errors are serious, is the system behaviour as expected
   * define rules that determine the system is working (use the SLOs)
   * not low level "usage shouldn't exceed 95pc", higher level "new customers can register", "we sell 100 products an hour"
   * product owner needs to contribute
   * real time monitoring and synthetic transactions
   * need to be ready and able to emit data that's used for important business metrics (eg. sales data might be locked in a database, how will you make it available for live monitoring)
* testing in production
   * synthetic transactions
      * fake user behaviour injected into prod regularly
      * work on data that just gets dumped into a test queue or test user etc.
      * better indicator than low level metrics (but not a replacement)
      * already have most of the tools as you can write and run tests -- just run some subset of tests that add data
      * good for key operations (SLOs)
   * AB tests
   * canary release, parallel run
   * smoke tests -- test in prod before users
   * chaos engineering


## security


* larger attack surface but opportunity to defend in depth and limit access scope, a simultaneously more and less secure system
* only as secure as your least secure component
* types of security controls
   * preventative -- stop before it happens
      * good secret storage
      * encryption
      * authentication and authorization
   * detective
      * firewalls, IDS
   * responsive
      * automation to rebuild and restore
      * backups
      * communication plan
* agile application security (Laura bell)


### core principles


* principle of least privilege
   * give users and services the most minimal access
   * limit read and write scopes, limit resources
   * expiry
* defense in depth
   * breaking function into multiple systems
   * hosting services on different network segments
   * mix technology to limit impact of 0 days
* automation
* build security into the delivery process
   * testing, usability and operations were all previously looked at as blockers to releasing software, until DevOps
   * developers need basic understanding
   * probe the system in the test suite
      * ZAP
      * Brakeman, Snyk


### five functions of cyber security


* identify
   * threat modelling -- who might be after our stuff and why
   * make sure to focus on the right stuff and everything is in scope
   * easy to get distracted with heavy technical details
   * think from the outside in, not inside out
   * threat modelling (Adam Shostack)
* protect
   * more things to protect
* detect
   * more to watch in microservices, log aggregation and IDS
   * Aqua
* respond
   * understand the scope and reach
   * follow security and privacy incident and notification plan
   * may need to inform a data protection officer
   * companies often compound the damage of a breach at this step
   * GDPR requires notice within 72 h
   * blame culture will make understanding the incident impossible
* recover


### foundations of application security


* credentials
   * microservices, same number of humans but many more credentials
   * may be tempted to offer fewer broader keys
   * user credentials and secrets
   * theft of user credentials is one the top causes of a breach (including phishing and brute force)
   * Code Spaces destroyed overnight as their entire business was run through one AWS account with one API key
   * hackers scan for compromised keys to automatically set up bitcoin miners
* secrets
   *  certificates, SSH keys, API keys, databases
   * lifecycle
      * creation -- how to make it
      * distribution -- how to put it in the right place
      * storage -- how to protect it so it's only read by what should read it
      * monitoring -- knowing how and when the secret is used
      * rotation -- what's the plan to change the key and will it cause problems
   * how
      * kubernetes has a built in secret manager
      * hashicorp vault is sophisticated "Swiss army knife of secrets management", supports consul template to automatically change keys
      * AWS secret manager, Azure key vault
   * rotation
      * time limited API keys
      * can be painful
   * revocation
      * need a plan for what happens when you know a key has been compromised
      * may need a rolling restart of systems if the key cannot be dynamically changed
   * git-secrets commit hook, git leaks
   * limit scope
      * fine grained credentials
* patching
   * keep track of security notices on third party software you use and rely on
      * Snyk, GitHub code scanning
   * may want to use managed systems or managed kubernetes clusters to offload responsibility
   * Aqua can monitor containers to understand issues in the image
* backups
   * people say it more than they actually do it
   * better disks and databases make us feel safer but backups are still important!
   * don't need full machine backups as the state can be deployed automatically
   * backup logs too
   * avoid the Schrödinger backup, try to restore things
   * store cloud backups on another account and region or even provider
* rebuilding
   * ability to wipe an entire system and rebuild from scratch is very effective
   * use the same process to rebuild as deployment
   * can you rebuild the underlying container platform as well?


### implicit trust vs zero trust


* often there is implicit trust from calls inside the perimeter, but what about when attackers are inside the perimeter?
* zero trust is a mindset that just assumes you're always working in an environment that is compromised -- perimeterless
   * Jan Schaumann-- you can do some odd things, if you're already perimeterless you can be more generous with opening services to the internet as it's no less trustworthy than the internal network
* can make this decision for each service if you want, use your threat model to guide
   * can form zones based on data privacy policy
      * good defense in depth


### securing data


* data in transit
   * server identity
      * is this really the microservice
      * http with TLS (the S is for ssl which isn't used anymore) server certificate
   * client identity
      * is this really the right client
      * shared secret, client certificate
      * mutual authentication to confirm client and server certificate -- mutual TLS, good for certifying two services (not so much a user)
      * service mesh can handle this
   * visibility
      * who can see data in transit
      * squid and varnish reverse proxies can't cache with HTTPS requests
   * manipulation
      * who can change a request
      * HMAC to check data hasn't changed
* data at rest
   * data at rest is a liability
   * go well known
      * use existing encryption systems (eg. your database probably has its own encryption)
      * salted password hashing, use a standard
      * badly implemented security worse than having none
   * pick your targets
      * what to put in log files
      * computational overhead of encryption
      * limit to things that need to be encrypted
   * be frugal
      * scrub what data you can as soon as possible if you don't need it
      * don't need to worry about breaching data if you don't store it
      * store only what is absolutely required
         * do you need full names, full IPs?
      * decrypt on demand
   * keys
      * security appliance to store keys
   * backups
      * encrypt them if you encrypt production data


### authentication and authorization


* authentication -- are you who you say you are
   * username and password
   * checking the identity of the principal
* authorization -- map a principal to an action


* service to service authentication
   * mutual TLS
   * hashing client key to prove to server the key was valid
* human authentication
   * username and password
   * 2fa/mfa
   * SMS, magic links, yubikey, biometrics
* single sign on SSO
   * authenticate with identity provider 
      * open id
      * directory service, LDAP or AD
   * service provider decides to authorise
      * OpenID connect
      * OAuth2
      * SAML SOAP protocol is dead
* single sign on gateway
   * centralised behaviour for redirecting and handshaking identity provider
   * essentially a barrier that directs unauthenticated users to the identity provider before services can respond
   * can use http headers to store authentication information so services know who a user is
   * alternatively, JWT 
   * developers need to be able to launch a gateway for testing
   * can become a coupling point as more functionality gets added to gateway
* fine grained authorization
   * user roles
   * JWTs can be useful
   * use coarse grained roles based on the organisation
* confused deputy
   * upstream party tricks an intermediate into doing things that shouldn't be allowed
   * what if a user tricks the application into viewing someone else's account
   * user is authenticated but not authorized
* decentralised authorisation
   * microservices should make this decision, not a gateway
   * what if users alter http headers
   * how can you know what user is really authenticated
   * it's jwt again
* JSON web tokens JWT
   * store claims about a user in a passable token
   * token is signed to show it hasn't been altered
   * can be configured to expire
   * public claims (standard fields) should be used where appropriate
   * jwt token can be passed in http headers
   * example
      * user logs in, oauth token on device
      * jwt used at perimeter, valid for length of request
      * jwt passed to downstream services
      * service can validate token and use claims
   * jwt best solution in this space so far
   * need to think about key management, how will services be able to validate signature
   * need to get token validity right if you're using async processing (long tokens with special scope or just stop accepting the token for certain long action)
   * token can become unwieldy if you use a lot of custom claims (but you're probably doing it wrong)


## resiliency


* David D Woods, resilience engineering
   * robustness -- absorb perturbation
      * build mechanisms to accommodate expected problems in systems and people
      * requires prior knowledge of what might go wrong (or what has gone wrong before)
      * can increase complexity, it's a trade off
      * non software robustness
   * rebound -- recovery from an event
      * can spend a lot of time trying to eliminate the chance of an outage and then be caught off guard
      * give people a role, a playbook
      * backups
   * graceful extensibility -- deal with unanticipated scenarios
      * automation cannot handle surprise, need people in place to handle new situations
   * sustained adaptability -- adapt to changing environment
      * constantly adapt towards resilience
      * because you haven't suffered a catastrophic failure yet doesn't mean one can't happen tomorrow
      * balance between short term delivery and long term adaptability


### failure is everywhere


* "failure is a statistical certainty at scale"
* make sure to spend time thinking what to do when it happens to you
* how much failure can the system tolerate? don't overengineer in situations where downtime is tolerable
* what are the requirements
   * response time and latency
      * how does increasing load impact service
      * monitor and set goals for percentile
   * availability
      * is this a 24/7 service
      * can I rely on this service or not
   * durability
      * how much loss is acceptable
      * how long to store for
* degrading functionality
   * need to work our what services rely on eachother and have a plan to degrade when they are unavailable
   * think from a business context not a technical one
   * more complex than the binary status monolith
* slow services are worse than dead services
* failing fast is better than failing slow
* John Allspaw -- blameless post mortems


### stability patterns


* timeout
   * important to get right
   * too long and you can lock up the system, too short and you might fail requests in the tail that would have succeeded
   * defaults usually long (or disabled), consider the context, a 30s timeout on a service that populates a web page is pointless as no user will wait that long anyway
   * use healthy system information to pick good limits
   * log and review timeouts
   * overall operation timeouts, let systems just give up on requests
* retry
   * use http codes to determine if a retry is useful
   * think about the worst case total retry time
* bulkhead
   * Michael Nygard - Release It
   * separation of concerns
   * eg. use multiple different worker pools to limit the impact of a pool exhaustion
   * preventative -- avoids resource contention in the first place
   * load shedding -- reject requests to stay afloat
* circuit breaker
   * try to mitigate cascading failure by isolating parts of the system away from eachother
   * "blow the breaker" when a service starts acting unhealthily, new requests fail fast and avoid a cascade
   * what is a failure? up to you, http 5xx codes are a start
   * how to start sending when it's healthy again
   * async circuit breaker can just queue new requests
   * also useful to have in place for maintenance
* isolation
   * strive to logically and physically isolate services
   * improves robustness at a cost
* redundancy
   * increase bus factor
   * tolerate service failure
* middleware
   * message brokers can offload some robustness concerns, sometimes
* idempotence
   * useful for message replay
   * easier to use events and messages as they can be delivered twice without concern
   * business operation is idempotent, anything else is up for grabs


### cap theorem


* consistency -- each request gets the same answer
* availability -- each request gets a response
* partition tolerance


* systems that cede consistency to keep AP are eventually consistent, all nodes will see updated data but not right away, accept that some users will see old data
* multi node consistency probably the hardest distributed problem
* Friends don't let friends write their own distributed consistent data store
* you can't sacrifice partition tolerance as your application just won't work on a network! there are no distributed CA systems
* more nuanced than AP and CP in real life
* focus on CP can sometimes miss the point, most consistent systems aren't completely consistent with real life anyway


### chaos engineering


* Netflix
* "the discipline of experimenting on a system in order to build confidence in the system's capabilities to withstand turbulent conditions in production" - Principles of Chaos
* more than just turning some machines off
* game days
   * surprise and test participants with a realistic outage
   * probe suspected weaknesses in systems and teams
* production experiments
   * Netflix purposefully incites failure with simian army
* chaos toolkit from reliably
* gremlin




## scaling


* improves performance and or robustness
* Martin Abbott scaling cube


### four axes of scaling


* vertical -- bigger machine
   * vertical scaling a go-to and much easier now with virtualized and cloud computing
   * horizontal scaling might be appropriate if you've hit your limitations
   * fast to implement on cloud now
   * although many pieces of software don't gracefully scale to more cores and can waste it
   * does not improve robustness
* horizontal -- multiple things to do the thing
   * duplicate part of the system to handle more work
   * most obvious implementation is spinning up multiple instances behind a load balancer
   * competing consumers pattern, several workers take jobs off a queue
   * read replicas to reduce read load
   * quite transparent, doesn't tend to need changes to application
   * requires more infrastructure to worry about
   * may need load balancer with sticky sessions
* data partitioning -- division of work by data attributes
   * assign workload a key, apply a function and get the partition shard the work goes to
   * can partition at a database and/or even service level, can use headers to allow a proxy to direct work to the right worker
   * scales well for transaction workloads
   * can do rollouts on a per shard basis
   * geographic partitioning can be useful to keep data within certain jurisdiction
   * does not improve robustness
   * need to get the right partitioning key to ensure even load and avoid hot spots
   * querying across data shards can be hard, may need to ETL to a full read replica or use async mapreduce
   * nosql distilled (sadalage and Fowler)
* functional decomposition -- division of work by type of work
   * extract specific functions to scale particular workloads independently
   * effectively making microservices!
   * can assign hardware at a much more granular level, scaling certain functions up or down -- "rightsize infrastructure"
   * opens up opportunities to build a new system that can handle partial failure
   * migrate to languages or tech stacks more suited for the specific workload
   * biggest impact, significant work and adds complexity
* will likely use a combination of these (hence scale cube)
* optimisation should be driven by a real business need
   * experimentation is essential to confirm the thing needs optimising in the first place and that your proposal addresses it
* author suggests CQRS is one of the hardest scaling measures to get right


### caching


* cache of origin data, cache hit or miss
* why
* for performance
   * reduce load and latency
   * expensive queries can be caused until invalidated
* for scale
   * eg. avoid contention of database with read replicas
   * useful in any situation where there is resource contention
* for robustness
   * can operate even if the origin is unavailable
   * will need to avoid evicting stale data
   * favours availability over consistency
   * can crawl your website into a static backup to use in an outage (the Guardian)
* where 
   * client side
      * cached outside origin scope
      * eg. kept in the other microservice where the data is used
      * avoid network calls
      * can get inconsistencies with multiple systems caching the same thing but can improved with shared redis or memcached (but will need network call and who manages it?)
   * server side
      * origin keeps a cache on behalf of its consumers
      * better options for invalidation (write through cache)
      * different consumers won't see different data
      * could have a reverse proxy, hidden redis node, divert read queries to replica transparently
      * round trip still needed
      * requires the service to be working to serve the cache so decreases robustness
      * will improve performance of all consumers though, so can improve scaling
   * request cache
      * best way to optimise for speed
      * highly specific
      * cache hit means no web request or database hits in other services
* some options for invalidation
   * TTL 
      * simple but blunt
      * Cache-Control, Expires http headers
      * origin can give hints to clients for how long to cache things
      * depends on your tolerance for stale data
   * conditional GET
      * ETag entity tag
      * If-None-Match http header
      * returns 304 not modified or 200 ok
      * useful if the cost of crafting a response is high, doesn't reduce round trip time
   * notification
      * use events to tell subscribers that data has changed and they should update their cache
      * much more complex than something like TTL
      * reduces window of stale data
      * use a heartbeat so clients can gracefully fail (warn users data is stale, turn off caching or aspects of the service entirely)
      * if you can, send the new data in the event or all the subscribers will have to go and get it
   * write through
      * server side cache updated at the same time as the real origin
   * write behind
      * cache is a buffer for the origin and is updated first, then the origin
      * potential for loss if the cache isn't durable enough
      * more straightforward forms of caching are good enough
* finally
   * don't cache too much, fragments state and each cache makes the system harder to reason about
   * nested caching can cause stale data to persist after it really should have expired
   * "ideal number of caches is zero", these are optimisations that you need to prove are needed
   * balance between freshness and performance, risk of choosing such an aggressive invalidation scheme that you lose the benefits of caching
   * be careful with Expires: Never


### autoscaling 


* need to know how fast it takes to bring an instance up
* if you have autoscaling rules, have a test suite to try them out
* easiest to start with autoscaling on failure conditions, then load


## user interfaces


* think holistically about digital, web, mobile and desktop are weaved together
* decomposition of functionality isn't just for the backend
* dedicated frontend team isn't an inevitability
* retain cohesion between services and how those services are presented to the user
* balancing act of not putting too much behaviour in the UI


### ownership models


* traditional layered architecture can cause issues with trying to deliver software efficiently
* better to break systems into components under the end to end ownership of a single team for independent deployability
* drivers of dedicated frontend teams
   * lack of specialists
      * temptation is to stick all the ones you've managed to get together
      * deprives other developers the opportunity to learn from the specialist
      * can encourage developers to use centralised resources poorly and misunderstand how they work
      * embed the specialists
      * alternative - enabling team is like an internal consultancy team
   * consistency of look and feel for UI
      * shared resources, styles guides, UI component catalog
      * can lead to disjointed user experience if you're not careful, or if you want to prioritize speed of delivery
   * technical challenges
      * single page applications harder to break apart
      * apply good patterns
* towards stream aligned teams
   * align teams to end to end functionality
   * reduce handoff points
   * gives team a better idea of the user rather than "backend" teams just vaguely working to some spec
   * harder to understand if your contribution is successful the more distant you are from the end product and users
* GraphQL
   * clients issue queries to access or mutate data
   * can reduce round trips
   * clients can ask specifically for the data they want rather than everything
   * needs a resolver to handle queries
      * maps explicit types and IDs to calls to microservices
   * 

### patterns


* monolithic frontend
   * all UI state and behaviour is in the UI
   * calls to backend microservices to fetch and update
   * typical for SPA with dedicated team
   * encourages a centralised team
   * can be harder to tailor for other user experiences (all backend calls basically the same)
* micro frontend
   * Cam Jackson -- an architectural style where independently deliverable frontend applications are composed into a greater whole
   * essential pattern for stream aligned teams
   * independent deployability for the front end
   * widget based or page based decomposition
   * cross cutting concerns can be harder to get right
* page based decomposition
   * UI broken into pages served by different services
   * common navigation to stitch them together
   * authors default choice for a website UI
   * move toward SPA making the "traditional" linked web rarer
* widget based decomposition
   * can be used with or without page based decomposition
   * can align well with teams that own multiple services
   * more flexible than page decomposition
   * helps breaking out an SPA
   * container application needs to hold the core interface that weaves the widgets together
   * Spotify uses this pattern
   * avoid iFrames, rather use server side templating or dynamic insert on client
   * can lead to duplication when dependencies are involved (eg. multiple imports of react or bootstrap of different versions), bloat, slower load time
      * use tooling to keep track of dependencies, size and load time
   * widget communication -- emit events
   * web component standard was supposed to help all this but stalled somewhat
* central aggregating gateway
   * sits between external UI and internal services
   * filters data and aggregates calls to prevent sending too much to the client 
   * specific endpoints to organise the necessary calls to all involved services and return the minimal data required for the client to do its job
   * useful for batching requests, reducing bandwidth
   * could be hard to get the ownership right, needs to be close to the frontend team but they'll need the skills to build it
      * risk of delivery bottleneck
      * BFF pattern can help
   * can become bloated if calls exist to support different types of application (eg. web, mobile)
   * avoid it becoming generic
   * basically just use the bff pattern
* backend for frontend BFF
   * single purpose specific UI
   * used at SoundCloud
   * basically a central aggregating gateway but only supports one specific format of the UI for one application
   * coupled strongly to specific UI experience under the assumption the bff is maintained by the same team
   * has server and client side components
   * can split by client type (mobile) or even specific clients (eg. android)
      * use your organisation structure
   * may lead to duplication but that isn't necessarily bad
      * extracting shared code can lead to coupling
   * if you find there is shared code doing business stuff then you might want to extract it into a new service entirely
      * "creating an abstraction when you're about to implement something a third time"
   * BFF can work with web experience too, generation of server side templates which can also be cached with reverse proxy
   * can even be used to present an API to specific third parties
   * best choice for cases where there are discrete user experiences
   * can use GraphQL to remove some of the heavy lifting as it's easier to add and remove fields from the queries without changes in the microservice


## organisational structures


* ignore your company org chart at your peril
* a shift to microservices without a corresponding shift in the organisational structure will deliver less value
* Accelerate (Forsgren, Humble and Kim)
   * a team is loosely coupled if it can do X without another team
      * make large scale design changes
         * without permission
         * without depending
         * without creating work
      * do work without communication
      * deploy and release on demand
      * test and use a test environment on demand
      * perform deployment in normal hours
   * power and accountability is decentralised
   * can achieve better delivery performance (tempo and stability) with less burnout and pain
   * can grow an organisation and still improve productivity linearly
* Conway's law still holds true
* Amazon's infamous "two pizza team" -- teams should be small enough to be fed by two pizzas
* Netflix seats teams with closely linked services close together
* Rodriguez et al -- productivity decreased in teams eq or larger than 9
* Brooks' law -- adding manpower to a late software project will make it later
   * tasks are usually not simple subdivided, shared or parallelizable
* coordination is the biggest cost to efficiency
* Robin Dunbar -- 150 before a group collapses under itself
* Spotify's initial "guild and chapter" model
* ownership models
   * strong
      * a service is owned by a team
      * other teams need to ask or pull request changes that could be rejected
      * strong ownership, less coordination
      * full life cycle ownership is strongest
         * no ops team, no external signoff
         * aspirational, will take years
   * collective
      * free for all
      * will require some level of coordination
      * useful if your delivery bottleneck is number of people as you can move developers around more freely
      * needs more consistency in tech stack and ways of doing things
         * reduces your options
* principal engineer is a new fancy term for architect
   * often found in an enabling team
* community of practice
   * Emily Webber -- building successful communities of practice
   * "create the right environment for social learning"
   * foster a culture of finding the right way to do things
* enabling teams more like a full time role whereas cop more of a forum with a larger more fluid membership
* steam aligned teams need self service tooling
   * often drives adoption of things like kube
   * give teams bandwidth to actually deliver features
* platform team job is to make developing and shipping functionality easier, ultimately making developers lives easier
   * Devs are the user/customer
   * need to be open to feedback
   * what problems are people facing and how can we help them 
   * "paved road" concept, make it easier to get developers to the destination -- incentive for the platform teams to make sure the platform is used!
* try and make the way you think things should be done as easy as possible to adopt and don't prevent people from doing it a different way, let then choose the paved road or not
   * people will always find their way around if that is what they want to do
   * don't force people to use the platform, make them want to use it
* shared microservices
   * important to understand the drivers that make people share services before trying to fix it
   * cost of splitting might be too high
   * cross cutting changes can be painful
   * cost might be worth it in the long run
   * need to align infrastructure and teams
* internal open source
   * core committers making sure changes are sensible and high quality
   * whether you are a good gatekeeper or a bad one, this model uses lots of time
   * ideally the service would be quite mature first
   * a team with lots of PRs needs a check in, perhaps the service is in reality shared 
* pluggable modular microservices
   * code used by many teams may need to be turned into some kind of pluggable framework
   * take care that such things do not become bloated
   * alternatively have the components each formed as a library and the core team still owns the deployment
* review
   * Accelerate - teams that required external review had lower performance
   * peer review is better, someone else from the same team is going to know what a good and bad change is
   * reviews should be done as soon as possible in as much synchrony as possible to reduce context switching, pair programming valuable
   * ensemble programming (mob programming) potentially useful for large tricky problems
* geographic distribution
   * increases cost of coordination
   * will need to be considered when drawing team and software boundaries
   * reduce async communication as much as possible
   * can encourage good decoupling and modularity, especially if the organisation structure is working well for the company
* people
   * "no matter how it looks at first, its always a people problem" -- second law of consulting
   * this isn't the monolith world anymore
   * understand your staffs appetite for change
   * more important than ever to be clear about people's responsibilities
   * this can be a scary journey but will be a disaster from the start without the right people


## the evolutionary architect


* our industry is still so young
* hard to know what good means
* should see and understand the whole system and understand the forces acting upon it (developers and customers)
* deployed software isn't a never changing artifact, we must react and adapt to users
* Erik Doernenburg says it's more like town planning, optimising the layout of cities to suit the needs of today while thinking about tomorrow
   * more abstract than "build this here" they allow for local decision making with zones
* avoid the urge to overspecify
* facilitate the creation of a team API
* information hiding at microservice and microservice zone (team) level
* architecture is what happens, not what is planned. reflects the reality of what can be achieved, requires many people and many decisions
* Comcast architecture guild modelled on IETF
* Frank Buschmann habitability, comfortable and confident changes
* all about picking tradeoffs and there are plenty in microservices
* a principled approach
   * strategic goals
      * high level non technical goals
      * usually where the company is trying to go
      * goal as architect is to align technology decisions to these goals
      * will need to spend time with "the business"
   * principles
      * rules you've made to align to goals
      * eg. groups own their whole lifecycle or all components must be portable
      * ten maximum is good target
      * eg. herokus 12 factors
   * practices
      * how to ensure principles are followed
      * low level and technology oriented
      * coding guidelines, log locations, HTTP/REST communication style
      * can be changed but more likely to change than principles
* guiding an evolutionary architecture
   * ford, parsons, kua - building evolutionary architectures
      * fitness functions
      * collect performance data to assess behaviour meets requirements and software is achieving fitness
* architecture in stream aligned organisation
   * many responsibilities of the architect are enabling
   * enabling team is a good place for architect in stream aligned teams or even an enabling team of architects
   * augment small team of architects with developers from other teams on rotation
   * on disagreement: "done my best to convince people but ultimately I wasn't convincing enough"
      * knowing when to go along with a decision you don't agree on is one of the skills of an architect
      * need a grasp of when your team is about to do something catastrophic
   * what about teams making decisions that are undermining our principles
      * have a chat
      * agree on taking on some level of technical debt they are responsible for
      * perhaps they can be convinced
   * non technical product owners should be accountable for technical aspects to force them to understand the system and technology principles
* building a team
   * help people grow and be part of the vision
   * encourage people to step up and take ownership of a service
* required standard
   * what is a good and bad citizen in your microservice world? what should they do to make sure they are manageable
   * need to decide how much variability you allow in your system, some things will probably be quite tightly defined across services
   * monitoring
      * need a uniform system wide view
      * logging and health metrics should be standardized across all services
   * interfaces
      * not just deciding on http and rest but will you use verbs? how will pagination work? versioning?
   * safety
      * circuit breakers and bulkheads
      * agree on http codes to ensure error handling works uniformly
* governance and the paved road
   * COBIT
   * defining how should things be done, making sure people know how things should be done and making sure things are done that way
   * fundamentally a group responsibility
   * set your principles in a governance group if appropriate
   * tracks and manages risk
   * paved road is a useful concept to getting people to follow the way things should be done
* make it easy to follow the paved road
   * exemplars
      * real world microservices that embody your best practices
   * tailored microservices template
      * make it easy for people to get started
      * care must be taken to not destroy morale by forcing a mandated template
      * frameworks risk becoming a monolith in their own right
   * the more stacks you support the more resources you have to provide to maintain a repo of good practice
   * service mesh possibly useful here
* technical debt
   * often need to make a conscious decision to cut a corner
   * short term benefits have long term costs
   * good to have a view of what the debt is and how to pay it off -- debt log
* exceptional decision handling
   * keep a log of decisions that deviate from principles
   * might make sense to change a practice or principle
* summary of architect responsibility
   * vision
      * clear communication of the technical path for the system that helps an organisation meet the requirements of itself and customers
   * empathy
      * understand the impact of your decisions on customers and colleagues
   * collaboration
      * engage with as many peers as possible to define and refine and execute the vision
   * adapt
      * keep the vision aligned to requirements of customer and organisation
   * autonomy
      * find the right balance between standardizing and enabling team autonomy
   * governance
      * ensure the system fits the vision and make it easy for people to do the right thing
* constant balancing act
* trick is to know when to push back or go with the flow!
* Gregor Hohpe the software architect elevator


## bringing it all together


* microservices are a service oriented architecture with a focus on independent deployability
* information hiding helps keep interfaces stable by only exposing the bare minimum
* DDD helps find microservice boundaries
* monolith is a sensible starting point
* request response vs event driven communication styles
* event driven collaboration helps build a loosely coupled architecture but at the cost of making it harder to reason about the system
* use sagas to explicitly model business processes
* orchestrated sagas probably best when a single team owns the whole saga otherwise use choreographed sagas
* every service should have its own CI pipeline
* kubernetes good when you have lots of stuff
* FaaS is an interesting emerging style of deployment
* separate deployment and release
* reduce reliance on end to end tests
* use contract and schema checking between services when testing
* monitoring is a passive activity, observability is understanding a system from its outputs
* use parallel runs and synthetic transactions to pick up problems that will affect real users
* JWTs can decentralise authorisation logic
* operate with a zero trust mindset
* resilience -- robustness, rebound, graceful extensibility, sustained adaptability
* much of resilience is also influenced by team organisation and culture
* scale the easy way first (vertical and horizontal)
* UI should not be an afterthought
* backend for frontend pattern help aggregate calls and design more specific user experiences
* enabling teams have a cross cutting focus and support the work of many stream aligned teams
* the "platform" is a paved road
* architecture is not fixed and unchanging
* see also
   * accelerate - forsgren, humble, kim
   * team topologies -- Skelton and pais
* more options, more decisions
   * keep decisions small so it's not so bad when you get them wrong
* journey not a destination! go incrementally