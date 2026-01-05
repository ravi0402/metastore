# Metastore: Next-Generation Data Catalog Design

## Executive Summary

Metastore is a cloud-native, enterprise-grade data catalog designed to overcome limitations of existing metadata stores like Apache Atlas, Amundsen, Open Metadata and DataHub. Built on Java and Vert.x, it provides a scalable, cost-efficient, and extensible platform for managing metadata at enterprise scale.

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
│                    Kong API Gateway                             │
│  (Routing, Rate Limiting, Auth, Caching, Logging)               │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    Application Service Layer                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │ Metadata │  │  Search  │  │  Audit   │  │  Notify  │         │
│  │ Service  │  │  Service │  │  Service │  │  Service │         │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────-┐        │
│  │   Auth   │  │  Tenant  │  │ Ingestion│  │ Background│        │
│  │  Service │  │  Service │  │  Service │  │  Task Mgr │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────-┘        │
└──────────────────────┬────────────────────────────────┬─────────┘
                       |                                │
                       |                          ┌─────▼──────────────────────────────────────────────────────────┐
                       │                          |                           Kafka                                │
                       │                          |  (Cache invalidation, Stream ingestion, Events, Notifications) │
                       |                          └─────┬──────────────────────────────────────────────────────-───┘
                       │                                |                                                 
┌──────────────────────▼────────────────────────────────▼─────────┐
│                      Data Access Layer                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │   Postgres   │  │  Aerospike   │  │ Elasticsearch│           │
│  │  (Primary)   │  │   (Cache)    │  │   (Search)   │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

### Component Details

#### 1. API Gateway Layer (Kong)
- **Kong Gateway**: Cloud-native API gateway for routing, compression
- **Kong Plugins**: 
  - JWT/OAuth2 authentication
  - Rate limiting (per-tenant, per-user)
  - Request transformation
  - Response caching
  - Logging and metrics
- **Request Context**: Tenant isolation via headers, request tracing with OpenTelemetry

**Why Kong over alternatives?**
| Feature | Kong | AWS API Gateway | Nginx |
|---------|------|-----------------|-------|
| Cloud-agnostic | ✓ | ✗ | ✓ |
| Plugin ecosystem | Rich | Limited | Moderate |
| Rate limiting | Built-in | Built-in | Manual |
| Auth plugins | JWT, OAuth2, OIDC | IAM-focused | Manual |
| Kubernetes-native | ✓ (Ingress) | ✗ | ✓ |
| Admin API | ✓ | ✓ | ✗ |
| DB-less mode | ✓ | N/A | N/A |

#### 2. Messaging Layer (Kafka)

**Apache Kafka** - Unified messaging for all event types:
- **Internal Events**: Cache invalidation, WebSocket broadcasts, task progress
- **Stream Ingestion**: CDC from external databases, bulk imports
- **Audit Events**: Immutable audit log with long retention
- **External Consumers**: Other services subscribe to metadata changes

**Topic Structure**:
```
metastore.internal.cache-invalidation  (1 day retention, compacted)
metastore.internal.notifications       (1 day retention)
metastore.ingestion.*                  (7 day retention)
metastore.audit.*                      (365 day retention)
metastore.events.*                     (30 day retention)
```

**Why Kafka as unified messaging?**
| Feature | Kafka | RabbitMQ | Redis Pub/Sub |
|---------|-------|----------|---------------|
| Durability | ✓ Always | ✓ Optional | ✗ None |
| Message replay | ✓ Built-in | ✗ No | ✗ No |
| Horizontal scaling | ✓ Partitions | ✓ Clustering | ✓ Sharding |
| Ordering | ✓ Per partition | ✓ Per queue | ✗ No |
| Consumer groups | ✓ Native | ✗ Manual | ✗ No |
| Ecosystem | Rich (Connect, Streams) | Good | Limited |

**Trade-off Acknowledged**: Kafka adds 5-10ms latency for internal events. This is acceptable because:
1. Cache TTL handles eventual staleness
2. Unified system reduces operational complexity
3. All events become replayable (useful for debugging)
4. Team only needs to learn one messaging system

#### 3. Application Layer (Microservices)

**Architecture**: Each service is an **independently deployable unit** with its own lifecycle.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Microservices Architecture                   │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │  Metadata   │  │   Search    │  │   Audit     │              │
│  │   Service   │  │   Service   │  │   Service   │              │
│  │             │  │             │  │             |              │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │
│         │                │                │                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │  Ingestion  │  │ Notification│  │    Task     │              │
│  │   Service   │  │   Service   │  │   Service   │              │
│  │             │  │             │  │             │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
└─────────────────────────────────────────────────────────────────┘
```

**Why Microservices?**

| Benefit | How It Helps Metastore |
|---------|------------------------|
| **Independent Scaling** | Ingestion needs 4x more Infra during bulk loads; Search scales based on query load |
| **Fault Isolation** | Audit service failure doesn't affect metadata reads |
| **Independent Deployments** | Deploy Search improvements without touching Metadata |
| **Technology Flexibility** | Search can use specialized libraries; Ingestion can be optimized differently |
| **Team Autonomy** | Different teams own different services with clear boundaries |
| **Targeted Resource Allocation** | Ingestion gets more CPU; Search gets more memory |

**Service Responsibilities:**

**Metadata Service** (Core)

**Metadata Service**
- CRUD operations on entities and relationships
- Version management
- Conflict resolution

**Search Service**
- Full-text search using Elasticsearch
- Faceted search (Elasticsearch does it by search + aggregations)
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

**Elasticsearch** 
- Full-text search index
- Aggregations
- Real-time indexing

---

## Challenges Addressed

### 1. Background Task Management

**Solution**: Distributed task queue with Redis and Kafka

- **Task Queue**: Redis for task distribution with prioritization
- **Event Coordination**: Kafka for task events and progress updates
- **Workers**: Stateless Java services processing tasks asynchronously
- **Features**:
  - Task prioritization (Redis sorted sets)
  - Retry with exponential backoff
  - Progress tracking via Kafka events
  - Dead letter queue (Redis)
  - Task cancellation

### 2. Authentication and Authorization

**Solution**: Multi-layered security with RBAC and ABAC

- **Authentication**:
  - JWT tokens (short-lived access, long-lived refresh)
  - API keys for programmatic access
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
- **L1**: In-memory cache (Caffeine) - 1-5ms
- **L2**: Aerospike distributed cache - 5-10ms
- **L3**: PostgreSQL - 50-100ms

**Cache Invalidation**: Kafka topic for cross-instance cache invalidation

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
- Anomaly detection

**Analytical Workloads**:
- Metadata analytics dashboard
- Usage patterns
- Data quality metrics

**Contextualized Querying**:
- Natural language queries
- Semantic search
- Recommendation engine

### 13. Deployment Models and Cost

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

## Technology Stack

### Core
- **Java**: LTS version, modern language features
- **Vert.x**: Reactive, non-blocking HTTP server and async I/O
- **PostgreSQL**: Primary datastore (RDS)
- **Aerospike**: Distributed cache

### API Gateway & Messaging
- **Kong Gateway**: API gateway for routing, authentication, rate limiting
- **Apache Kafka**: Unified messaging for all events (internal + external)

### Additional
- **Elasticsearch**: Full-text search (optional)
- **Redis**: Task queue and session storage
- **Kubernetes**: Orchestration
- **Prometheus**: Metrics
- **Grafana**: Dashboards
- **Jaeger**: Distributed tracing

### Libraries
- **Jackson**: JSON processing
- **Caffeine**: High-performance in-memory cache (L1)
- **JWT**: Authentication tokens
- **Apollo GraphQL**: GraphQL server
- **gRPC Java**: gRPC server

---

## Deployment Model

### Kubernetes Deployment

```yaml
# Deployment structure (Microservices)
- Namespace: metastore
- Ingress:
  - Kong Ingress Controller (API Gateway, routing to services)
- Deployments (each with independent HPA):
  - metadata-service    
  - search-service      
  - audit-service       
  - ingestion-service  
  - notification-service
  - task-service        
- StatefulSets:
  - PostgreSQL (or use managed RDS)
  - Aerospike cluster
  - Kafka cluster (or use managed MSK/Confluent)
- ConfigMaps: Per-service config, Kong plugins, shared config
- Secrets: Database credentials, API keys, JWT secrets (per-service access)
- HPA: Per-service auto-scaling based on service-specific metrics
```

**Why separate deployments?**
- **Independent scaling**: Ingestion scales 10x during bulk loads; others stay stable
- **Fault isolation**: One service crash doesn't take down the system
- **Rolling updates**: Deploy Search improvements without touching Metadata
- **Resource optimization**: Ingestion gets CPU-heavy nodes; Search gets memory-heavy nodes

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
   - Use spot instances for services and workers (70% savings)
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
- Tenant resource usage

### Logging
- Structured JSON logs
- Log aggregation (ELK)
- Log levels per environment

### Tracing (Jaeger)
- Distributed tracing across services
- Request correlation IDs
- Performance profiling

### Alerts
- High error rate
- Database connection exhaustion
- High storage utilization
- Cache miss rate threshold
- Task queue backlog
- Consumer Lag

---

## Evaluation of Existing Systems

### Apache Atlas
**Limitations**:
- Tightly coupled with Hadoop ecosystem
- Limited scalability (single JVM)
- Complex deployment
- Older API design
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
- Limited caching
- Resource intensive
- Complex deployment

**Improvements in Metastore**:
- Kafka for unified messaging (internal + external events)
- Multi-tier caching
- Resource efficient
- Kubernetes-native deployment with Kong API Gateway

---

## Architectural Trade-offs

### 1. Consistency vs. Performance
- **Choice**: Eventual consistency for cache/search
- **Rationale**: Metadata updates are not time-critical, read performance is

### 2. SQL vs. NoSQL
- **Choice**: PostgreSQL (SQL) for primary store
- **Rationale**: ACID guarantees, relationships, audit requirements

### 3. Monolith vs. Microservices
- **Choice**: Microservices architecture
- **Rationale**: 
  - **Independent Scaling**: Each service scales based on its specific load patterns
  - **Fault Isolation**: Failure in one service doesn't cascade to others
  - **Independent Deployments**: Ship features faster without coordinated releases
  - **Resource Optimization**: Allocate resources precisely where needed

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

