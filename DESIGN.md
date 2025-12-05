# Metastore: Next-Generation Data Catalog Design

## Executive Summary

Metastore is a cloud-native, enterprise-grade data catalog designed to overcome limitations of existing metadata stores like Apache Atlas, Amundsen, Open Metadata, and DataHub. Built on Java and Vert.x, it provides a scalable, cost-efficient, and extensible platform for managing metadata at enterprise scale (billions of assets).

## Table of Contents

1. [System Overview](#system-overview)
2. [Architecture](#architecture)
3. [Challenges Addressed](#challenges-addressed)
4. [Data Model](#data-model)
5. [API Design](#api-design)
6. [Technology Stack](#technology-stack)
7. [Deployment Model](#deployment-model)
8. [Cost Optimization](#cost-optimization)
9. [Observability](#observability)
10. [Evaluation of Existing Systems](#evaluation-of-existing-systems)

---

## System Overview

Metastore is designed as a distributed, multi-tenant metadata catalog that supports:
- **Scale**: Billions of metadata assets
- **Performance**: Sub-100ms query latency for common operations
- **Reliability**: 99.9% uptime SLA
- **Multi-tenancy**: Native tenant isolation
- **Extensibility**: Pluggable datastores and ingestion mechanisms
- **Cloud-agnostic**: Runs on AWS, GCP, Azure with minimal changes

---

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Layer                            │
│  (REST API, GraphQL, gRPC, WebSocket, SDKs)                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                      API Gateway Layer                          │
│  (Rate Limiting, Authentication, Request Routing)               │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    Application Service Layer                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │ Metadata │  │  Search  │  │  Audit   │  │  Notify  │         │
│  │ Service  │  │  Service │  │  Service │  │  Service │         │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │   Auth   │  │  Tenant  │  │  Ingestion│ │  Background│       │
│  │  Service │  │  Service │  │  Service │  │  Task Mgr │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘         │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                      Data Access Layer                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │   Postgres   │  │  Aerospike   │  │  Elasticsearch│          │
│  │  (Primary)   │  │   (Cache)    │  │   (Search)   │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

### Component Details

#### 1. API Gateway Layer
- **Vert.x Router**: Handles HTTP routing, CORS, compression
- **Authentication Middleware**: Validates JWT tokens, API keys
- **Rate Limiting**: Per-tenant rate limits
- **Request Context**: Tenant isolation, request tracing

#### 2. Application Services

**Metadata Service**
- CRUD operations on entities and relationships
- Version management
- Conflict resolution

**Search Service**
- Full-text search using Elasticsearch
- Faceted search
- Graph traversal queries

**Audit Service**
- Change tracking
- Compliance logging
- Query audit trails

**Notification Service**
- Event-driven notifications
- Webhook support
- Real-time updates via WebSocket

**Authentication Service**
- OAuth2/OIDC integration
- API key management
- Role-based access control (RBAC)

**Tenant Service**
- Tenant provisioning
- Resource isolation
- Quota management

**Ingestion Service**
- Batch ingestion (bulk imports)
- Stream ingestion (Kafka/event-driven)
- Pluggable connectors

**Background Task Manager**
- Async job processing
- Task scheduling
- Retry mechanisms
- Progress tracking

#### 3. Data Access Layer

**PostgreSQL (RDS)**
- Primary datastore for transactional data
- ACID guarantees
- Relationships and constraints
- Audit logs

**Aerospike**
- Distributed cache for hot data
- Sub-millisecond read latency
- TTL-based eviction
- Cache-aside pattern

**Elasticsearch** (Optional, for search)
- Full-text search index
- Aggregations
- Real-time indexing

---

## Challenges Addressed

### 1. Background Task Management

**Solution**: Distributed task queue with Vert.x event bus

- **Task Queue**: Redis/RabbitMQ for task distribution
- **Workers**: Vert.x verticles processing tasks asynchronously
- **Features**:
  - Task prioritization
  - Retry with exponential backoff
  - Progress tracking
  - Dead letter queue
  - Task cancellation

**Implementation**:
```java
// Task types: BULK_INGESTION, METADATA_SYNC, INDEX_REBUILD, etc.
// Workers scale horizontally based on queue depth
```

### 2. Authentication and Authorization

**Solution**: Multi-layered security with RBAC and ABAC

- **Authentication**:
  - JWT tokens (short-lived access, long-lived refresh)
  - API keys for programmatic access
  - OAuth2/OIDC for SSO
  - Service-to-service authentication

- **Authorization**:
  - Role-Based Access Control (RBAC)
  - Attribute-Based Access Control (ABAC)
  - Resource-level permissions
  - Tenant-level isolation

**Permission Model**:
```
Tenant → Users → Roles → Permissions → Resources
```

### 3. Multiple Stores and Consistency

**Solution**: Event-driven consistency with eventual consistency model

- **Primary Store**: PostgreSQL (source of truth)
- **Cache Store**: Aerospike (read optimization)
- **Search Store**: Elasticsearch (search optimization)
- **Consistency Strategy**:
  - Write-through to PostgreSQL
  - Async propagation to cache and search
  - Cache invalidation on updates
  - Conflict resolution via version vectors

**Interfaces**:
- REST API (primary)
- GraphQL (flexible queries)
- gRPC (high-performance)
- WebSocket (real-time)

### 4. Entities and Relationships

**Solution**: Graph-based data model with type system

**Core Entities**:
- **Asset**: Base entity (tables, dashboards, pipelines, etc.)
- **Schema**: Structure definitions
- **Lineage**: Data flow relationships
- **Tag**: Classification and discovery
- **Glossary**: Business terminology
- **User**: People and service accounts

**Relationships**:
- `CONTAINS` (schema → table)
- `DEPENDS_ON` (table → table)
- `PRODUCES` (pipeline → table)
- `TAGGED_WITH` (asset → tag)
- `OWNED_BY` (asset → user)

### 5. Audits

**Solution**: Immutable audit log with compliance features

- **Audit Events**: All mutations logged
- **Storage**: Separate audit table in PostgreSQL + S3 for long-term
- **Features**:
  - Who, what, when, why
  - Before/after values
  - Query audit trails
  - Compliance exports (GDPR, SOC2)
  - Retention policies

### 6. Tenancy Support OOTB

**Solution**: Native multi-tenancy with resource isolation

- **Tenant Isolation**:
  - Row-level security in PostgreSQL
  - Tenant ID in all queries
  - Separate cache namespaces
  - Resource quotas per tenant

- **Tenant Management**:
  - Self-service provisioning
  - Billing integration
  - Usage metrics
  - Tenant-specific configurations

### 7. CNCF Nature - Cloud Agnostic

**Solution**: Kubernetes-native with cloud-agnostic abstractions

- **Containerization**: Docker images
- **Orchestration**: Kubernetes (EKS, GKE, AKS)
- **Service Mesh**: Istio/Linkerd for traffic management
- **Storage**: CSI drivers for cloud storage
- **Secrets**: External Secrets Operator
- **Configuration**: ConfigMaps, environment variables

**Cloud-Specific Adaptations**:
- AWS: RDS, ElastiCache, S3
- GCP: Cloud SQL, Memorystore, Cloud Storage
- Azure: Azure Database, Redis Cache, Blob Storage

### 8. Distributed Caching

**Solution**: Multi-tier caching strategy

**Why Caching?**
- Metadata read-heavy workload (90% reads)
- Reduce database load
- Sub-10ms response times for hot data

**Cache Strategy**:
- **L1**: In-memory cache (Vert.x local map) - 1-5ms
- **L2**: Aerospike distributed cache - 5-10ms
- **L3**: PostgreSQL - 50-100ms

**Cache Patterns**:
- Cache-aside (lazy loading)
- Write-through for critical updates
- TTL-based expiration
- Cache invalidation on mutations

### 9. Modes of Ingestion

**Solution**: Dual-mode ingestion (batch + stream)

**Batch Ingestion**:
- Bulk CSV/JSON imports
- Scheduled jobs
- ETL pipeline integration
- Progress tracking

**Stream Ingestion**:
- Kafka/Event-driven
- Real-time metadata updates
- Change data capture (CDC)
- Backpressure handling

**Pluggable Connectors**:
- Database connectors (JDBC)
- Cloud storage (S3, GCS, Azure Blob)
- BI tools (Tableau, PowerBI)
- Data platforms (Snowflake, BigQuery)

### 10. Notifications

**Solution**: Event-driven notification system

- **Channels**:
  - WebSocket (real-time)
  - Webhooks (HTTP callbacks)
  - Email (SMTP)
  - Slack/Teams integration

- **Event Types**:
  - Asset changes
  - Lineage updates
  - Access requests
  - Compliance alerts

### 11. Scalability and Extensibility

**Solution**: Microservices architecture with plugin system

**Scalability**:
- Horizontal scaling (stateless services)
- Database read replicas
- Sharding by tenant (future)
- Auto-scaling based on metrics

**Extensibility**:
- Plugin architecture for custom entities
- Custom relationship types
- Pluggable authentication providers
- Custom ingestion connectors
- Webhook integrations

### 12. Innovation with AI and Analytics

**Solution**: AI-powered metadata enrichment

**Generative AI Integration**:
- Auto-tagging using LLMs
- Description generation
- Query understanding
- Anomaly detection

**Analytical Workloads**:
- Metadata analytics dashboard
- Usage patterns
- Data quality metrics
- Lineage impact analysis

**Contextualized Querying**:
- Natural language queries
- Semantic search
- Recommendation engine

### 13. Pluggable Datastores

**Solution**: Abstract storage interface with implementations

**Storage Interface**:
```java
interface MetadataStore {
    CompletableFuture<Entity> create(Entity entity);
    CompletableFuture<Entity> read(String id);
    CompletableFuture<Entity> update(Entity entity);
    CompletableFuture<Void> delete(String id);
}
```

**Implementations**:
- PostgreSQL (default)
- MongoDB (document store)
- Neo4j (graph database)
- DynamoDB (NoSQL)

### 14. Deployment Models and Cost

**Solution**: Flexible deployment with cost optimization

**Deployment Models**:
- **SaaS**: Multi-tenant cloud deployment
- **Dedicated**: Single-tenant cloud
- **On-Premise**: Self-hosted Kubernetes

**Cost Optimization**:
- Reserved instances for databases
- Spot instances for workers
- Auto-scaling to zero for dev environments
- Data tiering (hot/warm/cold)
- Compression for audit logs

---

## Data Model

### Entity-Relationship Diagram

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   Tenant    │─────────│    User     │─────────│    Role     │
└─────────────┘         └─────────────┘         └─────────────┘
      │                        │                        │
      │                        │                        │
      ▼                        ▼                        ▼
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   Asset     │─────────│  Lineage    │─────────│    Tag      │
└─────────────┘         └─────────────┘         └─────────────┘
      │                        │                        │
      │                        │                        │
      ▼                        ▼                        ▼
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   Schema    │         │  Glossary   │         │   Audit     │
└─────────────┘         └─────────────┘         └─────────────┘
```

### Core Tables (PostgreSQL)

#### tenants
- id (UUID, PK)
- name (VARCHAR)
- status (ENUM)
- created_at (TIMESTAMP)
- config (JSONB)

#### assets
- id (UUID, PK)
- tenant_id (UUID, FK)
- type (VARCHAR) -- TABLE, DASHBOARD, PIPELINE, etc.
- name (VARCHAR)
- qualified_name (VARCHAR, UNIQUE)
- metadata (JSONB)
- version (INT)
- created_at (TIMESTAMP)
- updated_at (TIMESTAMP)
- created_by (UUID, FK)

#### relationships
- id (UUID, PK)
- tenant_id (UUID, FK)
- source_asset_id (UUID, FK)
- target_asset_id (UUID, FK)
- type (VARCHAR) -- DEPENDS_ON, CONTAINS, etc.
- metadata (JSONB)
- created_at (TIMESTAMP)

#### tags
- id (UUID, PK)
- tenant_id (UUID, FK)
- name (VARCHAR)
- category (VARCHAR)
- created_at (TIMESTAMP)

#### asset_tags
- asset_id (UUID, FK)
- tag_id (UUID, FK)
- PRIMARY KEY (asset_id, tag_id)

#### audits
- id (UUID, PK)
- tenant_id (UUID, FK)
- entity_type (VARCHAR)
- entity_id (UUID)
- action (VARCHAR) -- CREATE, UPDATE, DELETE, READ
- user_id (UUID, FK)
- before_state (JSONB)
- after_state (JSONB)
- timestamp (TIMESTAMP)
- ip_address (INET)

#### users
- id (UUID, PK)
- tenant_id (UUID, FK)
- email (VARCHAR, UNIQUE)
- name (VARCHAR)
- roles (JSONB)
- created_at (TIMESTAMP)

#### background_tasks
- id (UUID, PK)
- tenant_id (UUID, FK)
- type (VARCHAR)
- status (ENUM) -- PENDING, RUNNING, COMPLETED, FAILED
- payload (JSONB)
- result (JSONB)
- created_at (TIMESTAMP)
- started_at (TIMESTAMP)
- completed_at (TIMESTAMP)
- retry_count (INT)

---

## API Design

### REST API

#### Base URL: `/api/v1`

#### Authentication
```
POST /auth/login
POST /auth/refresh
POST /auth/logout
```

#### Assets
```
GET    /assets                    # List assets (with filters)
POST   /assets                    # Create asset
GET    /assets/{id}               # Get asset
PUT    /assets/{id}               # Update asset
DELETE /assets/{id}               # Delete asset
GET    /assets/{id}/lineage       # Get lineage
POST   /assets/{id}/tags          # Add tags
```

#### Search
```
GET    /search?q={query}          # Full-text search
GET    /search/facets             # Faceted search
```

#### Lineage
```
GET    /lineage/{assetId}         # Get lineage graph
POST   /lineage                   # Create relationship
```

#### Background Tasks
```
POST   /tasks                     # Create task
GET    /tasks/{id}                # Get task status
GET    /tasks/{id}/progress       # Get progress
DELETE /tasks/{id}                # Cancel task
```

#### Ingestion
```
POST   /ingestion/batch           # Batch ingestion
POST   /ingestion/stream          # Stream ingestion endpoint
```

### GraphQL API

```graphql
type Query {
  asset(id: ID!): Asset
  assets(filter: AssetFilter, pagination: Pagination): AssetConnection
  search(query: String!): [Asset]
  lineage(assetId: ID!, depth: Int): LineageGraph
}

type Mutation {
  createAsset(input: AssetInput!): Asset
  updateAsset(id: ID!, input: AssetInput!): Asset
  deleteAsset(id: ID!): Boolean
  createRelationship(input: RelationshipInput!): Relationship
}

type Asset {
  id: ID!
  name: String!
  type: AssetType!
  metadata: JSON
  tags: [Tag!]
  lineage: LineageGraph
}
```

### gRPC API

```protobuf
service MetadataService {
  rpc GetAsset(GetAssetRequest) returns (Asset);
  rpc ListAssets(ListAssetsRequest) returns (ListAssetsResponse);
  rpc CreateAsset(CreateAssetRequest) returns (Asset);
  rpc UpdateAsset(UpdateAssetRequest) returns (Asset);
  rpc DeleteAsset(DeleteAssetRequest) returns (Empty);
  rpc GetLineage(GetLineageRequest) returns (LineageGraph);
}
```

---

## Technology Stack

### Core
- **Java 17**: LTS version, modern language features
- **Vert.x 4.x**: Reactive, non-blocking framework
- **PostgreSQL 15**: Primary datastore (RDS)
- **Aerospike**: Distributed cache

### Additional
- **Elasticsearch**: Full-text search (optional)
- **Redis**: Task queue (optional, can use Vert.x event bus)
- **Kafka**: Event streaming (for stream ingestion)
- **Kubernetes**: Orchestration
- **Prometheus**: Metrics
- **Grafana**: Dashboards
- **Jaeger**: Distributed tracing

### Libraries
- **Jackson**: JSON processing
- **HikariCP**: Connection pooling
- **JWT**: Authentication tokens
- **GraphQL Java**: GraphQL server
- **gRPC Java**: gRPC server

---

## Deployment Model

### Kubernetes Deployment

```yaml
# Deployment structure
- Namespace: metastore
- Services:
  - metadata-service (Deployment)
  - search-service (Deployment)
  - ingestion-service (Deployment)
  - background-worker (Deployment)
- StatefulSets:
  - PostgreSQL (or use managed RDS)
  - Aerospike cluster
- ConfigMaps: Application config
- Secrets: Database credentials, API keys
```

### Auto-Scaling
- HPA (Horizontal Pod Autoscaler) based on CPU/memory
- VPA (Vertical Pod Autoscaler) for resource optimization
- Cluster Autoscaler for node scaling

### High Availability
- Multi-AZ deployment
- Database replication (read replicas)
- Load balancer (ALB/NLB)
- Health checks and graceful shutdown

---

## Cost Optimization

### Strategies

1. **Compute**
   - Use spot instances for workers (70% savings)
   - Reserved instances for databases (40% savings)
   - Auto-scale to zero for dev environments

2. **Storage**
   - Tiered storage (hot/warm/cold)
   - Compression for audit logs
   - Lifecycle policies (move old data to S3 Glacier)

3. **Database**
   - Read replicas for scaling reads
   - Connection pooling to reduce connections
   - Query optimization

4. **Caching**
   - Aggressive caching to reduce DB load
   - Cache warming strategies

5. **Monitoring**
   - Cost alerts
   - Resource usage dashboards
   - Right-sizing recommendations

### Estimated Costs (AWS, 1M assets, 1000 tenants)
- RDS (db.r5.xlarge): ~$300/month
- EC2 (3x t3.medium): ~$150/month
- Aerospike (cache.r6g.large): ~$200/month
- S3 (storage): ~$50/month
- **Total**: ~$700/month base + usage

---

## Observability

### Metrics (Prometheus)
- Request rate, latency, error rate
- Database connection pool usage
- Cache hit/miss ratio
- Task queue depth
- Tenant resource usage

### Logging
- Structured JSON logs
- Log aggregation (ELK/Loki)
- Log levels per environment
- PII scrubbing

### Tracing (Jaeger)
- Distributed tracing across services
- Request correlation IDs
- Performance profiling

### Alerts
- High error rate
- Database connection exhaustion
- Cache miss rate threshold
- Task queue backlog

---

## Evaluation of Existing Systems

### Apache Atlas
**Limitations**:
- Tightly coupled with Hadoop ecosystem
- Limited scalability (single JVM)
- Complex deployment
- Poor API design
- No native multi-tenancy

**Improvements in Metastore**:
- Cloud-native architecture
- Horizontal scalability
- Native multi-tenancy
- Modern APIs (REST, GraphQL, gRPC)
- Better caching strategy

### Amundsen
**Limitations**:
- Search-first design (Elasticsearch dependency)
- Limited relationship modeling
- No built-in audit
- Basic authentication

**Improvements in Metastore**:
- Flexible data model
- Comprehensive audit logging
- Advanced authorization
- Better extensibility

### Open Metadata
**Limitations**:
- Complex architecture (many components)
- Heavy resource requirements
- Limited cloud optimization
- Steep learning curve

**Improvements in Metastore**:
- Simpler architecture
- Cost-optimized
- Cloud-agnostic design
- Better developer experience

### DataHub
**Limitations**:
- Kafka dependency (complexity)
- Limited caching
- Resource intensive
- Complex deployment

**Improvements in Metastore**:
- Optional Kafka (can use simpler queues)
- Multi-tier caching
- Resource efficient
- Kubernetes-native deployment

---

## Architectural Trade-offs

### 1. Consistency vs. Performance
- **Choice**: Eventual consistency for cache/search
- **Rationale**: Metadata updates are not time-critical, read performance is

### 2. SQL vs. NoSQL
- **Choice**: PostgreSQL (SQL) for primary store
- **Rationale**: ACID guarantees, relationships, audit requirements

### 3. Monolith vs. Microservices
- **Choice**: Modular monolith (Vert.x verticles)
- **Rationale**: Simpler deployment, easier debugging, can split later

### 4. Synchronous vs. Asynchronous
- **Choice**: Async for ingestion, sync for queries
- **Rationale**: User-facing queries need immediate feedback

### 5. Cache Strategy
- **Choice**: Cache-aside with TTL
- **Rationale**: Simpler than write-through, good enough for metadata

---

## Future Enhancements

1. **Graph Database**: Neo4j for complex lineage queries
2. **ML Pipeline**: Automated metadata quality scoring
3. **Data Quality**: Integration with data quality tools
4. **Collaboration**: Comments, discussions on assets
5. **Versioning**: Full version history with diff views
6. **Sharding**: Tenant-based sharding for scale

---

## Conclusion

Metastore addresses the limitations of existing metadata stores by providing:
- **Scalability**: Horizontal scaling, caching, read replicas
- **Cost-Efficiency**: Optimized resource usage, tiered storage
- **Enterprise-Ready**: Multi-tenancy, audit, security
- **Cloud-Native**: Kubernetes, CNCF standards
- **Extensible**: Plugin architecture, multiple APIs
- **Modern**: AI integration, real-time capabilities

The architecture balances simplicity with power, ensuring it can scale from small deployments to billions of assets while remaining cost-effective and maintainable.

