# Entity-Relationship Model

## Overview

The Metastore data model is designed to support flexible metadata management with strong relationships, multi-tenancy, and audit capabilities.

## Core Entities

### 1. Tenant
Represents a customer/organization using the system.

**Attributes**:
- `id` (UUID, PK): Unique identifier
- `name` (VARCHAR(255)): Tenant name
- `status` (ENUM): ACTIVE, SUSPENDED, DELETED
- `config` (JSONB): Tenant-specific configuration
- `created_at` (TIMESTAMP): Creation timestamp
- `updated_at` (TIMESTAMP): Last update timestamp

**Relationships**:
- One-to-Many with User
- One-to-Many with Asset
- One-to-Many with Tag
- One-to-Many with Audit

### 2. User
Represents a user within a tenant.

**Attributes**:
- `id` (UUID, PK): Unique identifier
- `tenant_id` (UUID, FK): Reference to Tenant
- `email` (VARCHAR(255), UNIQUE): User email
- `name` (VARCHAR(255)): Display name
- `roles` (JSONB): Array of role names
- `api_key_hash` (VARCHAR): Hashed API key (optional)
- `created_at` (TIMESTAMP): Creation timestamp
- `last_login_at` (TIMESTAMP): Last login timestamp

**Relationships**:
- Many-to-One with Tenant
- One-to-Many with Asset (as owner)
- One-to-Many with Audit (as actor)

### 3. Asset
Core entity representing any metadata asset (table, dashboard, pipeline, etc.).

**Attributes**:
- `id` (UUID, PK): Unique identifier
- `tenant_id` (UUID, FK): Reference to Tenant
- `type` (VARCHAR(50)): Asset type (TABLE, DASHBOARD, PIPELINE, DATASET, etc.)
- `name` (VARCHAR(255)): Asset name
- `qualified_name` (VARCHAR(500), UNIQUE): Fully qualified name (e.g., "db.schema.table")
- `description` (TEXT): Asset description
- `metadata` (JSONB): Flexible metadata payload
- `version` (INT): Version number for optimistic locking
- `created_by` (UUID, FK): Reference to User
- `created_at` (TIMESTAMP): Creation timestamp
- `updated_at` (TIMESTAMP): Last update timestamp
- `deleted_at` (TIMESTAMP): Soft delete timestamp

**Relationships**:
- Many-to-One with Tenant
- Many-to-One with User (owner)
- Many-to-Many with Tag
- One-to-Many with Relationship (as source)
- One-to-Many with Relationship (as target)
- One-to-Many with Schema (if applicable)

**Indexes**:
- `idx_asset_tenant_type`: (tenant_id, type)
- `idx_asset_qualified_name`: (qualified_name)
- `idx_asset_created_at`: (created_at DESC)

### 4. Relationship
Represents relationships between assets (lineage, containment, etc.).

**Attributes**:
- `id` (UUID, PK): Unique identifier
- `tenant_id` (UUID, FK): Reference to Tenant
- `source_asset_id` (UUID, FK): Source asset
- `target_asset_id` (UUID, FK): Target asset
- `type` (VARCHAR(50)): Relationship type (DEPENDS_ON, CONTAINS, PRODUCES, USES, etc.)
- `metadata` (JSONB): Relationship-specific metadata
- `created_at` (TIMESTAMP): Creation timestamp
- `created_by` (UUID, FK): Reference to User

**Relationships**:
- Many-to-One with Tenant
- Many-to-One with Asset (source)
- Many-to-One with Asset (target)
- Many-to-One with User

**Indexes**:
- `idx_relationship_source`: (source_asset_id, type)
- `idx_relationship_target`: (target_asset_id, type)
- `idx_relationship_tenant`: (tenant_id)

### 5. Tag
Classification and discovery mechanism.

**Attributes**:
- `id` (UUID, PK): Unique identifier
- `tenant_id` (UUID, FK): Reference to Tenant
- `name` (VARCHAR(255)): Tag name
- `category` (VARCHAR(100)): Tag category (e.g., "PII", "Sensitive", "Department")
- `description` (TEXT): Tag description
- `created_at` (TIMESTAMP): Creation timestamp

**Relationships**:
- Many-to-One with Tenant
- Many-to-Many with Asset

**Indexes**:
- `idx_tag_tenant_name`: (tenant_id, name)
- `idx_tag_category`: (category)

### 6. AssetTag
Junction table for Asset-Tag many-to-many relationship.

**Attributes**:
- `asset_id` (UUID, FK): Reference to Asset
- `tag_id` (UUID, FK): Reference to Tag
- `created_at` (TIMESTAMP): Creation timestamp
- `created_by` (UUID, FK): Reference to User

**Primary Key**: (asset_id, tag_id)

**Indexes**:
- `idx_asset_tag_asset`: (asset_id)
- `idx_asset_tag_tag`: (tag_id)

### 7. Schema
Represents schema/structure information for assets (columns, fields, etc.).

**Attributes**:
- `id` (UUID, PK): Unique identifier
- `asset_id` (UUID, FK): Reference to Asset
- `name` (VARCHAR(255)): Schema element name (e.g., column name)
- `type` (VARCHAR(100)): Data type
- `nullable` (BOOLEAN): Whether nullable
- `metadata` (JSONB): Additional schema metadata
- `order` (INT): Display order
- `created_at` (TIMESTAMP): Creation timestamp

**Relationships**:
- Many-to-One with Asset

**Indexes**:
- `idx_schema_asset`: (asset_id, order)

### 8. Audit
Immutable audit log for compliance and tracking.

**Attributes**:
- `id` (UUID, PK): Unique identifier
- `tenant_id` (UUID, FK): Reference to Tenant
- `entity_type` (VARCHAR(50)): Type of entity (ASSET, RELATIONSHIP, TAG, etc.)
- `entity_id` (UUID): ID of the entity
- `action` (VARCHAR(50)): Action type (CREATE, UPDATE, DELETE, READ)
- `user_id` (UUID, FK): Reference to User (actor)
- `before_state` (JSONB): State before action (for UPDATE/DELETE)
- `after_state` (JSONB): State after action (for CREATE/UPDATE)
- `metadata` (JSONB): Additional audit metadata
- `ip_address` (INET): Client IP address
- `user_agent` (VARCHAR(500)): User agent string
- `timestamp` (TIMESTAMP): Action timestamp

**Relationships**:
- Many-to-One with Tenant
- Many-to-One with User

**Indexes**:
- `idx_audit_tenant_timestamp`: (tenant_id, timestamp DESC)
- `idx_audit_entity`: (entity_type, entity_id)
- `idx_audit_user`: (user_id, timestamp DESC)

### 9. BackgroundTask
Tracks background/asynchronous tasks.

**Attributes**:
- `id` (UUID, PK): Unique identifier
- `tenant_id` (UUID, FK): Reference to Tenant
- `type` (VARCHAR(50)): Task type (BULK_INGESTION, METADATA_SYNC, INDEX_REBUILD, etc.)
- `status` (ENUM): PENDING, RUNNING, COMPLETED, FAILED, CANCELLED
- `priority` (INT): Task priority (higher = more important)
- `payload` (JSONB): Task input parameters
- `result` (JSONB): Task output/result
- `error_message` (TEXT): Error message if failed
- `progress` (INT): Progress percentage (0-100)
- `retry_count` (INT): Number of retries attempted
- `max_retries` (INT): Maximum retries allowed
- `created_at` (TIMESTAMP): Creation timestamp
- `started_at` (TIMESTAMP): When task started
- `completed_at` (TIMESTAMP): When task completed

**Relationships**:
- Many-to-One with Tenant

**Indexes**:
- `idx_task_tenant_status`: (tenant_id, status)
- `idx_task_type_status`: (type, status)
- `idx_task_created_at`: (created_at DESC)

## Relationship Types

### Asset Relationships
- **DEPENDS_ON**: Asset A depends on Asset B (data dependency)
- **CONTAINS**: Asset A contains Asset B (schema → table, dashboard → chart)
- **PRODUCES**: Asset A produces Asset B (pipeline → table)
- **USES**: Asset A uses Asset B (dashboard → table)
- **RELATED_TO**: General relationship

### User Relationships
- **OWNS**: User owns an asset
- **FOLLOWS**: User follows an asset (notifications)

## Data Persistence Strategy

### PostgreSQL (Primary Store)
- All entities stored in PostgreSQL
- ACID guarantees for consistency
- Foreign key constraints for referential integrity
- JSONB for flexible metadata
- Row-level security for tenant isolation

### Aerospike (Cache)
- Cache key format: `{tenant_id}:{entity_type}:{entity_id}`
- TTL: 1 hour default, configurable
- Cache-aside pattern
- Invalidation on updates

### Elasticsearch (Search Index)
- Indexed fields: name, description, qualified_name, tags
- Full-text search with analyzers
- Faceted search on type, tags, categories
- Real-time indexing via change events

## Consistency Model

1. **Write Path**:
   - Write to PostgreSQL (transactional)
   - Invalidate cache entry
   - Publish event for search index update

2. **Read Path**:
   - Check cache (Aerospike)
   - If miss, read from PostgreSQL
   - Populate cache
   - Return result

3. **Eventual Consistency**:
   - Cache and search index updated asynchronously
   - Acceptable for metadata (not real-time critical)
   - Version numbers for conflict detection

## Multi-Tenancy

- **Row-Level Security**: All queries filtered by tenant_id
- **Index Strategy**: Include tenant_id in composite indexes
- **Cache Isolation**: Tenant ID in cache keys
- **Resource Quotas**: Per-tenant limits on assets, relationships, etc.

## Example Queries

### Get Asset with Lineage
```sql
-- Get asset
SELECT * FROM assets WHERE id = ? AND tenant_id = ?;

-- Get outgoing relationships
SELECT r.*, a.name as target_name 
FROM relationships r
JOIN assets a ON r.target_asset_id = a.id
WHERE r.source_asset_id = ? AND r.tenant_id = ?;

-- Get incoming relationships
SELECT r.*, a.name as source_name 
FROM relationships r
JOIN assets a ON r.source_asset_id = a.id
WHERE r.target_asset_id = ? AND r.tenant_id = ?;
```

### Search Assets
```sql
SELECT a.*, 
       array_agg(t.name) as tags
FROM assets a
LEFT JOIN asset_tags at ON a.id = at.asset_id
LEFT JOIN tags t ON at.tag_id = t.id
WHERE a.tenant_id = ?
  AND (a.name ILIKE ? OR a.description ILIKE ?)
  AND a.deleted_at IS NULL
GROUP BY a.id
ORDER BY a.updated_at DESC
LIMIT ? OFFSET ?;
```

### Get Audit Trail
```sql
SELECT * FROM audits
WHERE tenant_id = ?
  AND entity_type = ?
  AND entity_id = ?
ORDER BY timestamp DESC
LIMIT ?;
```

