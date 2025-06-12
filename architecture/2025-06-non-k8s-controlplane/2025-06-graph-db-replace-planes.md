## Proposal: Integrating a Graph Database for Radius Application Graph

**Version:** 1.2
**Date:** June 11, 2025
**Author:** Sylvain Niles

---

### Overview

Project Radius currently defines and manages application graphs, representing resources and their relationships within cloud-native applications. Currently Radius relies on Kubernetes to install etcd, the default datastore, where these graph structures are stored as key value pairs. As we are adding support for nested connections the imperative Go code is becoming complex and brittle because it needs to implement basic graph traversal logic not present in etcd. Additionally we are already hitting performance limits in test environments (under a hundred resources were reported to slow things to a crawl) which inspired the work @superbeeny has done to swap the key value operations out to use postgres. Radius users could not use Drasi today to act on changes to their environments as there's no current support for key value stores and if that was added the client side filtering requirements would be a challenge in Drasi, requiring extensive middleware to parse the custom Radius data structures. 

This proposal outlines a plan to modify Project Radius to utilize a **graph database** as the primary store for its **application graph data specifically**, accessed via a new **Graph Access Layer (GAL)**. This change aims to decouple the core graph logic and operations from etcd, enabling more powerful graph queries, improving performance for complex relationship traversals, and offering a more specialized and efficient graph persistence layer. **This proposal is scoped specifically to application graph operations - individual resource metadata and configuration will continue to use the existing `database.Client` interface with the current storage backends (etcd via Kubernetes APIServer, or Postgres for production deployments).** One of the benefits to recipe authors is this will allow them to define nested connections of types to both return complex relationships and properties to a recipe as well as see these relationships in the graph/dashboard.

The **Graph Access Layer (GAL)** will be designed with a pluggable backend architecture that supports any Cypher-compliant graph database, enabling flexible deployment scenarios and allowing users to choose the graph database that best fits their operational requirements and constraints. This approach provides the freedom to leverage different graph database technologies while maintaining a consistent abstraction layer for Radius components.

**Storage Strategy Rationale:** This proposal focuses specifically on application graph data because: (1) Graph databases excel at relationship queries but may be overkill for simple key-value resource storage; (2) The existing `database.Client` interface and Postgres backend already provide excellent performance for individual resource operations; (3) This allows incremental adoption with lower risk; (4) Different data types (graph relationships vs. resource metadata) have different access patterns and requirements; (5) Production deployments can continue using battle-tested Postgres for resource storage while gaining graph capabilities for application relationship queries.

Delivering this will allow us to shift to a far better user experience where connections become a rich re-usable concept that shares data and exposes deep relationships currently obfuscated by monolithic types with embedded objects (ex: database has a credentials object with username and password properties).

### Terms and Definitions

* **Radius:** An open-source, cloud-native application platform that helps developers build, deploy, and manage applications across various environments.
* **Application Graph:** A representation of an application's components (resources, services, environments, etc.) as nodes and their interconnections as edges, including metadata on both.
* **Kubernetes (K8s):** An open-source system for automating deployment, scaling, and management of containerized applications.
* **Graph Access Layer (GAL):** The internal abstraction layer that mediates all graph operations between Radius components and the underlying graph database.
* **Kùzu:** An embeddable, transactional, high-performance graph database management system (GDBMS) supporting the Cypher query language and property graphs. Provided for dev/test/POC.
* **Postgres with Apache AGE:** A production-ready, scalable graph database solution supporting Cypher queries, built on PostgreSQL.
* **Node:** An entity in a graph (e.g., a Radius resource, an environment).
* **Edge:** A relationship between two nodes in a graph (e.g., "connectsTo", "runsIn"). All edges in Radius will be of type "CONNECTION".
* **Property:** Key-value pairs associated with nodes or edges, storing metadata.
* **Cypher:** A declarative graph query language.
* **RP (Resource Provider):** A component in Radius responsible for managing a specific type of resource.

### Objectives

1.  **Decouple Graph Storage:** Abstract the application graph storage from etcd, allowing Radius to use a dedicated graph database via the Graph Access Layer while maintaining existing resource storage through the current `database.Client` interface.
2.  **Enhance Query Capabilities:** Leverage Cypher query language for more complex and efficient graph traversals and relationship analysis than what is easily achievable with etcd retrieval and client side filtering.
3.  **Improve Performance:** Improve the performance of graph read and write operations, especially for large or complex application graphs, while maintaining current performance for individual resource operations.
4.  **Maintain Existing Functionality:** Ensure that all existing Radius features that rely on the application graph continue to function correctly with the new backend, and that all resource management operations continue unchanged.

### Issue Reference:

* `radius-project/radius#<NewIssueID>` (To be created)

### Goals

* Implement a Graph Access Layer (GAL) that abstracts application graph operations with pluggable backend support, working alongside the existing `database.Client` interface for resource storage.
* Integrate Kùzu as an embedded graph database for development, testing, and proof-of-concept environments.
* Develop Postgres with Apache AGE plugin support for production environments.
* Define a clear schema for the Radius application graph within both backends.
* Migrate existing application graph data representation (currently derived from resource relationships in etcd/postgres) to the dedicated graph database data model.
* Update Radius components to use the new GAL for all application graph operations while continuing to use `database.Client` for individual resource storage.
* Provide migration tools for moving application graph data to the graph database.
* Provide mechanisms for backup and restore of the graph database as part of Radius install/upgrade/rollback operations.
* Develop a comprehensive test suite covering graph operations with both backends.
* Add configuration options to select between graph database providers.
* Ensure the GAL can reconstruct application graphs from existing resource data during migration.


### Non-goals

* Providing a distributed Kùzu cluster as part of this initial integration (Kùzu is primarily embedded; clustering would be a separate, future consideration if needed).
* Exposing direct Cypher query capabilities to end-users of Radius (interaction should remain through Radius APIs and abstractions).
* Supporting zero-downtime migration from etcd to graph database (migration will require a maintenance window).
* Replacing the existing `database.Client` interface or resource storage mechanisms - individual resource metadata will continue to use the current storage backends (etcd via Kubernetes APIServer or Postgres).
* Migrating non-graph data (individual resource configurations, secrets, etc.) to the graph database - this proposal is specifically scoped to application graph relationships and traversal operations.

### User Scenarios (optional)

* **Scenario 1 (Developer):** A Radius developer needs to address performance issues for application graphs containing many resources. Retrieving all resources connected to a specific environment, which is a very expensive operation. Using Cypher through the GAL would be more expressive and orders of magnitude faster than multiple etcd calls and client-side filtering.
* **Scenario 2 (Developer):** A developer is troubleshooting a deployment that isn't working correctly. By examining the application graph through the new deeply nested app graph, they can see that their gateway resource is connected to two secret resources (crt and key), and upon inspection, they discover this certificate is for the wrong domain, explaining why their HTTPS connections are failing.

### User Experience (if applicable)

* **End-users (Application Developers using Radius CLI/APIs):** The change should be largely transparent. Existing commands and APIs for managing applications and resources should continue to work. Performance improvements might be noticeable.
* **Radius Developers/Contributors:** Will need to learn how to interact with the new GAL and potentially understand the graph data model and Cypher for advanced debugging or development. The graph implementation is fairly simple, with a small number of queries that should remain static, only new queries to support new functionality would require ramp-up on Cypher. Non-cloud test suites may speed up significantly.
* **Operators:** Will need to be aware of the new graph database component for backup, monitoring, and troubleshooting purposes. The operational burden of managing etcd for graph data would be shifted.

### Sample Input/Output:

**Example 1: Application Graph Query (replicating `rad app graph todo`)**

* **Sample Input:**
    ```
    // Show the full graph for the "todo" application with connections
    MATCH (app:Application {name: 'todo'})-[rel:CONNECTION]->(res:Resource)
    OPTIONAL MATCH (res)-[conn:CONNECTION]->(connected:Resource)
    RETURN app.name AS application, res.name AS resource, res.type AS resourceType, 
           collect(connected.name) AS connections
    ```

* **Sample Output:**
    ```json
    [
      {
        "application": "todo",
        "resource": "frontend",
        "resourceType": "Applications.Core/container",
        "connections": ["backend"]
      },
      {
        "application": "todo",
        "resource": "backend",
        "resourceType": "Applications.Core/container",
        "connections": ["todo-db", "redis-cache"]
      },
      {
        "application": "todo",
        "resource": "todo-db",
        "resourceType": "Applications.Core/postgres"
      },
      {
        "application": "todo",
        "resource": "redis-cache",
        "resourceType": "Applications.Core/redis"
      }
    ]
    ```

**Example 2: Troubleshooting Gateway Certificates (Realistic User Interaction)**

* **User Command:**
    ```bash
    rad resource inspect gateway api-gateway --show-dependencies --type secret
    ```

* **Internal GAL Query (not exposed to user):**
    ```
    // GAL finds secrets connected to the gateway
    MATCH (gateway:Resource {name: 'api-gateway', type: 'Applications.Core/gateway'})
    -[conn:CONNECTION]->(secret:Resource {type: 'Applications.Core/secret'})
    RETURN secret.id AS secretId, secret.name AS secretName
    ```

* **GAL Response to CoreRP:**
    ```json
    [
      {"secretId": "/subscriptions/.../secrets/tls-cert", "secretName": "tls-cert"},
      {"secretId": "/subscriptions/.../secrets/tls-key", "secretName": "tls-key"}
    ]
    ```

* **CoreRP then retrieves full secret data via database.Client and returns to user:**
    ```json
    {
      "gateway": "api-gateway",
      "connected_secrets": [
        {
          "name": "tls-cert",
          "type": "Applications.Core/secret",
          "properties": {
            "type": "certificate",
            "domain": "wrong-domain.com",
            "expires": "2024-12-31T23:59:59Z"
          }
        },
        {
          "name": "tls-key", 
          "type": "Applications.Core/secret",
          "properties": {
            "type": "private-key",
            "domain": "wrong-domain.com"
          }
        }
      ]
    }
    ```

* **User Benefit:** User immediately sees all certificate secrets connected to their gateway and discovers the wrong domain configuration, enabling quick troubleshooting without manually checking each secret individually.

### Design

#### High-Level Design

1.  **Introduce Graph Access Layer (GAL):** The GAL will be integrated as an internal service within Radius components responsible for managing the application graph, working alongside the existing `database.Client` interface for resource storage.
2.  **Dual Storage Architecture:** Resources will continue to be stored using the current `database.Client` interface (etcd via Kubernetes APIServer, or Postgres), while application graph relationships will be managed by the GAL with graph database backends.
3.  **Pluggable Backend Support:** The GAL will support both Kùzu (for dev/test/POC) and Postgres with Apache AGE (for production), with configuration to select the backend.
4.  **Schema Definition:** A formal schema for application graph entities (Applications, Environments, Resources as nodes) and their relationships (as edges with types and properties) will be defined and enforced in both graph backends.
5.  **Minimal Graph Data Storage:** The GAL will store only essential resource metadata in the graph database (resource ID, type, connections, and optional properties needed for query filtering), with full resource data retrieval continuing through the existing `database.Client` interface based on graph query results.
6.  **Data Synchronization:** The GAL will maintain synchronization between resource changes (via `database.Client`) and their corresponding minimal graph representations, ensuring the application graph accurately reflects the current state of resource relationships and queryable properties.
7.  **Component Updates:** Radius components that currently perform application graph operations (traversals, relationship queries) will be updated to use the GAL for graph queries, then retrieve full resource data via `database.Client` based on the graph results, while continuing to use `database.Client` directly for individual resource CRUD operations.

**Storage Strategy Rationale:** We chose to maintain the existing `database.Client` interface for resource storage because: (1) The current Postgres and etcd backends already provide excellent performance for individual resource operations; (2) Graph databases are optimized for relationship queries, not necessarily single-record lookups; (3) This approach allows incremental migration with lower risk; (4) Different data types (individual resources vs. relationships) have different access patterns and requirements; (5) Production deployments can continue using battle-tested storage patterns while gaining graph capabilities for application relationship queries. **The GAL will store only minimal resource metadata (ID, type, connections, and optional query-filtering properties) in the graph database, with complete resource data retrieval continuing through the existing `database.Client` based on graph query results.**

#### Architecture Diagram

```mermaid
graph TD
    A["Radius CLI/API <br/>(User Interactions)"]
    B["Radius CoreRP<br/>"]
    C["database.Client <br/>(Resource Storage)"]
    D["Graph Access Layer <br/>(Application Graph)"]
    E1["etcd (Current) | Postgres <br/>(Production)"]
    F["Cypher-compliant Graph DB <br/>(Kùzu, Postgres+AGE, Neo4j, etc.)"]

    A <--> B
    B <-->|"resource CRUD"| C
    B <-->|"graph queries"| D
    C --> E1
    D --> F
```

* **Current (Simplified):** Radius Core Components <-> database.Client <-> etcd/Postgres
* **Proposed:** 
  * Resource Storage: Radius Core Components <-> database.Client <-> etcd/Postgres (unchanged)
  * Application Graph: Radius Core Components <-> Graph Access Layer <-> Cypher-compliant Graph Database

#### Detailed Design

1.  **Graph Access Layer (GAL) Implementation:**
    * The GAL will be implemented as a Go service that abstracts all application graph operations, working alongside the existing `database.Client` interface.
    * It will provide a pluggable interface allowing different graph database backends.
    * Configuration will determine which backend to use (Kùzu for dev/test, Postgres+AGE for production).
    * The GAL will be responsible for maintaining synchronization between resource changes and their graph representations.

2.  **Dev/Test/POC Backend - Kùzu Integration:**
    * The Kùzu Go driver (`github.com/kuzudb/go-kuzu`) will be used.
    * Kùzu database will be initialized during `rad init`. The database file (`radius_app_graph.kuzu`) will be stored on persistent storage accessible to the Radius control plane.
    * **Performance Advantage:** As an embedded graph database, Kùzu eliminates network connection overhead between the GAL and the graph database, providing significantly faster performance for development and test scenarios compared to networked database solutions. This enables rapid iteration during development and faster test suite execution.
    * The GAL will populate the graph database by analyzing existing resource relationships stored via `database.Client`.
    * All application graph queries will use the GAL, while individual resource operations continue through `database.Client`.

3.  **Production Backend - Postgres with Apache AGE:**
    * Postgres with Apache AGE plugin will be supported for production deployments.
    * Network-based connection using standard PostgreSQL drivers with AGE extensions.
    * Support for connection pooling, high availability, and scaling. (Can be a later phase)

4.  **Schema Management:**
    * A Go module will define constants for node labels (e.g., `NodeTypeApplication`, `NodeTypeResource`) and edge labels (all edges will be type `CONNECTION`).
    * On startup, the GAL will ensure the schema (node tables, relationship tables, property definitions) exists in the backend, creating or migrating it if necessary.
    * **Minimal Data Storage:** Graph nodes will contain only essential resource metadata (ID, type, and optional properties needed for query filtering), with complete resource data remaining in the existing `database.Client` storage. API responses will use graph queries to identify relevant resources, then retrieve full resource data via `database.Client`.
    * The GAL will handle synchronization between resource updates (via `database.Client`) and their corresponding minimal graph representations.

5.  **Graph Access Layer (GAL) API:**
    * Example Go interface:
        ```go
        type GraphStore interface {
            // Node operations
            CreateNode(ctx context.Context, node Node) error
            GetNode(ctx context.Context, nodeID string) (Node, error)
            UpdateNodeProperties(ctx context.Context, nodeID string, properties map[string]interface{}) error
            DeleteNode(ctx context.Context, nodeID string) error // Handle cascading deletes for owned relationships

            // Edge operations (connections)
            CreateEdge(ctx context.Context, edge Edge) error
            GetEdge(ctx context.Context, fromNodeID, toNodeID string, edgeType string) (Edge, error) // Or a unique edge ID
            UpdateEdgeProperties(ctx context.Context, edgeID string, properties map[string]interface{}) error
            DeleteEdge(ctx context.Context, edgeID string) error

            // Query operations
            GetOutgoingNeighbors(ctx context.Context, nodeID string, edgeTypePattern string) ([]Node, error)
            GetIncomingNeighbors(ctx context.Context, nodeID string, edgeTypePattern string) ([]Node, error)
            FindPaths(ctx context.Context, startNodeID, endNodeID string, maxHops int) ([][]Node, error) // More complex queries
            ExecuteCypherQuery(ctx context.Context, query string, params map[string]interface{}) ([]map[string]interface{}, error) // For advanced internal use
        }        type Node struct {
            ID         string
            Type       string // e.g., "Applications.Core/application"
            Properties map[string]interface{} // Minimal properties for query filtering only
        }

        type Edge struct {
            ID         string // Optional, could be derived from both nodes
            FromNodeID string
            ToNodeID   string
            Type       string // e.g., "Connection"
            Properties map[string]interface{}
        }
        ```

4.  **Data Persistence and State:**
    * **Kùzu (Dev/Test/POC):** Kùzu runs embedded, so the Radius process managing it is authoritative. The database file (`radius_app_graph.kuzu`) is stored on persistent storage accessible to the Radius control plane.
    * **Postgres+AGE (Production):** Network-based connection with standard PostgreSQL high availability, clustering, and backup mechanisms.

5.  **Transaction Management:**
    * All compound operations (e.g., creating a resource node and its relationship edge) must be performed within a database transaction to ensure atomicity. The GAL will manage this.
    * Radius upgrades and rollbacks would need to coordinate with the GAL.

#### Advantages (of Graph Database via GAL for application graph operations)

* **Rich Querying:** Cypher provides significantly more powerful and expressive graph query capabilities than filtering etcd values client side for application graph traversals.
* **Performance:** For complex graph traversals (multi-hop queries, pathfinding), a graph database is likely to be much faster as it's optimized for such operations. For Radius this would be during most recipe execution as the entire graph is rendered.
* **Specialized Data Store:** Graph databases are purpose-built for graph data, leading to efficient storage and indexing for application graph structures while allowing the existing `database.Client` to continue handling individual resource storage efficiently.
* **Pluggable Backend:** The GAL will allow for Radius users to use any Cypher compatible graph database such as Neo4J or CosmosDB with Gremlin for application graph operations.
* **Transactional Guarantees:** Both Kùzu and Postgres with Apache AGE provide ACID transactions for graph operations, ensuring application graph consistency during complex deployments.
* **Schema Enforcement:** Better ability to define and enforce an application graph schema separate from individual resource schemas.
* **Support for streaming monitoring of graph changes:** A project like Drasi can consume a change feed of the Radius application graph because graph databases can provide relationship-aware change streams, eliminating the need for extensive middleware to parse custom Radius data structures.
* **Simplified Go Code:** Eliminates the complex imperative Go code currently required for creating and traversing application graph relationships, replacing it with declarative Cypher queries that are more maintainable and less error-prone.
* **Separation of Concerns:** Application graph operations and individual resource storage can be optimized independently, with each using the most appropriate storage technology.

---

**Modeling Deep Relationships Instead of Monolithic Types**

Currently, many Radius resource types (such as databases) are modeled as monolithic objects with embedded properties or sub-objects. For example, a database resource might have a `credentials` object, which itself contains `username` and `password` properties. This approach makes it difficult to express and traverse relationships between resources, and limits reusability and visibility in the application graph.

With a graph database, these relationships can be modeled explicitly. Instead of embedding credentials as an object within the database resource, the database resource can be connected to a separate `credentials` resource(e.g., named `db_creds` of type `Credentials`). This credentials resource can then be connected to two `secret` resources representing the username and password. This approach enables:

- **Intuitive Use:** Resources can be accessed within the recipe context in an intuitive manner.
- **Visibility:** Relationships between deeply nested resources are explicit and queryable.
- **Extensibility:** New types of relationships or properties can be added without changing the monolithic resource schema. Future API versions could allow some properties to be private (not exposed by connection). For example, a new edge type "USED_BY" could link a recipe to all the resources using it across multiple environments and applications, enabling infrastructure operators to understand the impact of registering a new version of a recipe by querying which resources would be affected.


**Example Graph Structure:**
```mermaid
graph TD
    db[Database]
    creds[Credentials: db_creds]
    user[Secret: username]
    pass[Secret: password]

    db -- "CONNECTION" --> creds
    creds -- "CONNECTION" --> user
    creds -- "CONNECTION" --> pass
```

In this model:
- The `Database` node is connected to a `Credentials` node via a CONNECTION.
- The `Credentials` node is connected to two `Secret` nodes (for username and password) via CONNECTIONs.

This structure enables richer queries and recipe author use cases like `context.connected_resources.database.credentials.username` instead of only being able to access the embedded `credentials` object and requiring the recipe author to parse.
Additionally it provides a better separation of concerns, and a more flexible, maintainable application graph.

#### Disadvantages (of Graph Database integration)

* **New Dependency:** Introduces a graph database as a new core dependency for Radius, including its Go driver(s).
* **Operational Overhead:**
    * Managing the graph database file or instance (backups, storage).
    * Monitoring performance and health.
    * Requires persistent volume or managed database for production usage.
* **Complexity:** Adds a new layer (GAL, graph database integration) to the Radius architecture.
* **Learning Curve:** Radius developers might need to learn Cypher and the specifics of the chosen graph database(s).

#### Proposed Option

Integrate a **Graph Database** behind a **Graph Access Layer (GAL)** with pluggable backend support. Initially provide **Kùzu as an embedded graph database** for dev/test/POC environments and **Postgres with Apache AGE** for production environments. This approach balances the benefits of dedicated graph DB capabilities with flexibility in deployment scenarios.

### API design

The primary API change will be internal, within the Graph Access Layer (GAL) as described in "Detailed Design." External Radius APIs (e.g., `rad resource list`, `rad application graph`) should remain functionally the same, but their implementation will now call the GAL instead of directly querying etcd.

No changes to the public Radius REST API are anticipated initially, other than potential performance improvements or new (future) API endpoints that leverage advanced graph queries.

### CLI Design

* Existing `rad` CLI commands should continue to work transparently.

### Implementation Details


#### Core RP (Resource Provider)

* Core RP will continue to use the existing `database.Client` interface for all individual resource storage, retrieval, and full CRUD operations.
* Core RP will use the GAL to perform application graph queries (finding connected resources, traversing relationships), then retrieve complete resource data via `database.Client` based on the resource IDs returned from graph queries.
* The GAL will be responsible for maintaining synchronization between resource changes (via `database.Client`) and their corresponding minimal representations in the application graph (ID, type, connections, and essential query properties only).
* **API Response Pattern:** Graph queries identify relevant resources → Full resource data retrieved via `database.Client` → Complete API responses assembled from full resource data.

### Error Handling

* The GAL will be responsible for translating backend-specific errors into standardized Radius errors.
* Errors such as database connection issues, query failures, transaction rollbacks, or schema violations must be handled gracefully.
* Retry mechanisms for transient errors will be implemented in the GAL.
* The GAL will integrate with the Radius OpenTelemetry implementation.

### Test plan

1.  **Unit Tests:**
    * Test individual functions within the Graph Access Layer (mocking graph database drivers).
    * Test schema creation and migration logic for both backends.
2.  **Integration Tests:**
    * Test the GAL against actual backend instances (both Kùzu and Postgres with Apache AGE).
    * Verify CRUD operations for nodes and edges with various property types.
    * Test transactional behavior for both backends.    * Test Core RP interacting with the GAL-backed graph stores.
    * Verify that graph queries return correct resource IDs and that subsequent `database.Client` retrievals return complete resource data.
3.  **End-to-End (E2E) Tests:**
    * Adapt existing Radius E2E tests to ensure all application deployment and management scenarios function correctly with both graph backends.
    * Verify that resource operations via `database.Client` continue to work unchanged.
    * Test that application graph operations via GAL work correctly alongside resource operations.
    * Verify that API responses contain complete resource data assembled from graph queries + `database.Client` retrieval.
    * Include tests for data synchronization between resource storage and minimal graph representation.
    * Include tests for data persistence across Radius restarts and upgrades/rollbacks.
4.  **Performance Tests:**
    * Benchmark graph read/write operations with both backends against the current key/value based implementation for representative workloads.
    * Validate performance claims from the advantages section, specifically:
        * Complex graph traversal performance compared to etcd + client-side filtering
        * Recipe execution performance when rendering the entire graph
        * Query performance for large application graphs (100+ resources)
    * Test concurrent access to the graph for both backends.
    * Add checks to LRT Cluster for graph operations.
5.  **Backup/Restore Tests:**
    * Verify that database backups can be successfully created and restored for both backends.
6.  **Backend Compatibility Tests:**
    * Ensure identical behavior and results across Kùzu and Postgres with Apache AGE backends.
    * Test configuration switching between backends.

### Security

* **Data at Rest:** 
  * **Kùzu:** The database file (`radius_app_graph.kuzu`) contains the application graph data. It should be protected by appropriate file system permissions on the persistent volume where it's stored. 
  * **Postgres with Apache AGE:** Standard PostgreSQL security practices apply, including encryption at rest, access controls, and network security.
  * Encryption at rest for storage should be considered, managed by the underlying infrastructure (e.g., Kubernetes PV encryption).
* **Access Control:** Access to the graph database is through the GAL within the Radius process. Standard Radius authentication and authorization mechanisms (when implemented) will protect the Radius APIs that indirectly interact with the graph database. There is no direct network exposure of Kùzu in the embedded model. PostgreSQL with Apache AGE will use a standard networked database access model.
* **Input Sanitization:** If any user-provided data is used to construct Cypher queries (even if parameterized), ensure proper parameterization is always used by the GAL to prevent injection vulnerabilities.
* **Threat Model:** The Radius threat model must be updated to have a section for the GAL.

### Compatibility (optional)

* **Backward Compatibility:**
    * For existing Radius deployments using etcd as the graph store, a migration path will be necessary. 
    * The public Radius API and CLI should remain backward compatible.
* **Data Format:** The structure of the application graph (apps, resources, properties) should remain conceptually the same, even though the storage backend changes.
* **Backend Compatibility:** The GAL ensures that both Kùzu and Postgres with Apache AGE backends provide identical functionality and behavior to Radius components.

### Monitoring and Logging

* **Logging:**
    * The Graph Access Layer should log all significant operations (e.g., graph queries, errors, transaction boundaries) at appropriate log levels.
* **Metrics:**
    * Expose metrics from the GAL in Radius OpenTelemetry:
        * Number of graph queries (per type: read/write, per backend).
        * Latency of graph queries (per backend).
        * Error rates for graph operations (per backend).
        * Database size and health metrics.
        * Transaction commit/rollback counts (per backend).

### Development plan

0.  **Phase 0: GAL (Milestone 0)**
    * Create the GAL with pluggable backend interface.
    * Implement CRUD endpoints representing Radius abstraction level graph operations.
    * Develop initial unit & integration tests for the GAL.
    * Ensure via debug logging that no components are communicating directly with etcd other than the GAL.
1.  **Phase 1: Kùzu Integration (Milestone 1)**
    * Phase 1 will be worked on with etcd code still in use, a configuration flag will be used to use the graph backend so we can ship smaller change sets and users can test if desired.
    * Set up Kùzu as an embedded dependency with pluggable architecture.
    * Define and implement Kùzu schema creation.
    * Implement robust error handling and transaction management in the GAL for Kùzu backend.
    * Add Kùzu specific tests (schema creation, backup/restore, etc).
    * Modify Radius init and upgrade processes to trigger appropriate behavior in the GAL.
    * Write idempotent migration tool for etcd => graph db.
2.  **Phase 2: Postgres with Apache AGE Integration (Milestone 2)**
    * Research Postgres with Apache AGE capabilities and integration requirements.
    * Implement Postgres with Apache AGE backend for the GAL.
    * Ensure feature parity between Kùzu and Postgres with Apache AGE backends.
    * Add comprehensive tests for Postgres with Apache AGE backend.
    * Implement configuration options for backend selection.
3.  **Phase 3: Testing & Documentation (Milestone 3)**
    * Implement backup/restore CLI commands for both backends.
    * Conduct comprehensive E2E testing, performance testing, and security review.
    * Develop documentation for operators and developers.
4.  **Phase 4: Query Enhancement (Milestone 4 - optional)**
    * Enhance GAL with more advanced query capabilities (pathfinding, complex traversals, to support new User Stories defined by product).

### Open Questions

1.  **Schema Evolution:** How will schema changes (e.g., adding new node/edge types, new properties) be managed over time with Radius upgrades? This will be critical for the GAL to handle gracefully.
2.  **Resource Footprint:** What is the typical CPU, memory, and disk I/O footprint of each backend for representative Radius graph sizes?
3.  **Dashboard:** The changes proposed here such as nested types and expanded use of connections will make the app graph both richer and larger, the existing dashboard will probably need some UX design and work in order to leverage that effectively and intuitively.

### Alternatives considered

1.  **Continue using etcd:**
    * **Advantages:** Leverages existing Kubernetes provided etcd installation and expertise. No new database dependency.
    * **Disadvantages:** Limited query capabilities, known performance bottlenecks for sizeable application graphs, nested rendering logic very manual and complex, tightly coupled to key value stores.
2.  **Other Embedded Graph Databases (e.g., a Go-native one if a mature one exists):**
    * **Advantages:** Could offer tighter integration if fully Go-native.
    * **Disadvantages:** Kùzu is chosen for dev/test for its performance, Cypher support, and active development. A pure Go alternative might lack some of these mature features or performance characteristics.
3.  **Hosted/Server-based Graph Databases (e.g., Neo4j, Dgraph as a service, NebulaGraph):**
    * **Advantages:** Mature, feature-rich, often provide built-in clustering and HA.
    * **Disadvantages:** Adds significant operational complexity (managing a separate database cluster), network latency between Radius and the DB, cost, and deviates from the goal of a more self-contained/embeddable solution for core graph logic. This proposal prioritizes decoupling and enhancing capabilities with a pluggable solution first.

### Full Storage Migration Effort Evaluation

During the design phase, we evaluated the effort required to move ALL Radius storage (not just application graph operations) to the graph database. This analysis revealed that such an approach would require:

**Effort Assessment:**
- **Timeline**: 12-18 months of development effort
- **Team Size**: 6-8 engineers
- **Risk Level**: Very High - no rollback path, unknown performance characteristics

**Key Challenges Identified:**
1. **Database Client Interface Replacement**: The unified `database.Client` interface is used by all 5 resource providers (CoreRP, DatastoresRP, DaprRP, MessagingRP, DynamicRP) and supports complex query patterns, optimistic concurrency control, and resource metadata operations that would require significant re-engineering for storage operations to be via a single component.

2. **Data Model Transformation**: Current resources are optimized for key-value storage with complex nested JSON structures, scope-based organization, and rich metadata that would require fundamental restructuring for graph storage.

3. **Performance Trade-offs**: Most Radius operations are simple CRUD operations on individual resources, where traditional databases excel. Graph databases optimize for relationship traversals but may perform worse for Radius's retrieval workload patterns.

**Conclusion**: The scoped approach (application graph only) provides the core benefits of graph database technology (enhanced relationship querying, better performance for graph traversals, support for complex connections) while maintaining the proven, optimized storage patterns for individual resource operations. This delivers significant value with much lower risk and development investment. Once Radius Extensibility has shipped many of the existing RPs will be deprecated in favor of Core Types in DynamicRP, making the work to transition to a single data store viable if we decide to do it to simplify the architecture.

### Graph Database Selection Methodology

**Based on established Radius configuration patterns, the following two-tier approach provides consistent user experience for selecting which graph database to use for a Radius installation:**

#### Tier 1: Interactive Installation Configuration

**Command:** `rad init --full`

Users are presented with an interactive menu during the full initialization process to select their preferred graph database backend:

```
? Select graph database for application graph operations:
  > Kùzu (Embedded - recommended for development/testing)
    PostgreSQL with Apache AGE (Network-based - recommended for production)
    Custom Cypher-compatible database (advanced configuration)
```

**Implementation Details:**
- Follows established pattern from `rad init --full` for AWS IRSA vs access keys, Azure Service Principal vs Workload Identity
- Default selection: Kùzu for simplicity and zero external dependencies
- Stores selection in Radius configuration for subsequent `rad install` operations
- Advanced option allows users to specify custom connection strings for other Cypher-compatible databases

#### Tier 2: Non-Interactive Installation Parameters

**Command:** `rad install kubernetes --set global.graphDatabase.*`

For automated deployments and GitOps scenarios, users can specify graph database configuration via Helm chart parameters:

**Kùzu Configuration (Default):**
```bash
rad install kubernetes --set global.graphDatabase.type=kuzu \
  --set global.graphDatabase.kuzu.persistentVolume.enabled=true \
  --set global.graphDatabase.kuzu.persistentVolume.size=10Gi
```

**PostgreSQL with Apache AGE Configuration:**
```bash
rad install kubernetes --set global.graphDatabase.type=postgresql-age \
  --set global.graphDatabase.postgresql.host=postgres.example.com \
  --set global.graphDatabase.postgresql.port=5432 \
  --set global.graphDatabase.postgresql.database=radius_graph \
  --set global.graphDatabase.postgresql.username=radius_user \
  --set global.graphDatabase.postgresql.passwordSecretName=postgres-credentials \
  --set global.graphDatabase.postgresql.sslMode=require
```

**Custom Cypher Database Configuration:**
```bash
rad install kubernetes --set global.graphDatabase.type=custom \
  --set global.graphDatabase.custom.connectionString="bolt://neo4j.example.com:7687" \
  --set global.graphDatabase.custom.credentialsSecretName=neo4j-credentials \
  --set global.graphDatabase.custom.dialect=neo4j
```

#### Configuration Schema

The GAL will support the following configuration structure in Helm values:

```yaml
global:
  graphDatabase:
    type: kuzu  # kuzu | postgresql-age | custom
    
    # Kùzu-specific configuration
    kuzu:
      persistentVolume:
        enabled: true
        size: 10Gi
        storageClass: ""
      dataDirectory: "/data/kuzu"
    
    # PostgreSQL with Apache AGE configuration
    postgresql:
      host: "localhost"
      port: 5432
      database: "radius_graph"
      username: "radius_user"
      passwordSecretName: "postgres-credentials"
      sslMode: "prefer"  # disable | prefer | require
      connectionPoolSize: 10
    
    # Custom Cypher database configuration
    custom:
      connectionString: ""
      credentialsSecretName: ""
      dialect: "neo4j"  # neo4j | amazon-neptune | others
      additionalParams: {}
```

#### Environment Variable Mapping

The GAL will read configuration from these environment variables (set by Helm chart):

```bash
# Graph database type selection
RADIUS_GRAPH_DATABASE_TYPE=kuzu

# Kùzu configuration
RADIUS_GRAPH_KUZU_DATA_DIR=/data/kuzu

# PostgreSQL configuration  
RADIUS_GRAPH_POSTGRESQL_HOST=postgres.example.com
RADIUS_GRAPH_POSTGRESQL_PORT=5432
RADIUS_GRAPH_POSTGRESQL_DATABASE=radius_graph
RADIUS_GRAPH_POSTGRESQL_USERNAME=radius_user
RADIUS_GRAPH_POSTGRESQL_PASSWORD_FILE=/etc/secrets/postgres/password
RADIUS_GRAPH_POSTGRESQL_SSL_MODE=require

# Custom configuration
RADIUS_GRAPH_CUSTOM_CONNECTION_STRING=bolt://neo4j.example.com:7687
RADIUS_GRAPH_CUSTOM_CREDENTIALS_FILE=/etc/secrets/custom/credentials
RADIUS_GRAPH_CUSTOM_DIALECT=neo4j
```

#### Design Rationale

This approach follows **established Radius patterns**:

1. **Interactive Configuration Pattern**: Mirrors `rad init --full` interactive flows for cloud provider credential configuration
2. **Installation-time Parameters**: Consistent with `rad install kubernetes --set` usage for Helm chart customization
3. **Credential Management**: Follows existing patterns for handling sensitive configuration via Kubernetes secrets
4. **Default Behavior**: Provides sensible defaults (Kùzu) while allowing production-ready alternatives (PostgreSQL+AGE)
5. **Extensibility**: Supports future Cypher-compatible databases through custom configuration

**Benefits:**
- **Consistency**: Aligns with existing Radius user experience patterns
- **Flexibility**: Supports both development (embedded Kùzu) and production (networked PostgreSQL) scenarios
- **Automation-Friendly**: Non-interactive configuration supports GitOps and CI/CD workflows
- **Progressive Disclosure**: Simple defaults with advanced options for power users

**Implementation Notes:**
- GAL initialization logic will read configuration from environment variables
- Database connections will be established during GAL startup with appropriate error handling
- Connection pooling and retry logic will be implemented for networked database options
- Schema initialization will be database-specific but abstracted through the GAL interface
