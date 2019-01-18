# EVSTORE (Foxx Microservice) #
A git-inspired event store for ArangoDB.

### DISCLAIMER ###
* This project is under active development.
* Expect heavy feature churn and unstable builds in the initial days.
* **DO NOT** use in production systems until a stable build is announced!


### Introduction ###
_evstore_ is an event-based datastore with version-control - like features.

It is a [Foxx Microservice](https://www.arangodb.com/why-arangodb/foxx/) for [ArangoDB](https://www.arangodb.com/) that features _git-like_ semantics in its interface, and is backed by a transactional event-sourcing tracker.
 
### Quick Technical Overview ###
This quick overview is intended to introduce the user to some high level concepts that would let them get started with the service. A more detailed technical document would soon be made available in the project's wiki.

_evstore_ exposes multiple write methods for individual/multiple nodes (documents/edges). Supported write method contracts (current and planned) are intended to closely follow the core REST API that ArangoDB already exports. These include:
1. Create (POST) (Implemented)
2. Replace (PUT) (Implemented)
3. Update (PATCH) (Planned)
4. Delete (DELETE) (Implemented)

Node read methods would be no different from what the core REST API already provides, and so they are left out the microservice.

When a write method is invoked on a node, the following things happen behind the scenes:
1. A transaction is opened with read and write (non-exclusive) locks on appropriate collections.
2. The provided node is written.
3. An event object corresponding to the write is created, that records the current time, event type (create/update/delete) and some meta information about the node. This event is appended to a service-managed **document** collection.
4. A command object is created using [JSON Patch RFC6902](https://tools.ietf.org/html/rfc6902) to compute a reversible diff from the last known state (`{}` by default) to the current state of the node. This command is appended to a service-managed **edge** collection, linking the current event to the last one (an `origin` event by default).
5. Optionally, if a specified number of events have been recorded for the node, a snapshot object is created which records the entire current state of the node. This snapshot object is linked to the current event and persisted to a service-managed collection. The number of events that must occur between two consecutive snapshots is configurable at a default level as well as a collection-specific level in the service configuration.
6. The transaction is committed.

If something goes wrong at any step in the above process, the transaction is rolled back.

This way, every time something happens to a node (a create/update/delete event), a permanent, immutable record of that event is stored forever in the database. These records can be queried in different ways to either:
* view a node's mutation history, or
* rewind a node to any point in time in its mutation history, or
* even bring a deleted node back to life!

Snapshots, when available, are used on a best-effort basis to minimize the number of diff calculations required to perform a rewind/fast-forward.

**The process described above makes the implicit assumption that all mutation methods for a node were invoked through _evstore_'s API, allowing it to record all changes, and no direct manipulation happened**. But what if somehow, a node underwent a few direct mutations via other means (AQL/Core REST API/Client)?

Well, all is not lost in this case, since _evstore_, like Git, supports a **commit** operation that lets you explicitly add an event record post hoc. Obviously, this would create only a single diff from the last known state to the current state, and any intermediate mutations would collapse into that diff. Unfortunately, there is no way around this.

_evstore_ manages all its bookkeeping in a set of service-managed collections, and does not write anything to user-defined collections, other than the specific node records that the user explicitly asked to save. This means that the user gets a clean view of their own collections/data, not polluted by any service metadata (just like Git's working tree). They can query this data as though the service is not even there!

### Salient API Features ###
Detailed API docs are available in the [project's wiki](https://github.com/adityamukho/evstore/wiki/API). A lightweight documentation is also embedded in the built-in Swagger console. 

#### Document ####
* Create - Create single/multiple nodes (documents/edges)
* Replace - Replace entire single/multiple nodes with new content
* Delete - Delete single/multiple nodes
* **(Planned)** Update - Add/Update specific fields in single/multiple nodes

#### Operations ####
* **(Planned)** Explicit Commits - Commit a node's changes separately, after it has been written to DB via other means (AQL/Core REST API/Client)
* Log - Fetch a filtered and optionally grouped log of events for a given path pattern (path determines scope of nodes to pick)
* **(Planned)** Diff - Fetch a list of forward or reverse commands (diffs) between commit endpoints for specified nodes (might use `log` behind the scenes)
* **(Planned)** Patch - Apply a set of diffs to specified nodes to rewind/fast-forward them in time (will use `diff` behind the scenes)

### Setting Up ###
1. Clone this repository.
2. Follow the instructions in the [Foxx Deployment Manual](https://docs.arangodb.com/3.4/Manual/Foxx/Deployment.html). The web interface is the easiest, while the `foxx-cli` is more suitable for power users.
3. Try out the API endpoints through the Swagger console.

### Testing ###
**IMPORTANT:** Running tests will create some test collections apart from the usual service collections. This has a few caveats. **Carefully read the following points before running this service's test suites:**
1. Although test collections are namespaced using a prefix (the service mount point) in order to minimize chances of collision with user-defined collections, there is a small chance that it could still happen, especially when the same prefix is also used for user-defined collections.
2. Both service and test collections are populated with test data.
3. **Both service and test collections are truncated at the start of every test run!**

To avoid getting into trouble while testing, it is best to deploy this service to a blank database that isn't used for anything else, and then run the test suites there.

Run tests via the web interface or `foxx-cli`. Note that the tests take quite some time to finish, and only print their results in a batch at the end. It may look like your database has gone out for lunch, but it is actually busy crunching numbers. Use `top` or equivalent to monitor the process if you're unsure.

### Docs ###
* Some documentation is already available through the Swagger interface.
* Detailed API docs are available [here](https://github.com/adityamukho/evstore/wiki/API).
* Detailed technical documentation is actively being worked on, and will be available in the project wiki very soon.

### Limitations ###
1. Although the test cases are quite extensive and have good coverage, this service has only been tested on single-instance DB deployments, and **not on clusters**.
2. Since ArangoDB 3.4 does not support ACID transactions in [cluster mode](https://docs.arangodb.com/3.4/Manual/Transactions/Limitations.html#in-clusters), transactional ACIDity is not guaranteed for such deployments.

### Contribution Guidelines ###
A formal contribution guideline document will be prepared eventually. In the meantime,

* stick to the exisiting coding and formatting styles,
* maintain the current file and folder structure (it is quite simple and self-explanatory right now. Drop me a line if something isn't clear),
* write test cases for every new piece of functionality and ensure good  code coverage. Automated builds and coverage reports will be set up soon.

### Get in Touch ###
* Raise an issue or PR on this repo, or
* Mail me (email link in Github profile), or
* DM me on Slack - `adityamukho@arangodb-community.slack.com`.