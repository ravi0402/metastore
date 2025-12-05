# Metastore Diagrams

This document contains Mermaid diagrams for the Entity-Relationship model and System Architecture.

## Entity-Relationship Diagram

![ER Diagram](diagrams/images/01-er-diagram.png)

```mermaid
erDiagram
    TENANT ||--o{ USER : "has"
    TENANT ||--o{ ASSET : "contains"
    TENANT ||--o{ TAG : "owns"
    TENANT ||--o{ RELATIONSHIP : "contains"
    TENANT ||--o{ AUDIT : "logs"
    TENANT ||--o{ BACKGROUND_TASK : "executes"
    
    USER ||--o{ ASSET : "creates"
    USER ||--o{ RELATIONSHIP : "creates"
    USER ||--o{ AUDIT : "performs"
    USER ||--o{ ASSET_TAG : "assigns"
    
    ASSET ||--o{ RELATIONSHIP : "source"
    ASSET ||--o{ RELATIONSHIP : "target"
    ASSET ||--o{ SCHEMA : "has"
    ASSET }o--o{ TAG : "tagged_with"
    
    ASSET_TAG }o--|| ASSET : "references"
    ASSET_TAG }o--|| TAG : "references"
    
    TENANT {
        uuid id PK
        varchar name
        enum status
        jsonb config
        timestamp created_at
        timestamp updated_at
    }
    
    USER {
        uuid id PK
        uuid tenant_id FK
        varchar email UK
        varchar name
        jsonb roles
        varchar api_key_hash
        timestamp created_at
        timestamp last_login_at
    }
    
    ASSET {
        uuid id PK
        uuid tenant_id FK
        varchar type
        varchar name
        varchar qualified_name UK
        text description
        jsonb metadata
        int version
        uuid created_by FK
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }
    
    RELATIONSHIP {
        uuid id PK
        uuid tenant_id FK
        uuid source_asset_id FK
        uuid target_asset_id FK
        varchar type
        jsonb metadata
        uuid created_by FK
        timestamp created_at
    }
    
    TAG {
        uuid id PK
        uuid tenant_id FK
        varchar name
        varchar category
        text description
        timestamp created_at
    }
    
    ASSET_TAG {
        uuid asset_id FK
        uuid tag_id FK
        uuid created_by FK
        timestamp created_at
    }
    
    SCHEMA {
        uuid id PK
        uuid asset_id FK
        varchar name
        varchar type
        boolean nullable
        jsonb metadata
        int order
        timestamp created_at
    }
    
    AUDIT {
        uuid id PK
        uuid tenant_id FK
        varchar entity_type
        uuid entity_id
        varchar action
        uuid user_id FK
        jsonb before_state
        jsonb after_state
        jsonb metadata
        inet ip_address
        varchar user_agent
        timestamp timestamp
    }
    
    BACKGROUND_TASK {
        uuid id PK
        uuid tenant_id FK
        varchar type
        enum status
        int priority
        jsonb payload
        jsonb result
        text error_message
        int progress
        int retry_count
        int max_retries
        timestamp created_at
        timestamp started_at
        timestamp completed_at
    }
```

## System Architecture Diagram

![System Architecture](diagrams/images/02-system-architecture.png)

```mermaid
graph TB
    subgraph "Client Layer"
        REST[REST API Clients]
        GraphQL[GraphQL Clients]
        gRPC[gRPC Clients]
        WS[WebSocket Clients]
        SDK[SDKs]
    end
    
    subgraph "API Gateway Layer"
        Router[Vert.x Router]
        Auth[Authentication Middleware]
        RateLimit[Rate Limiting]
        CORS[CORS Handler]
    end
    
    subgraph "Application Service Layer"
        subgraph "Core Services"
            MetadataService[Metadata Service<br/>CRUD, Versioning]
            SearchService[Search Service<br/>Full-text, Faceted]
            AuditService[Audit Service<br/>Change Tracking]
            NotifyService[Notification Service<br/>Events, Webhooks]
        end
        
        subgraph "Support Services"
            AuthService[Auth Service<br/>JWT, OAuth2, RBAC]
            TenantService[Tenant Service<br/>Isolation, Quotas]
            IngestionService[Ingestion Service<br/>Batch, Stream]
            TaskManager[Background Task Manager<br/>Async Jobs, Scheduling]
        end
    end
    
    subgraph "Data Access Layer"
        subgraph "Primary Store"
            Postgres[(PostgreSQL RDS<br/>Primary Datastore<br/>ACID, Relationships)]
        end
        
        subgraph "Cache Layer"
            Aerospike[(Aerospike<br/>Distributed Cache<br/>Sub-ms Latency)]
        end
        
        subgraph "Search Index"
            Elasticsearch[(Elasticsearch<br/>Full-text Search<br/>Aggregations)]
        end
    end
    
    subgraph "Infrastructure"
        EventBus[Vert.x Event Bus]
        TaskQueue[Task Queue<br/>Redis/RabbitMQ]
        Monitoring[Prometheus<br/>Metrics]
        Logging[Log Aggregation<br/>ELK/Loki]
        Tracing[Jaeger<br/>Distributed Tracing]
    end
    
    REST --> Router
    GraphQL --> Router
    gRPC --> Router
    WS --> Router
    SDK --> Router
    
    Router --> Auth
    Router --> RateLimit
    Router --> CORS
    
    Auth --> MetadataService
    Auth --> SearchService
    Auth --> AuditService
    Auth --> NotifyService
    Auth --> AuthService
    Auth --> TenantService
    Auth --> IngestionService
    Auth --> TaskManager
    
    MetadataService --> Postgres
    MetadataService --> Aerospike
    MetadataService --> EventBus
    
    SearchService --> Elasticsearch
    SearchService --> Postgres
    
    AuditService --> Postgres
    
    NotifyService --> EventBus
    NotifyService --> WS
    
    AuthService --> Postgres
    
    TenantService --> Postgres
    
    IngestionService --> Postgres
    IngestionService --> EventBus
    IngestionService --> TaskQueue
    
    TaskManager --> TaskQueue
    TaskManager --> Postgres
    TaskManager --> EventBus
    
    Postgres -.Cache Invalidation.-> Aerospike
    Postgres -.Index Update.-> Elasticsearch
    
    MetadataService -.Metrics.-> Monitoring
    SearchService -.Metrics.-> Monitoring
    AuditService -.Metrics.-> Monitoring
    
    MetadataService -.Logs.-> Logging
    SearchService -.Logs.-> Logging
    
    MetadataService -.Traces.-> Tracing
    SearchService -.Traces.-> Tracing
    
    style Postgres fill:#336791,color:#fff
    style Aerospike fill:#C41E3A,color:#fff
    style Elasticsearch fill:#005571,color:#fff
    style MetadataService fill:#4CAF50,color:#fff
    style SearchService fill:#2196F3,color:#fff
    style AuditService fill:#FF9800,color:#fff
```

## Component Interaction Flow

![Component Interaction](diagrams/images/03-component-interaction.png)

```mermaid
sequenceDiagram
    participant Client
    participant API Gateway
    participant Metadata Service
    participant Cache
    participant Database
    participant Search Index
    participant Event Bus
    
    Client->>API Gateway: HTTP Request (JWT Token)
    API Gateway->>API Gateway: Validate Auth & Rate Limit
    API Gateway->>Metadata Service: Forward Request
    
    alt Read Operation
        Metadata Service->>Cache: Check Cache
        alt Cache Hit
            Cache-->>Metadata Service: Return Cached Data
            Metadata Service-->>Client: Response
        else Cache Miss
            Metadata Service->>Database: Query PostgreSQL
            Database-->>Metadata Service: Return Data
            Metadata Service->>Cache: Store in Cache
            Metadata Service-->>Client: Response
        end
    else Write Operation
        Metadata Service->>Database: Write to PostgreSQL
        Database-->>Metadata Service: Confirm Write
        Metadata Service->>Cache: Invalidate Cache
        Metadata Service->>Event Bus: Publish Change Event
        Event Bus->>Search Index: Update Search Index (Async)
        Metadata Service-->>Client: Response
    end
```

## Data Flow Diagram

![Data Flow](diagrams/images/04-data-flow.png)

```mermaid
flowchart LR
    subgraph "Ingestion"
        Batch[Batch Ingestion<br/>CSV/JSON]
        Stream[Stream Ingestion<br/>Kafka/Events]
        Connectors[Pluggable Connectors<br/>JDBC, S3, etc.]
    end
    
    subgraph "Processing"
        Validator[Validation]
        Transformer[Transformation]
        Enricher[Enrichment<br/>AI Tagging]
    end
    
    subgraph "Storage"
        Postgres[(PostgreSQL)]
        Cache[(Aerospike)]
        Search[(Elasticsearch)]
    end
    
    subgraph "Consumption"
        REST_API[REST API]
        GraphQL_API[GraphQL API]
        gRPC_API[gRPC API]
    end
    
    Batch --> Validator
    Stream --> Validator
    Connectors --> Validator
    
    Validator --> Transformer
    Transformer --> Enricher
    Enricher --> Postgres
    
    Postgres --> Cache
    Postgres --> Search
    
    Postgres --> REST_API
    Cache --> REST_API
    Search --> REST_API
    
    Postgres --> GraphQL_API
    Cache --> GraphQL_API
    Search --> GraphQL_API
    
    Postgres --> gRPC_API
    Cache --> gRPC_API
```

## Deployment Architecture

![Deployment Architecture](diagrams/images/05-deployment-architecture.png)

```mermaid
graph TB
    subgraph "Kubernetes Cluster"
        subgraph "Namespace: metastore"
            subgraph "API Layer"
                API_Pod1[API Pod 1]
                API_Pod2[API Pod 2]
                API_Pod3[API Pod N]
            end
            
            subgraph "Service Layer"
                Worker_Pod1[Worker Pod 1]
                Worker_Pod2[Worker Pod 2]
                Worker_PodN[Worker Pod N]
            end
            
            subgraph "Config"
                ConfigMap[ConfigMap]
                Secrets[Secrets]
            end
        end
    end
    
    subgraph "Managed Services"
        RDS[(RDS PostgreSQL<br/>Multi-AZ)]
        Aerospike_Cluster[(Aerospike Cluster)]
        Elasticsearch_Cluster[(Elasticsearch<br/>Managed Service)]
    end
    
    subgraph "External"
        LoadBalancer[Load Balancer<br/>ALB/NLB]
        Monitoring[Prometheus<br/>Grafana]
        Logging[CloudWatch<br/>ELK Stack]
    end
    
    LoadBalancer --> API_Pod1
    LoadBalancer --> API_Pod2
    LoadBalancer --> API_PodN
    
    API_Pod1 --> RDS
    API_Pod2 --> RDS
    API_PodN --> RDS
    
    API_Pod1 --> Aerospike_Cluster
    API_Pod2 --> Aerospike_Cluster
    API_PodN --> Aerospike_Cluster
    
    API_Pod1 --> Elasticsearch_Cluster
    API_Pod2 --> Elasticsearch_Cluster
    API_PodN --> Elasticsearch_Cluster
    
    Worker_Pod1 --> RDS
    Worker_Pod2 --> RDS
    Worker_PodN --> RDS
    
    API_Pod1 -.Metrics.-> Monitoring
    API_Pod2 -.Metrics.-> Monitoring
    Worker_Pod1 -.Metrics.-> Monitoring
    
    API_Pod1 -.Logs.-> Logging
    API_Pod2 -.Logs.-> Logging
    Worker_Pod1 -.Logs.-> Logging
    
    ConfigMap --> API_Pod1
    ConfigMap --> API_Pod2
    Secrets --> API_Pod1
    Secrets --> API_Pod2
```

## Multi-Tenancy Isolation

![Multi-Tenancy Isolation](diagrams/images/06-multi-tenancy.png)

```mermaid
graph TB
    subgraph "Tenant A"
        TenantA_User[User A1, A2, ...]
        TenantA_Assets[Assets A]
        TenantA_Data[(Tenant A Data<br/>tenant_id = 'A')]
    end
    
    subgraph "Tenant B"
        TenantB_User[User B1, B2, ...]
        TenantB_Assets[Assets B]
        TenantB_Data[(Tenant B Data<br/>tenant_id = 'B')]
    end
    
    subgraph "Shared Infrastructure"
        Postgres[(PostgreSQL<br/>Row-Level Security)]
        Cache[(Aerospike<br/>Namespace Isolation)]
        Search[(Elasticsearch<br/>Index per Tenant)]
    end
    
    TenantA_User --> TenantA_Assets
    TenantA_Assets --> TenantA_Data
    TenantA_Data --> Postgres
    TenantA_Data --> Cache
    TenantA_Data --> Search
    
    TenantB_User --> TenantB_Assets
    TenantB_Assets --> TenantB_Data
    TenantB_Data --> Postgres
    TenantB_Data --> Cache
    TenantB_Data --> Search
    
    style TenantA_Data fill:#E3F2FD
    style TenantB_Data fill:#FFF3E0
    style Postgres fill:#336791,color:#fff
```

