# System Design — Master Index

A complete roadmap of every topic you need to crack system design interviews and build real-world distributed systems. Topics are ordered from foundational to advanced. Each link will point to a dedicated `.md` file with deep explanations, diagrams, trade-offs, and examples.

---

## 1. Foundations & Mental Models

- [What is System Design](./01-foundations/what-is-system-design.md)
- [Functional vs Non-Functional Requirements](./01-foundations/functional-vs-non-functional.md)
- [How to Approach a System Design Interview](./01-foundations/interview-approach.md)
- [Back-of-the-Envelope Estimation](./01-foundations/estimation.md)
- [Powers of Two, Latency Numbers Every Engineer Should Know](./01-foundations/latency-numbers.md)
- [Throughput vs Latency vs Bandwidth](./01-foundations/throughput-latency-bandwidth.md)
- [Scalability, Reliability, Availability, Maintainability](./01-foundations/core-properties.md)
- [Vertical vs Horizontal Scaling](./01-foundations/vertical-vs-horizontal-scaling.md)
- [Stateful vs Stateless Services](./01-foundations/stateful-vs-stateless.md)
- [Monolith vs Microservices vs Serverless](./01-foundations/monolith-microservices-serverless.md)

---

## 2. Networking Basics

- [OSI Model & TCP/IP Stack](./02-networking/osi-tcp-ip.md)
- [IP Addressing, Subnets, CIDR](./02-networking/ip-subnets-cidr.md)
- [TCP vs UDP](./02-networking/tcp-vs-udp.md)
- [DNS — How It Works](./02-networking/dns.md)
- [HTTP/1.1 vs HTTP/2 vs HTTP/3 (QUIC)](./02-networking/http-versions.md)
- [HTTPS, TLS/SSL Handshake](./02-networking/https-tls.md)
- [WebSockets](./02-networking/websockets.md)
- [Server-Sent Events (SSE)](./02-networking/sse.md)
- [Long Polling vs Short Polling](./02-networking/polling.md)
- [gRPC, Protocol Buffers, Thrift](./02-networking/grpc-protobuf.md)
- [REST vs GraphQL vs gRPC](./02-networking/api-styles.md)
- [Webhooks](./02-networking/webhooks.md)
- [CORS, CSRF, Same-Origin Policy](./02-networking/cors-csrf.md)

---

## 3. APIs & Communication

- [REST API Design Principles](./03-apis/rest-design.md)
- [GraphQL Fundamentals](./03-apis/graphql.md)
- [API Versioning Strategies](./03-apis/versioning.md)
- [API Pagination Techniques](./03-apis/pagination.md)
- [Idempotency](./03-apis/idempotency.md)
- [Rate Limiting (Token Bucket, Leaky Bucket, Fixed/Sliding Window)](./03-apis/rate-limiting.md)
- [API Gateway](./03-apis/api-gateway.md)
- [Service Mesh (Istio, Linkerd)](./03-apis/service-mesh.md)
- [BFF — Backend for Frontend](./03-apis/bff.md)
- [Synchronous vs Asynchronous Communication](./03-apis/sync-vs-async.md)

---

## 4. Databases

- [SQL vs NoSQL — When to Use What](./04-databases/sql-vs-nosql.md)
- [Relational Databases Deep Dive](./04-databases/relational-databases.md)
- [Key-Value Stores (Redis, DynamoDB)](./04-databases/key-value-stores.md)
- [Document Stores (MongoDB, CouchDB)](./04-databases/document-stores.md)
- [Wide-Column Stores (Cassandra, HBase, ScyllaDB)](./04-databases/wide-column-stores.md)
- [Graph Databases (Neo4j, Neptune)](./04-databases/graph-databases.md)
- [Time-Series Databases (InfluxDB, TimescaleDB)](./04-databases/time-series-databases.md)
- [Search Engines (Elasticsearch, OpenSearch, Solr)](./04-databases/search-engines.md)
- [Vector Databases (Pinecone, Weaviate, pgvector)](./04-databases/vector-databases.md)
- [NewSQL (CockroachDB, Spanner, TiDB)](./04-databases/newsql.md)
- [ACID vs BASE](./04-databases/acid-vs-base.md)
- [Database Indexing (B-Tree, Hash, LSM-Tree)](./04-databases/indexing.md)
- [Database Normalization & Denormalization](./04-databases/normalization.md)
- [Database Transactions & Isolation Levels](./04-databases/transactions-isolation.md)
- [Concurrency Control (Optimistic vs Pessimistic Locking)](./04-databases/concurrency-control.md)
- [MVCC — Multi-Version Concurrency Control](./04-databases/mvcc.md)
- [Replication (Master-Slave, Master-Master, Multi-Region)](./04-databases/replication.md)
- [Sharding & Partitioning](./04-databases/sharding-partitioning.md)
- [Consistent Hashing](./04-databases/consistent-hashing.md)
- [Database Federation](./04-databases/federation.md)
- [Read Replicas & Write-Through Patterns](./04-databases/read-replicas.md)
- [Connection Pooling](./04-databases/connection-pooling.md)
- [Database Migrations at Scale](./04-databases/migrations.md)
- [Change Data Capture (CDC)](./04-databases/cdc.md)
- [OLTP vs OLAP](./04-databases/oltp-vs-olap.md)
- [Data Warehouses & Data Lakes (Snowflake, BigQuery, Redshift)](./04-databases/warehouses-lakes.md)
- [Lakehouse Architecture (Delta Lake, Iceberg, Hudi)](./04-databases/lakehouse.md)

---

## 5. Caching

- [Why Cache? Cache Hierarchy](./05-caching/caching-overview.md)
- [Client-Side, CDN, Server-Side, Database Caching](./05-caching/cache-layers.md)
- [Cache Strategies (Cache-Aside, Read-Through, Write-Through, Write-Back, Write-Around)](./05-caching/cache-strategies.md)
- [Cache Eviction Policies (LRU, LFU, FIFO, ARC, TTL)](./05-caching/eviction-policies.md)
- [Cache Invalidation Patterns](./05-caching/cache-invalidation.md)
- [Distributed Caching (Redis, Memcached)](./05-caching/distributed-caching.md)
- [Redis Deep Dive (Data Structures, Pub/Sub, Streams, Persistence)](./05-caching/redis-deep-dive.md)
- [Cache Stampede, Thundering Herd, Hot Keys](./05-caching/cache-pitfalls.md)
- [CDN — Content Delivery Networks](./05-caching/cdn.md)

---

## 6. Load Balancing

- [Load Balancer Fundamentals](./06-load-balancing/load-balancer-basics.md)
- [Layer 4 vs Layer 7 Load Balancing](./06-load-balancing/l4-vs-l7.md)
- [Load Balancing Algorithms (Round Robin, Least Connections, IP Hash, Weighted, etc.)](./06-load-balancing/algorithms.md)
- [Health Checks & Failover](./06-load-balancing/health-checks.md)
- [Sticky Sessions](./06-load-balancing/sticky-sessions.md)
- [Global Server Load Balancing (GSLB)](./06-load-balancing/gslb.md)
- [Reverse Proxy vs Forward Proxy](./06-load-balancing/proxies.md)
- [Nginx, HAProxy, Envoy, AWS ELB/ALB/NLB](./06-load-balancing/lb-implementations.md)

---

## 7. Messaging, Queues & Streaming

- [Message Queues vs Pub/Sub vs Streams](./07-messaging/queue-vs-pubsub-vs-stream.md)
- [Kafka Deep Dive (Topics, Partitions, Consumer Groups)](./07-messaging/kafka.md)
- [RabbitMQ, ActiveMQ, AWS SQS, Google Pub/Sub](./07-messaging/message-brokers.md)
- [Event-Driven Architecture](./07-messaging/event-driven-architecture.md)
- [Event Sourcing](./07-messaging/event-sourcing.md)
- [CQRS — Command Query Responsibility Segregation](./07-messaging/cqrs.md)
- [Delivery Guarantees (At-Most-Once, At-Least-Once, Exactly-Once)](./07-messaging/delivery-guarantees.md)
- [Dead Letter Queues](./07-messaging/dead-letter-queues.md)
- [Stream Processing (Kafka Streams, Flink, Spark Streaming)](./07-messaging/stream-processing.md)
- [Batch vs Stream Processing](./07-messaging/batch-vs-stream.md)
- [Saga Pattern for Distributed Transactions](./07-messaging/saga-pattern.md)
- [Outbox Pattern](./07-messaging/outbox-pattern.md)

---

## 8. Distributed Systems Theory

- [CAP Theorem](./08-distributed-systems/cap-theorem.md)
- [PACELC Theorem](./08-distributed-systems/pacelc.md)
- [Consistency Models (Strong, Eventual, Causal, Read-Your-Writes, Monotonic)](./08-distributed-systems/consistency-models.md)
- [Consensus Algorithms (Paxos, Raft, ZAB)](./08-distributed-systems/consensus.md)
- [Leader Election](./08-distributed-systems/leader-election.md)
- [Distributed Locks (Redlock, ZooKeeper, etcd)](./08-distributed-systems/distributed-locks.md)
- [Clocks (Logical, Vector, Hybrid Logical Clocks)](./08-distributed-systems/clocks.md)
- [Two-Phase Commit (2PC) and Three-Phase Commit (3PC)](./08-distributed-systems/2pc-3pc.md)
- [Quorum-Based Replication](./08-distributed-systems/quorum.md)
- [Gossip Protocol](./08-distributed-systems/gossip-protocol.md)
- [Bloom Filters](./08-distributed-systems/bloom-filters.md)
- [Count-Min Sketch & HyperLogLog](./08-distributed-systems/probabilistic-data-structures.md)
- [Merkle Trees](./08-distributed-systems/merkle-trees.md)
- [CRDTs — Conflict-free Replicated Data Types](./08-distributed-systems/crdts.md)
- [Byzantine Fault Tolerance](./08-distributed-systems/bft.md)
- [Split-Brain Problem](./08-distributed-systems/split-brain.md)

---

## 9. Storage Systems

- [Block, File, and Object Storage](./09-storage/storage-types.md)
- [Distributed File Systems (HDFS, GFS)](./09-storage/distributed-file-systems.md)
- [Object Storage (S3, GCS, Azure Blob)](./09-storage/object-storage.md)
- [Storage Engines (LSM-Trees vs B-Trees)](./09-storage/storage-engines.md)
- [WAL — Write-Ahead Logging](./09-storage/wal.md)
- [Compaction & Tiered Storage](./09-storage/compaction.md)
- [Erasure Coding vs Replication](./09-storage/erasure-coding.md)

---

## 10. Scalability Patterns

- [Scaling Reads vs Scaling Writes](./10-scalability/reads-vs-writes.md)
- [Database Sharding Strategies (Range, Hash, Geo, Directory)](./10-scalability/sharding-strategies.md)
- [Hot Partition Problem](./10-scalability/hot-partitions.md)
- [Capacity Planning](./10-scalability/capacity-planning.md)
- [Auto-Scaling (Horizontal Pod Autoscaler, AWS ASG)](./10-scalability/auto-scaling.md)
- [Backpressure](./10-scalability/backpressure.md)
- [Geographically Distributed Systems (Multi-Region)](./10-scalability/multi-region.md)

---

## 11. Reliability & Resilience

- [SLA, SLO, SLI, Error Budgets](./11-reliability/sla-slo-sli.md)
- [Fault Tolerance Patterns](./11-reliability/fault-tolerance.md)
- [Circuit Breaker Pattern](./11-reliability/circuit-breaker.md)
- [Retry, Timeout, and Exponential Backoff](./11-reliability/retry-timeout-backoff.md)
- [Bulkhead Pattern](./11-reliability/bulkhead.md)
- [Graceful Degradation](./11-reliability/graceful-degradation.md)
- [Failover & Disaster Recovery](./11-reliability/failover-dr.md)
- [Chaos Engineering](./11-reliability/chaos-engineering.md)
- [Blast Radius & Cell-Based Architecture](./11-reliability/cell-architecture.md)
- [Idempotent Operations & Retries](./11-reliability/idempotency-retries.md)

---

## 12. Security

- [Authentication vs Authorization](./12-security/authn-vs-authz.md)
- [OAuth 2.0 & OpenID Connect](./12-security/oauth-oidc.md)
- [JWT — JSON Web Tokens](./12-security/jwt.md)
- [Session-Based Authentication](./12-security/sessions.md)
- [SSO — Single Sign-On (SAML, OIDC)](./12-security/sso.md)
- [API Keys, HMAC Signing](./12-security/api-keys-hmac.md)
- [RBAC, ABAC, ReBAC](./12-security/access-control.md)
- [Encryption at Rest & In Transit](./12-security/encryption.md)
- [Hashing, Salting, Password Storage (bcrypt, Argon2)](./12-security/password-storage.md)
- [Public-Key Cryptography Basics](./12-security/pki.md)
- [Secrets Management (Vault, AWS KMS, Secrets Manager)](./12-security/secrets-management.md)
- [DDoS Protection & WAF](./12-security/ddos-waf.md)
- [Zero Trust Architecture](./12-security/zero-trust.md)
- [OWASP Top 10](./12-security/owasp-top-10.md)

---

## 13. Observability

- [Logging Best Practices (Structured Logs)](./13-observability/logging.md)
- [Metrics & Time-Series (Prometheus, Datadog)](./13-observability/metrics.md)
- [Distributed Tracing (Jaeger, Zipkin, OpenTelemetry)](./13-observability/tracing.md)
- [The Three Pillars of Observability](./13-observability/three-pillars.md)
- [Alerting & On-Call](./13-observability/alerting.md)
- [Centralized Log Aggregation (ELK, Loki, Splunk)](./13-observability/log-aggregation.md)
- [Health Checks & Heartbeats](./13-observability/health-checks.md)

---

## 14. Architectural Patterns

- [Layered Architecture](./14-architecture/layered.md)
- [Hexagonal / Ports & Adapters](./14-architecture/hexagonal.md)
- [Clean Architecture / Onion Architecture](./14-architecture/clean-architecture.md)
- [Microservices Architecture](./14-architecture/microservices.md)
- [Service-Oriented Architecture (SOA)](./14-architecture/soa.md)
- [Serverless / FaaS](./14-architecture/serverless.md)
- [Event-Driven Microservices](./14-architecture/event-driven-microservices.md)
- [Strangler Fig Pattern (Monolith → Microservices)](./14-architecture/strangler-fig.md)
- [Sidecar Pattern](./14-architecture/sidecar.md)
- [Ambassador & Adapter Patterns](./14-architecture/ambassador-adapter.md)
- [Lambda vs Kappa Architecture](./14-architecture/lambda-kappa.md)
- [Domain-Driven Design (DDD)](./14-architecture/ddd.md)
- [Bounded Contexts & Aggregates](./14-architecture/bounded-contexts.md)

---

## 15. Deployment & Infrastructure

- [Containers (Docker)](./15-deployment/docker.md)
- [Container Orchestration (Kubernetes)](./15-deployment/kubernetes.md)
- [Blue-Green Deployment](./15-deployment/blue-green.md)
- [Canary Deployment](./15-deployment/canary.md)
- [Rolling Deployment](./15-deployment/rolling.md)
- [Feature Flags & Dark Launches](./15-deployment/feature-flags.md)
- [Infrastructure as Code (Terraform, Pulumi, CloudFormation)](./15-deployment/iac.md)
- [CI/CD Pipelines](./15-deployment/cicd.md)
- [Immutable Infrastructure](./15-deployment/immutable-infra.md)
- [Multi-Tenancy](./15-deployment/multi-tenancy.md)

---

## 16. Performance Engineering

- [Profiling & Benchmarking](./16-performance/profiling.md)
- [Concurrency vs Parallelism](./16-performance/concurrency-parallelism.md)
- [Threading, Async I/O, Event Loops](./16-performance/threading-async.md)
- [Connection Pooling & Keep-Alive](./16-performance/connection-pooling.md)
- [Compression (Gzip, Brotli, Zstd)](./16-performance/compression.md)
- [Serialization Formats (JSON, Protobuf, Avro, MessagePack)](./16-performance/serialization.md)
- [N+1 Query Problem](./16-performance/n-plus-one.md)
- [Batching & Debouncing](./16-performance/batching-debouncing.md)
- [Tail Latency & p99](./16-performance/tail-latency.md)

---

## 17. Big Data & Analytics

- [MapReduce](./17-big-data/mapreduce.md)
- [Hadoop Ecosystem](./17-big-data/hadoop.md)
- [Apache Spark](./17-big-data/spark.md)
- [Apache Flink](./17-big-data/flink.md)
- [ETL vs ELT](./17-big-data/etl-vs-elt.md)
- [Data Pipelines & Orchestration (Airflow, Dagster, Prefect)](./17-big-data/data-pipelines.md)
- [Data Modeling for Analytics (Star, Snowflake Schemas)](./17-big-data/dimensional-modeling.md)

---

## 18. Real-World System Design Case Studies

- [Design a URL Shortener (TinyURL/Bitly)](./18-case-studies/url-shortener.md)
- [Design a Pastebin](./18-case-studies/pastebin.md)
- [Design Twitter/X](./18-case-studies/twitter.md)
- [Design Instagram](./18-case-studies/instagram.md)
- [Design Facebook News Feed](./18-case-studies/news-feed.md)
- [Design WhatsApp / Messenger](./18-case-studies/whatsapp.md)
- [Design Slack / Discord](./18-case-studies/slack.md)
- [Design YouTube / Netflix (Video Streaming)](./18-case-studies/youtube-netflix.md)
- [Design Spotify (Music Streaming)](./18-case-studies/spotify.md)
- [Design Uber / Lyft](./18-case-studies/uber.md)
- [Design Airbnb / Booking.com](./18-case-studies/airbnb.md)
- [Design Amazon / E-Commerce Platform](./18-case-studies/amazon.md)
- [Design a Payment System (Stripe-like)](./18-case-studies/payment-system.md)
- [Design a Stock Exchange / Trading System](./18-case-studies/stock-exchange.md)
- [Design Google Search / Web Crawler](./18-case-studies/search-engine.md)
- [Design Google Maps](./18-case-studies/google-maps.md)
- [Design Google Drive / Dropbox](./18-case-studies/dropbox.md)
- [Design Google Docs (Real-time Collaboration)](./18-case-studies/google-docs.md)
- [Design Zoom / Video Conferencing](./18-case-studies/zoom.md)
- [Design Notification System](./18-case-studies/notification-system.md)
- [Design Rate Limiter](./18-case-studies/rate-limiter.md)
- [Design Distributed Cache](./18-case-studies/distributed-cache.md)
- [Design Distributed Counter](./18-case-studies/distributed-counter.md)
- [Design a Web Crawler](./18-case-studies/web-crawler.md)
- [Design Typeahead / Autocomplete](./18-case-studies/typeahead.md)
- [Design a Key-Value Store](./18-case-studies/key-value-store.md)
- [Design a Distributed Job Scheduler](./18-case-studies/job-scheduler.md)
- [Design a Distributed Logging System](./18-case-studies/logging-system.md)
- [Design a Metrics & Monitoring System](./18-case-studies/monitoring-system.md)
- [Design an Ad Click Aggregator](./18-case-studies/ad-click-aggregator.md)
- [Design a Recommendation System](./18-case-studies/recommendation-system.md)
- [Design a Search Autocomplete](./18-case-studies/search-autocomplete.md)
- [Design a Ticketing System (Ticketmaster)](./18-case-studies/ticketmaster.md)
- [Design a Food Delivery System (DoorDash)](./18-case-studies/food-delivery.md)
- [Design a Ride-Sharing Matchmaking Engine](./18-case-studies/ride-matching.md)
- [Design a Code Deployment System](./18-case-studies/code-deployment.md)
- [Design an Online Multiplayer Game Backend](./18-case-studies/multiplayer-game.md)
- [Design a Live Streaming Platform (Twitch)](./18-case-studies/twitch.md)
- [Design a Collaborative Whiteboard (Figma/Miro)](./18-case-studies/collaborative-whiteboard.md)
- [Design an Email System (Gmail)](./18-case-studies/email-system.md)
- [Design a Calendar System](./18-case-studies/calendar.md)
- [Design a To-Do App with Offline Sync](./18-case-studies/todo-offline-sync.md)
- [Design a Distributed ID Generator (Snowflake)](./18-case-studies/id-generator.md)
- [Design a Leaderboard](./18-case-studies/leaderboard.md)
- [Design a Hotel Reservation System](./18-case-studies/hotel-reservation.md)
- [Design a Parking Lot System](./18-case-studies/parking-lot.md)
- [Design an Elevator System](./18-case-studies/elevator-system.md)

---

## 19. Advanced & Specialized Topics

- [Geohashing & Quadtrees (Geo-spatial Indexing)](./19-advanced/geohashing-quadtrees.md)
- [R-Trees](./19-advanced/r-trees.md)
- [Trie Data Structure for Autocomplete](./19-advanced/trie.md)
- [Skip Lists](./19-advanced/skip-lists.md)
- [Inverted Indexes](./19-advanced/inverted-index.md)
- [PageRank Algorithm](./19-advanced/pagerank.md)
- [TF-IDF & BM25](./19-advanced/tf-idf-bm25.md)
- [Embedding-Based Retrieval (ANN, HNSW, FAISS)](./19-advanced/embedding-retrieval.md)
- [Real-Time Analytics (Apache Druid, Pinot, ClickHouse)](./19-advanced/real-time-analytics.md)
- [Edge Computing](./19-advanced/edge-computing.md)
- [Blockchain & Distributed Ledger Basics](./19-advanced/blockchain.md)
- [Peer-to-Peer Systems & DHTs](./19-advanced/p2p-dht.md)
- [WebRTC for Real-Time Media](./19-advanced/webrtc.md)
- [Quic & HTTP/3 Internals](./19-advanced/quic.md)
- [Multi-Tenant SaaS Architecture](./19-advanced/multi-tenant-saas.md)

---

## 20. Soft Skills for System Design Interviews

- [Communicating Trade-Offs Clearly](./20-interview-skills/tradeoffs.md)
- [Driving the Conversation (Drive vs Be-Driven)](./20-interview-skills/driving-conversation.md)
- [Drawing System Diagrams](./20-interview-skills/diagrams.md)
- [Handling "What if scale 10x?" Follow-Ups](./20-interview-skills/scaling-questions.md)
- [Common Mistakes to Avoid](./20-interview-skills/common-mistakes.md)
- [Senior vs Staff vs Principal Bar](./20-interview-skills/level-bars.md)

---

## How to Use This Repo

1. Start at the top — **Foundations** lay the vocabulary for everything else.
2. Walk down through Networking → Databases → Caching → Load Balancing → Messaging. These are the building blocks every system design problem reuses.
3. Read the **Distributed Systems Theory** section before tackling case studies — CAP, consensus, and consistency models will keep recurring.
4. Use the **Case Studies** to practice end-to-end. Try designing each one yourself first, then read the file.
5. Revisit **Soft Skills** before a real interview.

> Each topic file follows the same structure: *Overview → Why it matters → How it works → Trade-offs → Real-world examples → Interview tips → Further reading*.
