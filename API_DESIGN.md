# API Design

## Overview

Metastore exposes multiple API interfaces to support different use cases:
- **REST API**: Primary interface for CRUD operations
- **GraphQL API**: Flexible queries and relationships
- **gRPC API**: High-performance programmatic access
- **WebSocket API**: Real-time updates and notifications

## Base URLs

- REST: `https://api.metastore.io/v1`
- GraphQL: `https://api.metastore.io/graphql`
- gRPC: `grpc.metastore.io:443`
- WebSocket: `wss://api.metastore.io/v1/ws`

## Authentication

All APIs require authentication via:
- **JWT Bearer Token**: `Authorization: Bearer <token>`
- **API Key**: `X-API-Key: <key>`

Tenant context is automatically extracted from the token/key.

---

## REST API

### Authentication

#### Login
```http
POST /auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password"
}

Response: 200 OK
{
  "access_token": "eyJhbGc...",
  "refresh_token": "eyJhbGc...",
  "expires_in": 3600,
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "name": "John Doe",
    "roles": ["admin"]
  }
}
```

#### Refresh Token
```http
POST /auth/refresh
Content-Type: application/json

{
  "refresh_token": "eyJhbGc..."
}

Response: 200 OK
{
  "access_token": "eyJhbGc...",
  "expires_in": 3600
}
```

---

### Assets

#### List Assets
```http
GET /assets?type=TABLE&tag=PII&limit=20&offset=0&sort=updated_at:desc

Response: 200 OK
{
  "data": [
    {
      "id": "uuid",
      "type": "TABLE",
      "name": "users",
      "qualified_name": "prod.public.users",
      "description": "User table",
      "metadata": {...},
      "tags": ["PII", "Sensitive"],
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z"
    }
  ],
  "pagination": {
    "total": 100,
    "limit": 20,
    "offset": 0,
    "has_more": true
  }
}
```

#### Get Asset
```http
GET /assets/{id}

Response: 200 OK
{
  "id": "uuid",
  "type": "TABLE",
  "name": "users",
  "qualified_name": "prod.public.users",
  "description": "User table",
  "metadata": {
    "database": "prod",
    "schema": "public",
    "row_count": 1000000
  },
  "tags": ["PII", "Sensitive"],
  "schema": [
    {
      "name": "id",
      "type": "UUID",
      "nullable": false
    }
  ],
  "created_by": {
    "id": "uuid",
    "name": "John Doe"
  },
  "created_at": "2024-01-01T00:00:00Z",
  "updated_at": "2024-01-01T00:00:00Z"
}
```

#### Create Asset
```http
POST /assets
Content-Type: application/json

{
  "type": "TABLE",
  "name": "users",
  "qualified_name": "prod.public.users",
  "description": "User table",
  "metadata": {
    "database": "prod",
    "schema": "public"
  },
  "tags": ["PII"]
}

Response: 201 Created
{
  "id": "uuid",
  ...
}
```

#### Update Asset
```http
PUT /assets/{id}
Content-Type: application/json

{
  "description": "Updated description",
  "metadata": {
    "row_count": 2000000
  }
}

Response: 200 OK
{
  "id": "uuid",
  ...
}
```

#### Delete Asset
```http
DELETE /assets/{id}

Response: 204 No Content
```

#### Get Asset Lineage
```http
GET /assets/{id}/lineage?depth=2&direction=both

Response: 200 OK
{
  "asset": {
    "id": "uuid",
    "name": "users"
  },
  "upstream": [
    {
      "asset": {
        "id": "uuid",
        "name": "source_table"
      },
      "relationship": {
        "type": "DEPENDS_ON",
        "metadata": {}
      }
    }
  ],
  "downstream": [
    {
      "asset": {
        "id": "uuid",
        "name": "dashboard"
      },
      "relationship": {
        "type": "USES",
        "metadata": {}
      }
    }
  ]
}
```

---

### Relationships

#### Create Relationship
```http
POST /relationships
Content-Type: application/json

{
  "source_asset_id": "uuid",
  "target_asset_id": "uuid",
  "type": "DEPENDS_ON",
  "metadata": {
    "column_mapping": {...}
  }
}

Response: 201 Created
{
  "id": "uuid",
  "source_asset_id": "uuid",
  "target_asset_id": "uuid",
  "type": "DEPENDS_ON",
  "created_at": "2024-01-01T00:00:00Z"
}
```

#### Delete Relationship
```http
DELETE /relationships/{id}

Response: 204 No Content
```

---

### Search

#### Full-Text Search
```http
GET /search?q=user&type=TABLE&limit=20

Response: 200 OK
{
  "results": [
    {
      "asset": {
        "id": "uuid",
        "name": "users",
        "type": "TABLE"
      },
      "score": 0.95,
      "highlights": {
        "name": ["<em>user</em>s"],
        "description": ["<em>user</em> table"]
      }
    }
  ],
  "total": 10
}
```

#### Faceted Search
```http
GET /search/facets?q=table

Response: 200 OK
{
  "facets": {
    "type": {
      "TABLE": 50,
      "VIEW": 20
    },
    "tags": {
      "PII": 30,
      "Sensitive": 25
    }
  }
}
```

---

### Tags

#### List Tags
```http
GET /tags?category=PII

Response: 200 OK
{
  "data": [
    {
      "id": "uuid",
      "name": "PII",
      "category": "Sensitive",
      "description": "Personally Identifiable Information"
    }
  ]
}
```

#### Create Tag
```http
POST /tags
Content-Type: application/json

{
  "name": "PII",
  "category": "Sensitive",
  "description": "Personally Identifiable Information"
}

Response: 201 Created
{
  "id": "uuid",
  ...
}
```

---

### Background Tasks

#### Create Task
```http
POST /tasks
Content-Type: application/json

{
  "type": "BULK_INGESTION",
  "priority": 5,
  "payload": {
    "source": "s3://bucket/metadata.json",
    "format": "json"
  }
}

Response: 201 Created
{
  "id": "uuid",
  "type": "BULK_INGESTION",
  "status": "PENDING",
  "created_at": "2024-01-01T00:00:00Z"
}
```

#### Get Task Status
```http
GET /tasks/{id}

Response: 200 OK
{
  "id": "uuid",
  "type": "BULK_INGESTION",
  "status": "RUNNING",
  "progress": 45,
  "result": null,
  "created_at": "2024-01-01T00:00:00Z",
  "started_at": "2024-01-01T00:05:00Z"
}
```

#### Cancel Task
```http
DELETE /tasks/{id}

Response: 200 OK
{
  "id": "uuid",
  "status": "CANCELLED"
}
```

---

### Ingestion

#### Batch Ingestion
```http
POST /ingestion/batch
Content-Type: application/json

{
  "assets": [
    {
      "type": "TABLE",
      "name": "users",
      "qualified_name": "prod.public.users",
      "metadata": {...}
    }
  ],
  "options": {
    "upsert": true,
    "validate": true
  }
}

Response: 200 OK
{
  "processed": 10,
  "created": 8,
  "updated": 2,
  "errors": []
}
```

#### Stream Ingestion (Webhook)
```http
POST /ingestion/stream
Content-Type: application/json
X-Event-Type: asset.created

{
  "asset": {
    "type": "TABLE",
    "name": "users",
    ...
  }
}

Response: 202 Accepted
{
  "id": "uuid",
  "status": "accepted"
}
```

---

### Audit

#### Get Audit Trail
```http
GET /audits?entity_type=ASSET&entity_id=uuid&limit=50

Response: 200 OK
{
  "data": [
    {
      "id": "uuid",
      "entity_type": "ASSET",
      "entity_id": "uuid",
      "action": "UPDATE",
      "user": {
        "id": "uuid",
        "name": "John Doe"
      },
      "before_state": {...},
      "after_state": {...},
      "timestamp": "2024-01-01T00:00:00Z"
    }
  ]
}
```

---

## GraphQL API

### Schema

```graphql
type Query {
  # Assets
  asset(id: ID!): Asset
  assets(
    filter: AssetFilter
    pagination: Pagination
  ): AssetConnection
  
  # Search
  search(query: String!, filters: SearchFilters): [SearchResult]
  
  # Lineage
  lineage(assetId: ID!, depth: Int, direction: LineageDirection): LineageGraph
  
  # Tags
  tags(filter: TagFilter): [Tag]
  
  # Tasks
  task(id: ID!): BackgroundTask
  tasks(filter: TaskFilter): [BackgroundTask]
}

type Mutation {
  # Assets
  createAsset(input: AssetInput!): Asset
  updateAsset(id: ID!, input: AssetInput!): Asset
  deleteAsset(id: ID!): Boolean
  
  # Relationships
  createRelationship(input: RelationshipInput!): Relationship
  deleteRelationship(id: ID!): Boolean
  
  # Tags
  createTag(input: TagInput!): Tag
  addTagToAsset(assetId: ID!, tagId: ID!): Boolean
  removeTagFromAsset(assetId: ID!, tagId: ID!): Boolean
  
  # Tasks
  createTask(input: TaskInput!): BackgroundTask
  cancelTask(id: ID!): Boolean
}

type Subscription {
  assetUpdated(assetId: ID!): Asset
  taskProgress(taskId: ID!): TaskProgress
}

type Asset {
  id: ID!
  type: AssetType!
  name: String!
  qualifiedName: String!
  description: String
  metadata: JSON
  tags: [Tag!]!
  schema: [SchemaElement!]
  lineage: LineageGraph
  createdBy: User
  createdAt: DateTime!
  updatedAt: DateTime!
}

type Relationship {
  id: ID!
  source: Asset!
  target: Asset!
  type: RelationshipType!
  metadata: JSON
  createdAt: DateTime!
}

type LineageGraph {
  asset: Asset!
  upstream: [LineageEdge!]!
  downstream: [LineageEdge!]!
}

type LineageEdge {
  asset: Asset!
  relationship: Relationship!
}

enum AssetType {
  TABLE
  VIEW
  DASHBOARD
  PIPELINE
  DATASET
  CHART
}

enum RelationshipType {
  DEPENDS_ON
  CONTAINS
  PRODUCES
  USES
  RELATED_TO
}

input AssetInput {
  type: AssetType!
  name: String!
  qualifiedName: String!
  description: String
  metadata: JSON
  tags: [ID!]
}

input Pagination {
  limit: Int = 20
  offset: Int = 0
}

input AssetFilter {
  type: AssetType
  tags: [ID!]
  search: String
}
```

### Example Queries

#### Get Asset with Lineage
```graphql
query {
  asset(id: "uuid") {
    id
    name
    type
    description
    tags {
      id
      name
      category
    }
    lineage(depth: 2) {
      upstream {
        asset {
          id
          name
        }
        relationship {
          type
        }
      }
      downstream {
        asset {
          id
          name
        }
        relationship {
          type
        }
      }
    }
  }
}
```

#### Search with Filters
```graphql
query {
  search(query: "user", filters: { type: TABLE }) {
    asset {
      id
      name
      type
    }
    score
  }
}
```

---

## gRPC API

### Service Definition

```protobuf
syntax = "proto3";

package metastore.v1;

service MetadataService {
  // Assets
  rpc GetAsset(GetAssetRequest) returns (Asset);
  rpc ListAssets(ListAssetsRequest) returns (ListAssetsResponse);
  rpc CreateAsset(CreateAssetRequest) returns (Asset);
  rpc UpdateAsset(UpdateAssetRequest) returns (Asset);
  rpc DeleteAsset(DeleteAssetRequest) returns (Empty);
  
  // Lineage
  rpc GetLineage(GetLineageRequest) returns (LineageGraph);
  rpc CreateRelationship(CreateRelationshipRequest) returns (Relationship);
  
  // Search
  rpc Search(SearchRequest) returns (SearchResponse);
  
  // Tasks
  rpc CreateTask(CreateTaskRequest) returns (BackgroundTask);
  rpc GetTask(GetTaskRequest) returns (BackgroundTask);
}

message Asset {
  string id = 1;
  string type = 2;
  string name = 3;
  string qualified_name = 4;
  string description = 5;
  google.protobuf.Struct metadata = 6;
  repeated Tag tags = 7;
  int64 created_at = 8;
  int64 updated_at = 9;
}

message GetAssetRequest {
  string id = 1;
}

message ListAssetsRequest {
  string type = 1;
  repeated string tag_ids = 2;
  int32 limit = 3;
  int32 offset = 4;
}

message ListAssetsResponse {
  repeated Asset assets = 1;
  int32 total = 2;
}
```

---

## WebSocket API

### Connection
```javascript
const ws = new WebSocket('wss://api.metastore.io/v1/ws?token=...');
```

### Subscribe to Asset Updates
```json
{
  "type": "subscribe",
  "channel": "asset.updates",
  "asset_id": "uuid"
}
```

### Receive Updates
```json
{
  "type": "asset.updated",
  "asset_id": "uuid",
  "event": "UPDATE",
  "data": {
    "id": "uuid",
    "name": "users",
    ...
  }
}
```

### Subscribe to Task Progress
```json
{
  "type": "subscribe",
  "channel": "task.progress",
  "task_id": "uuid"
}
```

---

## Error Handling

### Error Response Format
```json
{
  "error": {
    "code": "ASSET_NOT_FOUND",
    "message": "Asset with id 'uuid' not found",
    "details": {
      "asset_id": "uuid"
    }
  }
}
```

### HTTP Status Codes
- `200 OK`: Success
- `201 Created`: Resource created
- `204 No Content`: Success, no content
- `400 Bad Request`: Invalid request
- `401 Unauthorized`: Authentication required
- `403 Forbidden`: Insufficient permissions
- `404 Not Found`: Resource not found
- `409 Conflict`: Version conflict
- `429 Too Many Requests`: Rate limit exceeded
- `500 Internal Server Error`: Server error
- `503 Service Unavailable`: Service unavailable

---

## Pagination

All list endpoints support pagination:
- `limit`: Number of items (default: 20, max: 100)
- `offset`: Offset for pagination (default: 0)

Response includes pagination metadata:
```json
{
  "pagination": {
    "total": 100,
    "limit": 20,
    "offset": 0,
    "has_more": true
  }
}
```

---

## Filtering and Sorting

### Filters
- `type`: Asset type
- `tag`: Tag ID or name
- `created_after`: ISO 8601 timestamp
- `created_before`: ISO 8601 timestamp
- `search`: Full-text search query

### Sorting
- `sort`: Field and direction (e.g., `updated_at:desc`, `name:asc`)
- Default: `updated_at:desc`

