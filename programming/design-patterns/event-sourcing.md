# Event Sourcing made Simple
[Reference](https://kickstarter.engineering/event-sourcing-made-simple-4a2625113224)

- Git: who made the change, what the change was, why the change was done (title), how the change was made (diff), and you can go back in time (checkout).
- Data history: `Datomic` or `PaperTrail.rb`.
- Event sourcing system:
  - Events, calculators, aggregates, reactors.
  - Martin Fowler: "All changes to an application state are stored as a sequence of events."
  - Things that generate events: user action, API calls, callbacks, cron jobs.
- Aggregates: represent the current state of the system (think order cart).
- Calculators: read events and update aggregates accordingly (a list of things ordered, the price of all those things).
- Reactor: Things that react to events: ex, getting shipped means sending an email notification.

## Components of an event sourcing system:

- Events to provide a history.
- Aggregates to represent the current state of the application.
- Calculator to update the state of the application.
- Reactors to trigger side effects as events happen.

## Why Event Sourcing?

- The data your write (events) is decoupled from the data you read (aggregates), so you can design your aggregates for the current needs of the application.
- Different possible aggregates for different read-intensive applications: orders' summary (list views), orders' details (one order), orders' daily reports (BI), etc.
- Good for distributed systems that tend to be asynchronous and have various services or serverless functions.

## Requirements

- Shouldn't be too slow, quick to learn, and easy to rip out/rollback to regular Rails/MVC.
- ES implementation doesn't reach GraphQL.
- Homemade ES framework: Aggregates has an event table associated to it. Events are created and applied to the aggregate in an SQL transaction.

## The Code

- Rails models are the same, but they have a `has_many :events`.
- Base class for `BaseEvent`.
- Subscribing reactors to events: Dispatcher, either async or synchronous.

## Commands

- Validates attributes, validates that the action can be performed, builds and persists the event.

# Rails Event Store
[Reference](http://railseventstore.org/)

- Publish-subscribe bus.
- Decoupling core business logic from external concerns.
- Alternative to AR callbacks and Observers.
- Communication layer.
- Audit log.
- Read-models.
- Implement event-sourcing.

- [Best Event Sourcing DB Strategy](https://stackoverflow.com/questions/28667367/best-event-sourcing-db-strategy)
  - One table for each event?
  - One generic table, save events as a serialized string.
    - Query the event stream by aggregate ID, not event type.
    - Reproducing the events in order is hard if they are in different tables.
    - It would make upgrading events a bit of pain.
    - Searching via event type: add an index on that column.
  - Something to consider: "We still had a set of normalized tables, because we just could not accept that in order to get the latest state of an object we would have to run all the events." So ES is good for versioning/serving as a full audit log, and it should be used just for that, not as a replacement of a set of normalized tables."
    - Treat the views as a convenience that can be trashed/rebuilt at any time.
- [Using an RDBMS as an Event Sourcing Storage](https://stackoverflow.com/questions/7065045/using-an-rdbms-as-event-sourcing-storage?rq=1)
  - Simplest form: Event → `Aggregate_ID`, `Aggregate_Version`, `Event_Payload`.
  - *Event store should not need to know about the specific fields or properties of events. Otherwise, every modification of your model would result in having to migrate your database.*
- [Design Patterns: Why Event Sourcing?](https://www.youtube.com/watch?v=rUDN40rdly8)
  - This is a good talk. Just didn't take down notes yet.
- [Why are event sourcing (CQRS) databases not popular?](https://dba.stackexchange.com/questions/147439/why-are-event-sourcing-cqrs-databases-not-popular)
  - It is arguable that the write-ahead-log that every SQL product uses is an event stream, allowing rebuilding of objects in the event of an instance fault.
  - CQRS can be considered a design pattern as much as it can be considered a product. As such, any mainstream store could implement a CQRS.
  - Accounting, banking, and finance systems: "every transaction is immutable".
  - On querying the history: your probably don't want to query against the event store itself. The most common solution would be to hook up a couple of event handlers that project the events into a reporting or a BI database, then replay the event history against these handlers.
  - CQRS: Separate your read and write models. ES: Use an event stream as the single source of truth in your application.

# Building an Event Storage
[Reference](https://cqrs.wordpress.com/documents/building-event-storage/)

- Basic Event Storage: RDBMS with 2 tables:
  - 1 table as Event log: `AggregateId`, `Data`, `Version`. The event is stored using some sort of serialization.
  - Possible additional information: timestamp, context information, the user who initiated the change, IP address, level or permission.
  - Other table: `AggregateId`, `Type`, `Version`, which stores all of the aggregates. The version number is denormalized. This value is used in the optimistic concurrency check.

- Operations for an event:
  - At the simplest level, there are just two operations. This makes Event Storage simpler than most data storage mechanisms as well as easier to optimize.
  - Operation 1: To get all of the events for an aggregate, then you do a `SELECT * FROM EVENTS WHERE AGGREGATE_ID='' ORDER BY VERSION`.
  - Operation 2: Writing the sets of events to an aggregate root. Either code or stored proc.
    - Write: Check if an aggregate exists, if not, create one and consider the current version to be zero. Then, attempt to do an optimistic concurrency test on the data coming in if the expected version does not match the actual version it will raise a concurrency exception. Then, loop through the events being saved and insert them into the events table, incrementing the version number by one for each event. Transaction to insure that optimistic concurrency amongst other things works in a distributed environment.

- Rolling snapshots: A denormalization of the aggregate at a given point in time. There might be some process of creating the snapshots (a process outside of the app server as a background process).
  - Creating a snapshot: Having the domain load up the current version of the Aggregate, then take a snapshot of it.
  - Use Memento pattern.

- ES as a Queue:
  - Very often, these events are not only saved but also published to a queue.
  - An issue that exists with many systems publishing events is that they require a two-phase commit between whatever storage they are using and the publishing of the events to the queue.

# Event Sourcing: What it is and why it's awesome
[Reference](https://dev.to/barryosull/event-sourcing-what-it-is-and-why-its-awesome)

- The status quo: web development is database driven. Problems with this:
  - We don't think like this, we usually just tell other people what has happened since we last talked.
  - Single data model: We usually design the tables from a write perspective, then figure out how to build our queries on top of those structures. It's usually impossible to build a generic model that is optimized for both reads and writes.
  - We lose business critical information: we get to lose the whole "how many times has a user done this" thing.
- Enforcing business constraints?
  - You replay a subset of the events, and the end result is a projection.
  - Showing data to the user? Instead of building it on the fly, you build it in the background, storing the intermediate results in the database. That way, users can query for data, and they will get it in the exact shape they need, with minimal delay.

## Benefits

- *Ephemeral data structures.* Since all state is derived from events, you are not longer bound to the current state of the app. Bye bye migration scripts. Just create a projection and discard the old one.
- *Easier to communicate with domain objects.*
- *Expressive models.* You model events as first class objects, rather than through implicit state changes.
- *Easier to build reports.* Because you have full history, you can ask any question you like about that data historically.
- *Composing services: use a bus or something.*
- *Databases are optimized for append-only operations.*
- *Easy to change database implementations.*

## Issues

- *Eventual consistency. An ES system is naturally eventually consistent.*
- *Event upgrading.* When the events change shape, you have to write an upgrader that takes in the old event and converts it into the new one.
- *Change in status quo.*

## Comments

- You can start small and progressively optimize.
- Challenge: Set validation, such as storing unique emails. The thing is still inefficient at this type of verification, since you have to replay the entire log every time to validate it.
- Unique index to force sequential operations? This can work, but you have to handle failing use cases.
- Monitoring tools?

# Hacker News: Building a CQRS/ES web application in Elixir using Phoenix
[Reference](https://news.ycombinator.com/item?id=13338848)

- *Versioning events, projection, reporting, maintenance, administration, dealing with failures, debugging, etc etc are all more challenging than with a traditional approach.*
- Migrating immutable events:
  - Multiple versions.
  - Upcasting.
  - Lazy transformation.
  - In-place transformation.
  - Copy and transformation.
- OTOH, if understanding ES just as an ADDITION to classical mutable-state processing - it can be made very useful. Not only ES will serve as a perfect audit, the duplication of information (once in mutable state and once in input events) will allow such things as regression testing, and fixing data problems caused by bugs, within the DB.
- Possible successful examples:
  - Double-entry accounting/bookkeeping/core banking systems.
  - RDMSes, specifically the replayable log structure of transactions.
- The alternative: `Customer` and `CustomerAudit` table. Write it in a transaction. Solved.
- Yes, you can fail with event sourcing. You can fail with Rails. It will take you longer to fail with Rails, though.
- Kafka:
  - "Event log".
  - *The issue isn't about the persistent store of events, but the event payloads changing.*
  - Projection is not the panacea! Inevitable, there will be syncing issues between the event store and the projected database/ES cluster/Mongo instance.
- Why CQRS failed?
  - Not good idea to do real-time querying of the event store.
  - How do you know if the projection worked? What if there was a conflict with another address change form somewhere else?
- Things I found:
  - Do not try to apply ES to all areas of your software, if you apply it at all.
  - CQRS does not require ES, ES does not require CQRS.
  - On-demand projections are fine for a lot of purposes, learn to tell when you're going to need a static projection.
  - A projection is a part of BC. Don't query other BC's at runtime for their data.
  - Use UUID for PK, and originate them with the client whenever possible.
- Fixing an event that should not have gotten in the store: reverse transaction.

- Long comment:
  - If you've only ever built systems under a single paradigm, then you might not even be aware of the differences in reversibility concerns between paradigms.
  - You won't learn it (and wield it safely) if you believe that all things should be as readily-consumed as Meteor over Mongo, or Rails, or pick-your-forms-over-data tool.
  - Event sourcing makes things simple - far simpler than ORM, for example. But it only does so when approaching it from its own predispositions, rather than the predispositions (especially unrecognized) of some foreign and impertinent approach.
  - I can do an event-sourced system today as fast as I could have built the same system in a rapid-prototyping tool like Rails in years past. However, unlike Rails (for example), my productivity, and my team's productivity remains stable, and changes in scale and complexity of the business and its organization don't induce the panic that it used to.
  - *By "problems that shouldn't exist in the first place", I mean complexity and lack of clarity that is the result of presuming to reproduce a relational database schema in an object model. This is the single biggest problem in application software development today, and unfortunately it's also a default mode of development.*
  - *Until it clicks that ORM is as unnatural an abstraction now as server pages was in the 2000s, event sourcing probably won't matter.* And working with event sourcing might create all kinds of problems for not having recognized how to partition a domain. You may just end up with a distributed monolith rather than a service architecture. And at that point, you might just end up blaming event sourcing as a pattern rather than the unconscious importing of anti-patterns from "traditional" development.
