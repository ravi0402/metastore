# Metastore - Next-Generation Data Catalog

A cloud-native, enterprise-grade data catalog designed to overcome limitations of existing metadata stores like Apache Atlas, Amundsen, Open Metadata, and DataHub.

## 📋 Overview

Metastore is built on **Java 17** and **Vert.x 4.x**, providing a scalable, cost-efficient, and extensible platform for managing metadata at enterprise scale (billions of assets).

### Key Features

- ✅ **Multi-tenancy**: Native tenant isolation out-of-the-box
- ✅ **Scalability**: Horizontal scaling to billions of assets
- ✅ **Performance**: Sub-100ms query latency with multi-tier caching
- ✅ **Cloud-agnostic**: Runs on AWS, GCP, Azure with minimal changes
- ✅ **Multiple APIs**: REST, GraphQL, gRPC, and WebSocket support
- ✅ **Enterprise-ready**: Comprehensive audit logging and compliance
- ✅ **Extensible**: Pluggable datastores and ingestion mechanisms

## 📚 Documentation

### 1. [DESIGN.md](DESIGN.md)
Comprehensive design document covering:
- System architecture (high-level and low-level)
- All 14 challenges addressed
- Technology stack and justifications
- Deployment model
- Cost optimization strategies
- Observability and monitoring
- Evaluation of existing systems

### 2. [ER_MODEL.md](ER_MODEL.md)
Detailed entity-relationship model:
- Core entities (Tenant, User, Asset, Relationship, Tag, etc.)
- Relationships and cardinalities
- Data persistence strategy
- Multi-tenancy isolation
- Example SQL queries

### 3. [API_DESIGN.md](API_DESIGN.md)
API interface specifications:
- REST API endpoints
- GraphQL schema and queries
- gRPC service definitions
- WebSocket real-time updates
- Authentication and authorization
- Error handling and rate limiting

### 4. [DIAGRAMS.md](DIAGRAMS.md)
Visual diagrams (Mermaid + PNG images):
- Entity-Relationship diagram
- System architecture diagram
- Component interaction flow
- Data flow diagram
- Deployment architecture
- Multi-tenancy isolation

## 🏗️ Architecture

### High-Level Layers

1. **Client Layer**: REST, GraphQL, gRPC, WebSocket, SDKs
2. **API Gateway Layer**: Authentication, rate limiting, routing
3. **Application Service Layer**: Metadata, Search, Audit, Notification, Auth, Tenant, Ingestion, Background Tasks
4. **Data Access Layer**: PostgreSQL (primary), Aerospike (cache), Elasticsearch (search)

### Key Components

- **Metadata Service**: CRUD operations, version management
- **Search Service**: Full-text and faceted search
- **Audit Service**: Immutable audit logs for compliance
- **Notification Service**: Event-driven notifications
- **Background Task Manager**: Async job processing
- **Multi-tier Caching**: L1 (in-memory), L2 (Aerospike), L3 (PostgreSQL)

## 🚀 Quick Start

### Prerequisites

- Java 17+
- Maven 3.8+
- PostgreSQL 15+
- (Optional) Aerospike for distributed caching
- (Optional) Elasticsearch for full-text search

### Build

```bash
mvn clean package
```

### Run

```bash
java -jar target/metastore-1.0.0-fat.jar
```

### Configuration

Create `config.json`:

```json
{
  "http": {
    "port": 8080
  },
  "database": {
    "host": "localhost",
    "port": 5432,
    "database": "metastore",
    "user": "postgres",
    "password": "postgres",
    "pool": {
      "size": 10
    }
  },
  "cache": {
    "ttl": 3600
  },
  "auth": {
    "jwt": {
      "secret": "your-secret-key"
    }
  }
}
```

## 🎯 Challenges Addressed

1. ✅ Background Task Management
2. ✅ Authentication and Authorization
3. ✅ Multiple Stores and Consistency
4. ✅ Entities and Relationships
5. ✅ Audits
6. ✅ Tenancy Support OOTB
7. ✅ CNCF Nature - Cloud Agnostic
8. ✅ Distributed Caching
9. ✅ Modes of Ingestion (Batch/Stream)
10. ✅ Notifications
11. ✅ Scalability and Extensibility
12. ✅ Innovation with AI and Analytics
13. ✅ Pluggable Datastores
14. ✅ Deployment Models and Cost Optimization

## 🛠️ Technology Stack

- **Language**: Java 17
- **Framework**: Vert.x 4.x
- **Database**: PostgreSQL 15 (RDS)
- **Cache**: Aerospike
- **Search**: Elasticsearch (optional)
- **Orchestration**: Kubernetes
- **Monitoring**: Prometheus, Grafana
- **Tracing**: Jaeger
- **Logging**: ELK/Loki

## 📊 Diagrams

All diagrams are available in:
- **Mermaid format**: `DIAGRAMS.md` (editable source)
- **PNG images**: `diagrams/images/` (rendered images)

View the diagrams:
- [Entity-Relationship Diagram](diagrams/images/01-er-diagram.png)
- [System Architecture](diagrams/images/02-system-architecture.png)
- [Component Interaction](diagrams/images/03-component-interaction.png)
- [Data Flow](diagrams/images/04-data-flow.png)
- [Deployment Architecture](diagrams/images/05-deployment-architecture.png)
- [Multi-Tenancy Isolation](diagrams/images/06-multi-tenancy.png)

## 🔐 Security

- JWT-based authentication
- Role-based access control (RBAC)
- Attribute-based access control (ABAC)
- Tenant-level isolation
- API key support for programmatic access
- Comprehensive audit logging

## 📈 Scalability

- Horizontal scaling (stateless services)
- Database read replicas
- Multi-tier caching strategy
- Sharding by tenant (future)
- Auto-scaling based on metrics

## 💰 Cost Optimization

- Reserved instances for databases
- Spot instances for workers
- Auto-scaling to zero for dev environments
- Data tiering (hot/warm/cold)
- Compression for audit logs

## 📝 License

This is a design and implementation exercise for evaluation purposes.

## 🤝 Contributing

This is a design challenge submission. For questions or feedback, please refer to the design documents.

---

**Built with ❤️ using Java, Vert.x, PostgreSQL, and Aerospike**

