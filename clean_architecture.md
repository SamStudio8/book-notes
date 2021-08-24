# Clean Architecture


* “If you think good architecture is expensive you should try bad architecture”
* We don’t necessarily know the right architecture for a problem, that is part of the journey
* Software architecture is only partially limited by physical constraints, almost more architecture than physical building architecture
   * Makes me think of Escher paintings - we can bend ourselves around shapes that don’t exist physically
* Languages have changed but programming concepts have not, thus software architecture is not only remarkably similar across varied projects, but timeless too
* Getting software to work is easy, getting it right is hard
* The goal of software architecture is to minimize the human resources required to build and maintain a software system
* Never leave “cleaning up” for later -- it never happens, developers get stuck in auto pilot and don’t “think”. Just as the Hare was confident in their speed, developers are confident in their productivity. The creeping mess they have left behind is the Tortoise.
* The only way to go fast, is to go well.
* Software has two values -- behaviour and structure
   * Software is supposed to be ...soft, the difficulty in making a change should be proportional to the scope of the change, not the shape of the change
   * Stakeholder’s think they are giving a stream of changes, developers feel like they are getting jigsaw pieces that need to be jammed into an ever complicating puzzle
   * Good architecture is “shape agnostic”
* It is the responsibility of the software developers to communicate the urgency of architecture over the urgency of features -- effective teams tackle this “struggle” head on


### Programming Paradigms


* Structured programming imposes discipline on direct transfer of control
   * Discovered by Dijkstra on his quest to formalise programming with proofs
   * Allows programs to be recursively decomposed into provable units
   * Killed goto -- uncontrolled transfer of control
   * But the proofs never caught on
   * Dijkstra said “testing shows the presence, not absence of bugs”
   * Software is science, not math. We can only prove incorrectness.
   * Strive to build easily falsifiable components and build them together
* Object orientated programing imposes discipline on indirect transfer of control
   * Encapsulation, inheritance and polymorphism
      * Yet all these things existed in C, but OO made polymorphism in particular, safer -- polymorphism is effectively a plugin architecture
      * Dependency inversion -- included components no longer have to follow the flow of control, allowing for things to be abstracted independently -- independent deployability
* Functional programming imposes discipline on assignment
   * Strongly based on Alonzo Church’s lambda calculus
   * You cannot have race conditions or concurrent updates if there are no variables to be updated
   * You cannot have a deadlock without mutable locks
   * Segregation of mutability -- allows some components to be immutable 
      * Often achieved with transactional memory, which treats variables like a database -- transact or retry
   * Often there are ways to mutate variables
      * Clojure has “atom” variables that can be updated with strict discipline
   * Architects would be wise to push as much as possible into immutable components 
      * Event sourcing -- don’t store results, compute them from the start of time (or checkpoints), CRUD is now CR


* Programs are composed of sequence, selection and iteration
* The way software is written has not really changed since Turing 1946


### Design Principles


* Mid-level principles -- just above the code
* Resuable, easy to understand, tolerate change


#### SOLID 
        
* Single Responsibility principle
   * Modules have one reason to change -- a module should be responsible to one and only one actor
   * Not to be confused with a module doing just one thing (a function should do one thing)
   * Consider a class for Employee with calculatePay, reportHours, save
      * These do different things for different actors; accounting, HR and database
      * These things should be different classes, and not know about eachother
      * If you want to keep the old interface, use a Facade to perform delegation
* Open-closed principle
   * “Objects or entities should be open for extension but closed for modification”, Meyer
   * Design such that changes should add new code, not change existing code
   * Imagine a system that generates financial data with negative numbers in red, what if we want to put them in parentheses instead? A good system should regular 0 code changed to allow this -- it’s just a new formatter
      * There are two separate and distinct responsibilities - calculation and presentation
      * If Component A should be protected from changes in B, then B depends on A (not the other way around)
      * There will be a hierarchy of protection -- normally business rules are at the top
      * We may also want to project medium level things from high level things by hiding their internals through an interface
   * Goal is to break the system into components that are easy to extend, arranging those components into a dependency hierarchy that protects high-level from low-level changes
* Liskov substitution principle
   * Build parts to a contract that allows them to be substituted for eachother
   * [1]“This means that every subclass or derived class should be substitutable for their base or parent class”
   * Consider the “infamous” square/rectangle problem -- if W and H can be set independently for Rectangle then Square cannot be substitutable 
   * Such things may call for configurable structures to support non-substitutable interfaces as substitutable
* Interface segregation principle
   * Avoid depending on things you do not use
   * [1]Consider a volume() def on a 3D shape, don’t include this in an interface that covers 2D shapes.
   * Imagine three users with a function in a class (seems like the SRP…)
   * Some consider ISP to be a language issue, not an Architecture one, as often the issue rears its head in statically typed languages
      * Authors argue it is harmful to rely on interfaces that offer more than what is needed -- consider this when deciding on frameworks and other 3rd party code -- relying on something that has baggage you don’t need can cause errors you don’t expect
* Dependency inversion principle 
   * Code implementing high level policy should not depend on low level code
   * Details depend on policies
   * “Entities must depend on abstractions, not on concretions.”
   * [1]A function that pulls from a database should not be dependent on the database itself
   * Simply you should not call “import” “use” or the like on a module that defines concrete behaviour -- ie. modules where functions are called, rather than implemented
   * DIP worries more about volatile concrete elements being included, we can tolerate some concrete dependency if they are very unlikely to change
      * Do not refer or derive from volatile concrete classes
      * Do not override concrete functions -- abstract the function and create multiple implementations
      * Volatile concrete objects should come from the Factory pattern, ideally


### Component Cohesion


#### Three Principles


* REP: Reuse Release Equivalence Principle
* CCP: Common Closure Principle
* CRP: Common Reuse Principle


These principles fight against each other. REP and CCP are inclusive, CRP is exclusive. Ignoring the CCP will have ungrouped components with wide reaching changes, ignoring the CRP will generate too many releases as software won't be appropriately split for release. Without REP to group for end users, it will be hard to reuse. 


A good architect finds a solution to the tension. In early projects the reuse will be sacrificed in favour of development speed. Cohesion is complex and not as dry cut as a module performing one and only one function. Generally projects will evolve and move from developability to reusability and takes balance.


##### REP


Granule of reuse is the granule of release. Seems obvious in retrospect. People will not reuse components that have not gone through a release cycle and have a version number. Release should have the appropriate notification and documentation for users to decide whether to use the new version and what changes are needed.


Classes and modules are a cohesive group and should be releasable together. This is a weak principle without obvious guidance. If you violate the REP principle, your users will know -- the code won't make sense.


##### CCP


Keep components that change at the same time for the same reasons together. This is basically the Single Responsibility Principle, for components. Just as a class should not contain multiple reasons to change, a component should not have multiple reasons to change.


For most applications, maintainability is more important than reusability. If classes always change together (because they are so tightly bound physically or conceptually) then they belong in the same component. Also related to the Open Closed Principle. When change does come along, components should be closed to the same types of changes.


##### CMP


Don't force users of a component to depend on things that don't need. When we depend on a component, we should depend on every class inside the component. Classes in a component should be inseparable. Otherwise users will deploy more components than necessary. Don't depend on things you don't need. See also the Interface Segregation Principle.


### Competent Coupling


Again, tension between develop ability and logical design. Technical, political and volatile forces impinge on architecture.


#### Acyclic dependencies principle


* Allow no cycles in the component dependency graph.
* The "morning after syndrome"; somebody stayed up later than you and broke something you rely on.
* In middle to large projects, components may have their own version numbers and be released to the wider team after their own integration and testing
* no team should be at the mercy of other teams
* To break the cycle you can use the DIP dependency inversion principle. Generate an interface that speaks to the thing you are currently depending on.
* as the project grows you will need to shift things around to cut out dependencies
* component dependency is less about the system function and more about building and maintaining
* protect stable high value components from volatile ones


#### Stable dependencies principle
Depend in the direction of stability. 


* Some components are designed to be volatile.
* However someone depending on your volatile module can make it harder to change.
* One sure way to make a module harder to change is a bunch of things depending on it
* Incoming dependencies (things that rely on your module) are responsibilities
* The most stable modules are responsible (making it harder to change) and independent (no dependencies of its own to force change)
* often the case to break a dependency into an interface that is depended on, and implemented by the dependency (DIP) as now the dependency depends on the interface not the other way around
* this is why abstract classes exist - very stable (effectively you are depending on a solid empty shell)
* this is less of a problem in dynamically typed languages that don't rely you to flag types and classes you depend on


#### Stable abstractions principle


A component should be as abstract as it is stable


* Policies should be encapsulated in stable components, enter the OCP (open closed principle)
* flexible without modification - the very definition of abstract classes!
* Stable components should be abstract so they are still allowed to change. Unstable components should be concrete so the code can be changed.
* Stable components should consist of interfaces. Interfaces are still flexible.
* SAP and SDP are basically DIP at the component level - dependencies will run in the direction of abstraction, that is the component way to invert dependence.
* Components are not black and white like classes. They can be partially abstract if needed.
* Consider a graph with Abstraction and Instability on each axis. Abstract and maximum stability, unstable and concrete are the two ideals.
* On the opposite corners are the Zone of Pain and Zone of Uselessness
* Zone of pain contains highly stable concrete classes. It's concrete code is too rigid. This isn't a catch all rule, imagine a database schema. Volatile, concrete and depended on by everything. This is why database updates tend to be so painful (although I think the database is just a inverse dependent on the models…). Also library classes that are just super nonvolatile like the String class 
* Zone of Uselessness is detritus. Abstract components that are not used.
* worth examining aberrant components


## Architecture
### What is architecture


* never fall for the lie that software architects pull back from programming!
* software architects are the best programmers and also steer a team to maximise productivity (sn: in peopleware it was said a manager just enables a team to do their job, I see a similar role here)
* they program less than the team but most program to understand the problems their architecture makes for the team
* the strategy is to leave as many options open as possible for as long as possible
* architecture has little bearing on whether the system works, of course we want it to work. architecture is about facilitating development deployment operation and maintenance.
* architecture is passive, almost cosmetic. architecture supports the life cycle of the system. good architecture makes the system easy to understand, which makes it easy to develop, deploy and maintain.
* real goal minimise lifetime cost and maximize development productivity
* easy to create a project with bad architecture in a small team as there is a cost to thinking about it. a system with N teams will likely gravitate to a system with N components if nobody does anything to intervene.
* deployment should be possible in one action in an ideal world
* bad operational architecture is usually solved with more hardware. hardware is cheap and people are expensive. architecture that impedes operation is less bad as architecture that impedes development deployment and maintenance. good architecture will make the operational requirements clearer to developers.
* maintenance is the most costly and longest bit of software.
* spelunking is the act of diving into a codebase to find defects or where new features go and strategies to implement them. good architecture will light the way for people to find things.
* the way you keep software soft is to leave as many options open as possible
* software essentially boils down to policies and details. policy embodies business rules and procedures. policy is the most essential element of the system. the goal is to make the details that interact with the policy irrelevant.
* this allows you to delay and defer decisions until you know what the right answer is
* a good architect maximises the number of decisions not made
* Conway's law - any organisation that designs a system will produce a design whose structure is a copy of the organisation's communication structure
* in reality the goals are indistinct and inconstant.


### Independence


think about what changes, how often and for what reason
* horizontal layers, decouple layers: UI, business rules, technical details
* thin vertical stripes across layers, decouple use cases: use cases are a natural way to divide the system. use cases will touch UI, rules and technology
* decouple mode of operation: can the UI and database run on different servers? can queries be run on different places with different hardware requirements? can things be deployed as a micro service?
* if layers and use cases are decoupled then that supports teams working on different components of the system. doesn't just limit you to teams that focus on layers or particular components. 
* not all duplication is true duplication, imagine a similar class wait a few years and they might have evolved completely different paths (sn: almost like gene copies!) - don't unify accident duplication, it can be hard to untangle later 
* don't lazily unify things either. consider a data query that passes to a view. don't pass the entire record. take the effort to make a view of the data. eg… majora… Django lets you so easily read things from the model object!
* there are ways to decouple modes. at the source level we can use dependency control (eg conda, pip) such that changes don't force recompilation of the whole system - commonly seen in monoliths. at the deployment level the dependency can be controlled dynamically with jar files, DLLs and libraries. processes may communicate through sockets and shared memory. at the service level, the dependency is just the data structure itself. communication between components is done through exchange of messages. there are no deployment dependencies as the service model is isolated.
* which is best, source, deployment or service? hard to know at the start and it's likely to change too. a single server app one day may need to scale to multiple servers.
* service level is the default these days but is the most expensive. making boundaries in the system that can only be crossed by data is a waste of effort, memory and cycles. 
* book recommends to keep components in the same address space for as long as possible but keep the option for service design open
* a good architecture will allow a system to be born as a monolith and grow to deployable units, all the way to independent services if needed it should be able to go all the way back to a monolith too
* yes, this is tricky. but if the decoupling mode is likely to change with time, a good architect will forsee and appropriately facilitate this


### Boundaries


* Software architecture is the art of drawing boundaries
* these boundaries separate components and prevent things on one side knowing about what's on the other side 
* decisions about frameworks, databases, web servers, libraries should be ancillary and deferrable. these decisions are premature.
* note that topology is not architecture, just part of it. deciding topology should be deferred too.
* sometimes all you need to defer a decision is an interface. need to access data? write an interface. subclass it to work in RAM, then through files, then eventually, if needed, a database.
* you draw lines between things that matter and things that don't. the GUI doesn't matter to your business logic so should be separated.
* IO is irrelevant. people accidentally think in terms of the GUI and revolve their thinking around that. 
* imagine the core application as business rules, providing interfaces aware of those business rules to anyone who wants to depend on them. this means a database or a GUI is just a plugin.
* boundaries are often drawn around axis of change. things on one side may change more than the other, this is just the SRP single responsibility principle again


### Boundary anatomy


* low level processes should be plugins to high level processes
* communication across local processes is more expensive that source code monoliths and same process dynamic deployment components.
* communications across services are much slower and more costly
* a service is just a facade for a monolith or set of local processes
* a system will have many boundaries, some will be chatty and pass a lot of data. some will have more latency.


### 19 Policy and Level


A computer system is a detailed description of the policy by which inputs are transformed into outputs.


* The art of architecture is separation of policies from one another and grouping their implementation in a way that allows policies that change for the same reason at the same rate to be collected together.
* source code is decoupled from data and coupled to level
* A strict definition of Level is "distance from inputs to outputs". Policies that manage inputs and outputs are the lowest level.
* high level things tend to change less often and for more important reason than low level things.
* this is all just a summation of the SRP single responsibility, OCP open closed, CCP Common Closure, DIP dependency inversion, SAP stable abstractions


### business rules


Business rules make or save money.


* * It usually doesn't matter if these rules are implemented on a computer or not, they are facts.
* * Critical business rules and data would exist whether or not there was a way to automate them.
*  The rules and data are often inextricably linked and are named Entities. 
* Entities may turn out to be a class but their definition is unsullied by technical chat. Entities are Pure Business.
* use cases are application specific business rules that constrain your system.
* "use cases control the dance of entities"
* notice the use case does not describe the UI in any means other than what it must do, or show to achieve its goal
* entities are high level and don't know or depend on the use cases that use them. use cases are closer to the inputs and outputs.


### screaming architecture


Ivar Jacobson said in Object Oriented software engineering says software architecture builds structures to support use cases of a system.


Architecture should not be about frameworks. Frameworks are tools to be used, not architectures to be conformed to. If your architecture is based off a framework, it's not based on your use cases. Frameworks help, but what is the cost? How can you protect yourself from bad parts or changes in the framework? What is the strategy to prevent the framework taking over the architecture?


If your architecture is truly about the use cases it should be possible to unit test the whole thing without it (sn: this seems quite extreme).


the web is merely another IO device! it's not an architecture.


### The Clean Architecture


* Built on Hexagonal Architecture (Ports and Adaptors), DCI, BCE
* Regardless of how you do it, its about separation of concerns
* Systems should be
   * Independent of frameworks, use them as tools not constraints
   * Testable, without the UI, database etc,
   * Independent of the UI
   * Independent of the database
   * Independent of any external agency
* Clean Architecture: Entities, Use cases, Controllers, External interfaces
* Source code dependencies should point inward, toward high level policies
   * Nothing in an inner circle should know the name of an outer circle
   * Operational changes should never impact the central entities layer, but are likely to change use cases
   * MVC GUI lives wholly in the controllers/interface adaptors layer; converting between formats suitable for the database and views
   * Finally the outermost layer is where frameworks should live
   * Entities or database rows should not cross boundaries, only data structures
   * The Golden Rule: talk inwards with simple structures, talk outwards through interfaces [2]
* Use Humble Objects - things that are hard to test are split in two. Consider a GUI, broken into a Presenter and View. The View is hard to test but the Presenter that sets all the strings and data structure for the View can be easily tested.
   * Database gateways; interface and interactor
   * Humble objects are used near and on boundaries
* Boundaries are expensive. Partial boundaries are cheap but can result in backchanneling. No boundaries are very hard to pull apart later.


#### Main component


* The dirtiest of dirty components. Nothing other than the OS relies on it. This is where all the dependencies are injected.
* Main creates all the dirty stuff then hands control of it over to the high level stuff


#### Services


Architecture is defined by boundaries that separate high level policy from low level detail. Services that separate application behaviours are just expensive function calls.
* This doesn't mean that services cant be significant to the architecture, its just being a service in of itself does not give it that status. A monolith is composed of boundaries separated by function calls -- but many of those functions aren’t significant.
* Services are talked about as if they are not coupled, but their data structures bind them together. A new field must go into all services that want to use it, and all services must agree on the field definitions. This makes them no different from interfaces.
* Think of a service as a set of abstract classes, then features implement those classes - new features are just new classes (the open closed principle)
* Boundaries can be cross cutting, dividing services themselves into components


#### Tests
* Tests are the perfect component. Dependent on everything and depended on by nothing.
* The fragile test problem - tests so coupled to the codebase that changing the code requires changing hundreds of tests
* Fragile tests make projects harder to change
* Don't depend on volatile things (and tests are no exception)
* Use a testing API that can bypass everything -- security, databases and so on, that can force the system into a testing state, and detached from the GUI
* Avoid structural coupling between the test and system


#### Embedded
* Use the concept of the HAL - the Hardware Abstraction Layer
* HAL is basically an API that the software uses to speak to firm or hardware
* A HAL might provide LED_TurnOn(Number) rather than writing to GPIO bits, maybe even higher than this - what does the LED mean? Perhaps the HAL abstracts this LED_TurnOn to Indicate_Battery(), Indicate_Activity()
* The HAL expresses “services” needed by the layers above the firm and hardware
* GPIO assignments should be hidden from the software
   * Don’t reveal the HAL to the users
* The HAL allows for off-target testing, saving you from manual tests and the target-hardware bottleneck
* Use an OSAL (OS abstraction layer) to avoid relying on your RTOS
* Dont clutter header files


### Details


good architects don't allow low level mechanisms to pollute the system architecture. the database is a piece of software, it's not the data model. many frameworks allow rows of data to be passed around, this is an architecture error.


the web is a detail. the web is an io device. given how tied and complex the web frameworks are, this abstraction will not be easy.


frameworks solve someone else's problem, not yours. they are often not clean, they require you to inherit their code into your entities, breaking the dependency rule! often the framework works early on and then imposes limitations that you outgrow later. derive proxy classes to their base classes. try to keep it at arm's length before you fully commit.


#### case study


* use cases, identify actors and activities. abstract use cases are a general policy
* single responsibility principle, the system will only have adds or change will serve only one person
* with the actors and use cases we can create a preliminary component architecture (controllers, interactors, presenters, views, gateways)
* input to controllers, processed into a result by interactors, presenters format the results and the view display the presentation


#### missing chapter by Simon brown


* you can start off packaging by layer, but move to packaging by feature
* packaging by feature is vertical slicing rather than horizontal
* we can do better - ports and adaptors. inside has domain knowledge and outside has the infrastructure
* but actually package by component is what this book suggests
a component is a grouping of related functionality behind a clean interface which resides inside an execution environment
see the C4 software model
containers contain components which contain classes


enforce architecture principles with the compiler, not self discipline


your best design intentions can be destroyed in a flash if you don't consider the intricacies of implementation strategy


* think about how to map your design to data structures
* think about decoupling
* leave options open, be pragmatic
* consider the team size and skill
* consider the complexity
* consider time and budget
* use your compiler to enforce rules
* watch out for coupling


#### afterword


it's like pool. it's not just about sinking the ball, you have to always be lining up the next shot. writing working code that doesn't block future options takes years to master


##### See also
* [1] https://www.digitalocean.com/community/conceptual_articles/s-o-l-i-d-the-first-five-principles-of-object-oriented-design
* [2] https://www.pycabook.com/#_communication_between_layers

