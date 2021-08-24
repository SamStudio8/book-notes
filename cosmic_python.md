# Architecture Patterns with Python


Test driven development, domain driven design and loosely coupled services are three tools for managing complexity. But how to get the best from your tests? How to build models that don’t get cluttered with implementation details? How to make services fit with frameworks?


Systems tend toward chaos, picking up cruft and edge cases as they age. The “big ball of mud” antipattern is the natural state of software, much the same way that wilderness is the natural state of a garden. It takes energy to prevent this.


Encapsulation and abstraction are the tools to push against chaos. Encapsulation simplifies behaviour and hides data. It is helpful to think of the “roles and responsibilities” your code has, rather than its tasks. This allows for abstractions that don’t think about the tasks or algorithms. See CRC (class-responsibility-collaborator cards).


## Architecture for Domain Modelling


* Repository: abstraction for persistent storage
* Service Layer: defines the beginning and end of use cases
* Unit of Work: atomic operations
* Aggregate: enforce data integrity


Always think about the cost and benefit of each architecture choice.


### Domain Modelling


* DDD (domain driven modelling) says the most important thing about software is it provides a useful model of a problem (the domain). If we get the model wrong, it becomes an obstacle to development. Jargon (ubiquitous understanding in DDD) used by stakeholders is distilled understanding and should be understood.
* `dataclass` - introduced by https://www.python.org/dev/peps/pep-0557/. TLDR, seems to add a nice __init__ and a bunch of useful “dunder” (double underscore https://nedbatchelder.com/blog/200605/dunder.html) methods to a class. See also https://github.com/python-attrs/attrs (more extensive, attrs is particularly good for runtime validation https://www.revsys.com/tidbits/dataclasses-and-attrs-when-and-why/). Frozen dataclass or attrs will generate an error when trying to assign a value to an instance of that class (immutable).
* `typing.NewType` will even allow you to wrap primitive types to create your own new types (for example you may have a NewType(“SampleId”, str) type to ensure that you’re only passed true SampleId’s rather than any old str.
* Business concepts that have data but no identity (ID) use the Value Object pattern. A Value Object is defined uniquely by the data it holds. Dataclass and attrs give us the eq dunder (__eq__) for free.
* Entities are data objects that have a persistent identity, and usually implement their own dunder eq and dunder hash. For value objects __hash__ should depend on all the attributes and the dataclass should be frozen to prevent changes to them. For entities, the simplest workaround is to set __hash__ to None to prevent it being hashed into dictionaries and so on, otherwise, it should rely on a fixed reference that cannot change.
* “Sometimes, it just isn’t a thing”. Domain service is essentially a standalone function that doesn’t fit into a value object or entity. Verbs should be domain service functions.
* Magic methods (dunder) can be added to Entities or Value Objects. For example, changing __gt__ allows them to be sorted.
* Exceptions should be domain orientated too.
* Domain model is where the value is. The model is the closest to the business. It should be easy to understand by all stakeholders and easy to modify.




### Repository Pattern


* ORM code ends up littered in your pristine model, you end up with an ugly dependency on something else instead. SQLAlchemy has a “classical mapper” which allows you to map your model onto the ORM yourself (Django does not have this).
* The Repository pattern abstracts persistence as if everything was being stored in memory. Effective provides a get() and add() function to get/add entities via the ORM.
* Building a Fake Repository is easy, just wrap a set or dict!
* You can think of AbstractRepo as port and ConcreteRepo as adaptor
* If it’s hard to fake, the abstraction is probably too complex
* abc.abstractmethod is what makes ABC actually work in python. Use pylint to help you enforce this sort of stuff. Often people ditch abc classes as they become a bit of a mess to maintain, there are alternatives, see https://www.python.org/dev/peps/pep-0544/
* Simple CRUD apps don’t need to go to the complexity of a Repository pattern
* Thinking about a Majora Repository, could handle the fetching of entire tables and munging them into Runs and so on. Means the Tables don’t have to reflect the real Entities!


### On Coupling and Abstractions


* Think about the distinct responsibilities your code needs to take -- what can be abstracted into interfaces
* “Given my abstraction, what abstraction of actions will occur” -- think about functions that return “commands” of things to do by the rest of the program
* A “core” of code that has no dependency on external state that is fed actions -- “functional core, imperative shell”
* Passing around stateful components is often referred to as “test-induced design damage”
* Spy object stores a list of actions for inspection
* Book argues that patching libraries are “code smell” -- and don’t help contribute to better design (recall the --dry-run example), design for testability and extensibility -- TDD as design first and test second


### Service Layers


* What is your app doing? A lot of orchestration, fetching, validating, error handling, committing.  These could be split out to a new layer -- orchestration layer, or service layer.
* Imagine a Flask app -- its just should be per-request session management, parsing, handling status codes and JSON
* Take anything service-y away from the app and abstract it to a service layer
* Note the difference between a Application Service (the service layer) and a Domain Service -- service layers drive the application (get, update, persist) and domain services handle important business rules that don’t belong in a Value Object or Entity (eg. a tax calculator could be a stateless TaxCalculator class, or just a calculate_tax domain service function).
* Goal is to write many small “fast” tests with few, slow, big end-to-end tests
* Service layer depends on Model and AbstractRepo (“port”)
* Tests can implement a FakeRepo
* Some people advocate for pushing service to models -- Fat Models, Thin Controllers


### High and Low Gear Tests


Low gear, low level tests against the domain models at the start for best feedback. High gear testing against the APIs and services offer more flexibility at the cost of some feedback.


Testing pyramid. Goal is to have as few E2E tests, then integration tests, then lots of unit tests.


Aim for one E2E test per feature to show all the moving parts work together. Service "edge to edge" tests offer a trade off between coverage, run time and efficiency. Small core of tests against domain model. 


Errors should bubble up to entry points, test them as a feature.


Express the service layer in terms of primitives, rather than domain objects.


Every line of test code is a blob of glue that makes things harder to change.


### Unit of Work


* If the Repository is the abstraction for persistent storage, UoW is the pattern for abstracting atomic operations. 
* The UoW is the single entrypoint to the persistent storage and keeps track of what objects were loaded and their latest state.
* The UoW and Repository are “collaborators” in the object modelling sense, so its ok if they are quite tightly coupled
* UoW provides commit() and rollback() and a member to access the Repo
* __enter__ and __exit__ are magic members that execute when moving into and out of the `with` block
* contextlib provides more functional stuff for contextery stuff
* Unit of Work concrete __enter__ will set up database session and Repository
* Don’t mock what you don’t own, here they mock the UoW rather than the Session
* UoW gives strong explicit control over the start and end of transactions
* However many ORMs have their own good abstractions of atomicity, you could even pass the SQLAlchemy Session around for a while until your app grows out of it
* If you want to write your own UoW, you need to think about rollbacks carefully - if you don’t know what you’re doing just stick to your ORM to KISS


### Aggregates and Consistency Boundaries


* Domain logic enforces constraints, invariants are things that have to be true when an operation is finished
* Concurrency makes invariants harder to enforce, enter Locks; there's no reason you have to worry about all invariants at the same time
* Aggregate is a pattern from DDD, it just contains a bunch of other Domain objects
   * ...a cluster of associated objects that we treat as a unit for the purpose of data changes (Eric Evans blue book), the Aggregate encapsulate access to the cluster
* Choice of Aggregate is somewhat arbitrary, but important - it will form the boundary around which we check for consistency
* The “Product” introduced here doesn’t look like what you’d expect, this is fine:
   * Bounded contexts (blue book)
   * “Customer” means different things to different parts of the business, no reason the code can’t represent this too
   * Attributes in one context are irrelevant in another
   * Better to have several models (even with the same name) with safe boundaries and handle the translation between them explicitly
   * This translates particularly well to microservices
   * As a rule of thumb, domain models only need the data they use
* Once you introduce Aggregates, they should be the only entrypoint to access those domain models (eg. Repositories should return Aggregates)
* Be comfortable changing your Aggregate if it leads to garbage performance, its there to manage technical constraints but must be balanced with performance and consistency
* Version numbers can be a way to manage race conditions, both queries read the same version number but only one gets to save the update
   * This is optimistic concurrency where things will probably be fine in the event of a concurrent update and we just have to detect it
      * https://www.postgresql.org/docs/13/transaction-iso.html#XACT-REPEATABLE-READ
   * Pessimistic concurrency is more aggressive and usually involves database locks (table locks or SELECT FOR UPDATE etc
* Good to use pythons public and “private” methods to allow Aggregate objects to “hide” things that are supposed to be internal


## Event driven architecture


* If you’re not careful with microservices you’ll build the one thing worse than a big ball of mud: a distributed big ball of mud


### Events and the message bus


* Why is all this testability and expressiveness worth the effort
* It’s not the obvious features that mess up our codebases, it’s the “goop”: reporting, permissions, notifications and other cross-cutting behaviour
* Fast solutions to these problems are the things that lead us down the road to a big ball of mud
* Imagine a notification for an out of stock item, it doesn’t belong in the controller (its just for serving HTTP), and models shouldn’t depend on infrastructure concerns, perhaps in the service? 
   * All these violate the SRP
   * “If you can’t describe what your function does without using words like ‘then’ or ‘and’ then you might be violating SRP”
* “Changing from orchestration to choreography” (Ed Jung)
* Introducing
   * Domain Events
   * Message Bus
* Event is a kind of value object, pure data structure with no behaviour
   * dataclasses good for this
* When a domain model records a fact has happened we say it raises an event
* These help address a code smell, Exceptions should not really be used for control flow (they are exceptional)
* Model can hold a list of Events, Message Bus is a dict that maps events to handlers
   * This is different from stuff that goes into external events (eg. Celery)
* Multiple options for implementation (of course)
   * 1) Simples - Service layer takes Events from the Model and puts them on the Bus
      * try, finally, messagebus.handle after UoW committed
   * 2) Service layer raises own Events
   * 3) UoW published to Bus
      * Already has a try, finally; and knows all the Aggregates
      * After committing, we run through all the objects in play and push their events to the Bus
      * Repository now needs to track loaded Aggregates with seen attribute set
      * Service layer kept clear of event handling
* “When X do Y” is a user’s way of telling us about an Event our system should handle
   * Events as first class things helps keep code testable and observable
* Must be careful to avoid loops in handlers
* Hard to know where all the code to handle a request is when handlers are in play
* Events can be used for eventual consistency; cancelled order? Send an event to a handler that goes and adds the stock for each


### Message Bus


* Situated software (Rich Hickey) is software that runs for extended periods of time, it's tricky to write as it needs to handle unexpected real world events 
* If everything is an event handler then we remove the distinction between external and internal events
* Preparatory refactoring: “make the change easy, then make the easy change”
* Coupling to Event data classes rather than primitives, but this is OK as the Events are not supposed to change often, and gives us a place to handle validation
* Event handler takes the UoW, pops the first event, collects new events and repeats until the queue is empty


### Commands


* Conceptual wart in that all API endpoints spawn internal events on the bus
* Commands are a type of message that capture a very specific intent (with a specific outcome), when they fail we want specific error information - named with imperative mood verbs like “allocate this”
* Events are different in that they are broadcast to everyone listening - named with past-tense verbs like “order allocated”
* Events usually spread knowledge that a command has been successful
* Handler now takes a Message which can be a Command or an Event
   * Message = Union[commands.Command, events.Event]
   * handle_command and handle_event helper funcs
   * handle_event catches Exceptions and writes to a log
   * handle_command raises Exceptions
   * Commands only have one handler, but Events can be handled by multiple handlers
* Commands should modify one aggregate and succeed or fail in totality, Events can handle eventual consistency and do book-keeping, cleanup and notifications -- these Events do not need to succeed for the Command to succeed
   * Imagine an order VIP system, the only thing that needs to work is taking the order; not sending them a welcome email or other cruft -- this is what the customer and business care about the most
   * Note that these events nicely delineate themselves around the start and end of business processes
* Dataclasses print nicely to the terminal/log with code to recreate them
* tenacity is a python library for common retry patterns
   * UoW and Command Handler mean that attempts will start only from consistent states
   * Retrying failed event operations is probably the single best way to improve the resilience
* This pattern requires good monitoring, everything is designed to fail neatly in isolation but you’ll want to keep good track of those failures


### Integrating microservices


* Takes the message bus further, exposing it to receive external events from Redis or some other broker (this is exactly what I was thinking about getting Majora to do!)
* Distributed systems suffer from connascence, either in execution (knowing what the right workflow through the system is) or timing (all components must be working at the same time to get the desired result)
* Goal is to replace these elements with connascence of naming (things only need to agree on what events are called and what they look like)
   * More on http://connascence.io/
* When breaking systems into components; Think in verbs, not nouns -- there are not systems for Orders or WarehouseBatches; there are systems for Ordering, Allocating
   * Helps determine SRP
   * Ordering only cares Orders are made, allocation is a later problem -- “everything can happen later, so long as it does happen”
* We can accept eventual consistency so don’t need to rely on synchronous calls
* Instead of tightly coupled HTTP API calls, we have async messages
* Independent failure modes mean the system is more resilient
* Of course there is complexity in choosing the right message platform (ordering, failure handling, idempotency)
* Pub/sub listener is just a wrapper for moving messages onto the internal Bus and back
* Outbound events should be validated before sending
* Distinguish between internal and external events too
* Events can be much harder to debug -- the flow of information is no longer explicit


### CQRS (Command-Query Responsibility Separation)


* Reads and writes are different, so should be treated as such
* Functions should modify state or answer questions, but never both
* Access pattern for reads is different (and usually more throughput)
   * Reads can be eventually consistent to perform better (eg. maybe stock information isnt real-time)
* “As soon as you have a web server and two customers, you have the potential for stale data”
* The service layer, UoW and domains give us nothing toward reading data
* APIs should reply with 201 Created (or 202 OK) and a Location header of the data
* The book creates a views.py in which it uses raw SQL (grim) to handle read-only query endpoints…
   *  it shows that adding the helpers to the Repository is not so straightforward
   * ...but the ORM could do this? 
      * The SELECT N+1 problem (although SQLAlchemy is quite good at avoid this with eager loading)
   * …”time to completely jump the shark” - no FKs, no types, just strs
* “Your domain model is not optimised for read operations”
* Common to move read-only state to denormalized views, read replicas or cache layers (was thinking of all this with Majora too!)
* Keeping the read model up to date is the challenge, views and triggers limit to the database itself, we can use events!
   * Event for add_x_to_read_model just runs raw SQL to add and remove data from read store
   * It still fails independently, bugs only break views into data
   * We can rebuild entire views from the write side
* Repo is simple and consistent but complex queries will lead to a lot of goop
* Custom ORM queries allows re-use of DB/models but sprinkles ORM specifics
* Raw SQL offers fine control but changes to the schema will impact queries (but likely will for your ORM too)
* Read store copies are easy to scale and need only be updated when changes are made, but it's a complex technique with a lot more work


### Dependency injection


* Regarded somewhat with suspicion in Python
* Adds a bootstrap script to clean up entry points and passing around of objects
* Called composition root in OO
* mock patch is how many dynamic languages get around changing explicit dependencies for testing but gets cumbersome with larger code bases
* mocks also tie you to the names of imports
* Putting the responsibility for passing around the dependencies to the right handlers in the bus is a violation of the SRP
* But what controls resolving the right dependency then? Enter the bootstrapper
* Achieved in python with partial functions, bootstrap can return the target function with the right dependency arguments for calling later (you can use functools.partial but a nested def gives better stack trace)
* This uses late binding closures
* Alternatively rewrite all the handlers as classes, and override __call__ with dependencies handled by __init__
* All orchestrated by bootstrap script, you can set up ORMs, logs and so on, you can use default args for production defaults
* Bootstrap script example returns the message bus, preloaded with all the right handlers - bus is also a class now
* Extra thought required for a thread safe message bus… not discussed here unfortunately
* inspect.signature used to cleverly inject the right dependency, but you could just as easily unroll it all and manage it in one place still
* see also Inject, Punq and DRY-Python/deps
* In the case of Flask, the bootstrap.bootstrap() is called in flask_app entry point, the bus can equally be created with test defaults as a @pytest.fixture
* Imagine a new adaptor for notifications, create an ABC and 2 concrete (one prod and one test). Init with some params. Notification now gets a parameter in the bootstrap init to take the concrete.
* MailHog good for email tests


## Epilogue


* This is a whole new arsenal of ways to shoot yourself in the foot
* With existing systems have a clear goal: is it hard to change, performance issues, weird bugs? There are pragmatic discussions to be had for addressing tech debt if you have a reasoned argument of what to change
* Complex changes are often easier to sell if you tie them to a feature (architecture tax)
* Easiest place to start is looking at responsibilities of each component and pulling things out to a service layer
* What are the use cases, each needs an imperative name (upload sample, view sample)
* Each use case should start a transaction, fetch its data, check preconditions (see Ensure pattern), update the domain and persist changes - these should fail as an atomic unit
* At the start of a cleanup your new service handlers will be long and messy but they'll be the first step in ordering the chaos!
* Duplication is fine in use case functions, better to do this in many cases than having use cases call eachother
* Pull data access and orchestration code from models to use cases, same for IO concerns
* Once you've done this you'll have a grasp of what each function does, the logging, data access requirements and error handling and can think about a way for each operation to have a clear start and end 
* dot notation from the ORM is nice but destroys performance and makes it hard to reason about the system, you also don't know which properties will trigger lookups
* Remember that use cases should aim to update only one type of Aggregate
* Similar to Majora the Epilogue talks about a system that had everything inherit from a fixed schema. Mostly fixed by removing FKs and replacing them with identifiers so that each Aggregate could be completely separate
* Tree structures are terrible as unrolling them requires nested loops triggering loads of lazy lookups on the ORM
* In some cases there may be nothing better than circumventing the ORM with a hard coded procedure - this also helps break the models involved in such queries apart from eachother
* see event storming and CRC modelling
* David Seddon "When the nice theory of a book meets the reality of your codebase it can be quite demoralising", "focus on a specific problem and put the relevant ideas to use". "Don't try to boil the ocean and don't be too afraid of making mistakes"
* if you find yourself updating two aggregates at the same time then your boundary is probably wrong. redraw the boundary or perhaps the two operations don't need to be atomic and can be split into two handlers
* Read views can be complex too, consider a view fetcher that reads data and returns basic structs, passed to a view builder that does filtering and maps


## Footguns


* Reliable messaging is hard, Tyler Treat "you cannot have exactly once delivery"
* Small atomic failures require tooling to monitor failures and to replay the transactions when the thing that cause the failure has been fixed
* What happens when handlers are retried, need to design for idempotence - allowing for events to be safely retried (without worrying they'll change state again)
* Event schemas need to be well documented; Greg Young versioning in an event based system


## Appendix 


* see also: clean architecture in python, monolith to microservices
* src folders testing and packaging by hynek schlawack, eventually you'll likely want folders for domain infrastructure services and API
* test have a shared fixture set in conftest.py
* config.py gets config info, use functions in case client code wants to modify os.environ or something
* environ-config is an elegant package for configs
* DRY-python have mappers that might help with translation between Django and Domain objects
* Repo and UoW are hard work in Django and buy you easier unit tests, long term they'll decouple you from Django so it's useful if you're thinking of migration in the long term
* Service Layer is good if there is duplication in views.py and helps separate use cases from the URL endpoints themselves
* Django community somewhat split between fat and thin models
* Good advice is to add a logic.py for business logic that forms and views can use
* Reads from one place helps to avoid ORM code sprinkled everywhere
* Django app hierarchy pretty bad for separating out modules
* Validation covers three preconditions: syntax, semantics and pragmatics
* syntax covers rules about fields (types, orders must have an ID and quantity) etc, these are validated at the edge - message handlers should only receive valid messages, you could put this on the event dataclass itself (see the schema library)
* Tolerant Reader pattern will accept most inputs and emit only validated output - only validate what you require
* message bus can also validate messages and log bad stuff (eg. unknown message types)
* semantics covers the meaning of messages (orders can't have negative quantity)
* create a MessageUnprocessable exception with subclasses for specific issues, use these to test preconditions that should be true (eg. product exists, batch does not already exist)
* subclass the exception to groups for how they should be treated MessageUnprocessable, MessageSkipped
* as a rule of thumb of a rule can be tested by the domain then it should be tested in the domain
* pragmatics is about how to process the message - this is where the real business rules are applied, can we allocate this stock?
* validate syntax on the message class, semantics on the message bus and pragmatics at the domain



