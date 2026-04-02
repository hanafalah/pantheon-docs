# Pantheon Infrastructure Documentation

## Overview
Dokumentasi infrastruktur lengkap untuk aplikasi Pantheon yang mencakup backend, frontend, database, message queue, dan CI/CD pipeline.

## Technology Stack

### Backend
- **Language:** Rust Programming Language
- **Framework:** TBD (harus support multi-tenant, customizable, multi-database)
- **API Documentation:** Swagger/OpenAPI
- **Architecture:** Monolith dengan multi-repository
- **Authentication:** JWT dengan HS256
- **Secret Key:** `YXYlGIbJ65VGjQnETWX23iCvssXg7PJu`

### Frontend
- **Framework:** Nuxt 4
- **Architecture:** BFF (Backend for Frontend)
- **CSS Framework:** PrimeVue (primevue.org)
- **Icons:** Iconify
- **Platform Support:**
  - Web (PWA)
  - Desktop (Electron)

### Database
- **DBMS:** PostgreSQL
- **Architecture:** Multi-database, Multi-schema
- **Databases:**
  - `pantheon` (Core database)
  - `pantheon_tenant_*` (Tenant databases, where * = tenant ID)
- **Schema Strategy:**
  - Multi-schema per database
  - Cluster schemas by year: `cashier_2026`, `scm_2026`
  - Cluster schemas by year-month: `cashier_2026_01`, `cashier_2026_02`
- **Cluster Database Auto-generation:** 5 days before period change via RabbitMQ

### Message Queue & Background Jobs
- **Primary:** RabbitMQ
- **Alternative:** MQTT (if RabbitMQ not suitable)
- **Use Cases:**
  - Cluster database auto-generation
  - Background processing
  - Asynchronous tasks

### Search Engine
- **Service:** Elasticsearch
- **Use Cases:**
  - Full-text search
  - Analytics
  - Log aggregation

### Containerization
- **Platform:** Docker
- **Container Strategy:**
  - Backend services
  - Frontend services
  - Database instances
  - Message queue services
  - Elasticsearch

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Web PWA    │  │   Electron   │  │    Mobile    │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
└─────────┼──────────────────┼──────────────────┼─────────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
┌─────────────────────────────▼───────────────────────────────────┐
│                      FRONTEND LAYER                             │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │              Nuxt 4 BFF Server                         │    │
│  │  - SSR/SSG                                             │    │
│  │  - API Routes (BFF)                                    │    │
│  │  - Authentication Middleware                            │    │
│  │  - PrimeVue Components                                 │    │
│  └─────────────────────────┬──────────────────────────────┘    │
└────────────────────────────┼───────────────────────────────────┘
                             │ REST API
┌─────────────────────────────▼───────────────────────────────────┐
│                      BACKEND LAYER (RUST)                       │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    API Gateway                           │  │
│  │  - JWT Authentication (HS256)                           │  │
│  │  - Request Validation                                    │  │
│  │  - Rate Limiting                                         │  │
│  └─────────────┬────────────────────────────────────────────┘  │
│                │                                                │
│  ┌─────────────▼────────────────────────────────────────────┐  │
│  │              Controllers/Handlers                        │  │
│  │  - REST API Endpoints                                    │  │
│  │  - Tenant Resolution                                     │  │
│  │  - Override Chain: Tenant → Group → Project → Default   │  │
│  └─────────────┬────────────────────────────────────────────┘  │
│                │                                                │
│  ┌─────────────▼────────────────────────────────────────────┐  │
│  │              Services (DI via Interfaces)                │  │
│  │  - Business Logic                                        │  │
│  │  - Database Manager Integration                          │  │
│  │  - Entity Operations                                     │  │
│  └─────────────┬────────────────────────────────────────────┘  │
│                │                                                │
│  ┌─────────────▼────────────────────────────────────────────┐  │
│  │              Database Manager                            │  │
│  │  - Connection Pool Management                            │  │
│  │  - Multi-tenant Resolution                               │  │
│  │  - Entity-to-Connection Mapping                          │  │
│  │  - Migration Manager                                     │  │
│  └─────────────┬────────────────────────────────────────────┘  │
└────────────────┼─────────────────────────────────────────────────┘
                 │
┌────────────────▼─────────────────────────────────────────────────┐
│                      DATABASE LAYER                              │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              PostgreSQL - pantheon (Core)               │    │
│  │  Schemas:                                               │    │
│  │  - public (core tables)                                 │    │
│  │  - hq_{tenant_id} (HQ level)                            │    │
│  │  - pantheon_group_{tenant_id} (Group level)             │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │         PostgreSQL - pantheon_tenant_{id}               │    │
│  │  Schemas:                                               │    │
│  │  - public (tenant default)                              │    │
│  │  - cashier_2026, cashier_2027 (Cluster by year)        │    │
│  │  - cashier_2026_01, cashier_2026_02 (Cluster by month) │    │
│  │  - scm_2026, scm_2027 (SCM cluster)                    │    │
│  └─────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────┐
│                    SUPPORTING SERVICES                            │
│                                                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │  RabbitMQ    │  │ Elasticsearch│  │    MQTT      │           │
│  │  - Background│  │  - Search    │  │  (Optional)  │           │
│  │    Jobs      │  │  - Analytics │  │              │           │
│  │  - Cluster   │  │  - Logs      │  │              │           │
│  │    DB Gen    │  │              │  │              │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└───────────────────────────────────────────────────────────────────┘
```

## Project Structure

### Rust Backend Structure
```
pantheon-rust/
├── Cargo.toml                    # Root workspace
├── .env
├── docker/
│   ├── Dockerfile.backend
│   └── docker-compose.yml
├── config/                       # Configuration files
│   ├── app.toml
│   ├── database.toml
│   └── swagger.toml
├── rust-support/                 # Base classes & utilities
│   ├── src/
│   │   ├── base_entity.rs
│   │   ├── base_resource.rs
│   │   ├── base_controller.rs
│   │   ├── base_service.rs
│   │   ├── database_manager.rs
│   │   └── connection_manager.rs
│   └── Cargo.toml
├── projects/                     # Project level
│   ├── pantheon-business/
│   │   ├── src/
│   │   ├── migrations/
│   │   └── Cargo.toml
│   └── pantheon-hq/
│       ├── src/
│       ├── migrations/
│       └── Cargo.toml
├── groups/                       # Group level customization
│   └── pantheon-group/
│       ├── src/
│       └── Cargo.toml
├── tenants/                      # Tenant level customization
│   └── example-tenant/
│       ├── src/
│       └── Cargo.toml
└── repositories/                 # Shared modules/submodules
    ├── auth-module/
    ├── product-module/
    ├── finance-module/
    ├── franchise-module/
    ├── cashier-module/
    └── hr-module/
```

### Nuxt Frontend Structure
```
pantheon-nuxt/
├── package.json
├── nuxt.config.ts
├── .env
├── docker/
│   ├── Dockerfile.frontend
│   └── docker-compose.yml
├── assets/                       # CSS, images
│   └── styles/
│       └── primevue/
├── components/                   # Vue components
│   ├── auth/
│   ├── dashboard/
│   ├── products/
│   └── shared/
├── composables/                  # Composable functions
│   ├── useAuth.ts
│   └── useApi.ts
├── layouts/                      # App layouts
│   ├── default.vue
│   └── auth.vue
├── middleware/                   # Route middleware
│   └── auth.ts
├── pages/                        # Application pages
│   ├── index.vue
│   ├── login.vue
│   └── dashboard/
├── plugins/                      # Nuxt plugins
│   ├── primevue.ts
│   └── iconify.ts
├── server/                       # BFF Server
│   ├── api/                      # API routes
│   │   ├── auth/
│   │   └── proxy/
│   ├── middleware/
│   └── utils/
├── stores/                       # Pinia stores
│   ├── auth.ts
│   └── user.ts
└── public/                       # Static files
    └── icons/
```

## Database Connection Strategy

### Connection Levels
1. **Core Level:** `pantheon.public.*`
2. **HQ Level:** `pantheon.hq_{tenant_id}.*`
3. **Group Level:** `pantheon.pantheon_group_{tenant_id}.*`
4. **Tenant Level:** `pantheon_tenant_{id}.public.*`
5. **Cluster Level:** `pantheon_tenant_{id}.{cluster_name}.*`

### Entity Mapping
Entities are mapped to connections via configuration:
```rust
// Example entity connection mapping
EntityConnectionMap {
    "User": Connection::Core,
    "Tenant": Connection::Core,
    "Group": Connection::Core,
    "HQMembership": Connection::HQ(tenant_id),
    "GroupSettings": Connection::Group(tenant_id),
    "Product": Connection::Tenant(tenant_id),  // default
    "CashierTransaction": Connection::Cluster("cashier_2026", tenant_id),
    "SCMInventory": Connection::Cluster("scm_2026", tenant_id),
}
```

### Dynamic Connection Resolution
Connection is resolved at runtime based on:
1. Authenticated user's tenant
2. Entity configuration
3. Time-based cluster selection (for clustered entities)

## Configuration System

### Configuration Hierarchy

Pantheon menggunakan sistem configuration yang dapat di-override berdasarkan level aplikasi:

```
repositories (default) → projects → groups → tenants (most specific)
```

**Override Priority:**
- **Repositories** - Default configuration
- **Projects** - Override repositories config
- **Groups** - Override projects config (jika ada customization)
- **Tenants** - Override groups config (jika ada customization) - **FINAL CONFIG**

### Configuration Structure

#### Folder Structure
```
pantheon-rust/
├── repositories/
│   └── auth-module/
│       └── config/
│           ├── entity_mapping.toml
│           ├── routes.toml
│           └── middleware.toml
├── projects/
│   └── pantheon-business/
│       └── config/
│           ├── entity_mapping.toml      # Override repositories config
│           ├── routes.toml
│           ├── cluster.toml
│           └── business_logic.toml
├── groups/
│   └── pantheon-group/                  # Only exists if customization needed
│       └── config/
│           ├── entity_mapping.toml      # Override projects config
│           └── custom_settings.toml
└── tenants/
    └── example-tenant/                  # Only exists if customization needed
        └── config/
            ├── entity_mapping.toml      # Override groups config - FINAL
            ├── cluster.toml             # Custom cluster settings
            └── tenant_specific.toml
```

### Configuration Loading Sequence

```
┌─────────────────────────────────────────────────────────────┐
│                   REQUEST LIFECYCLE                         │
└─────────────────────────────────────────────────────────────┘

1. Request Received
   ↓
2. JWT Authentication
   ↓ (extract tenant_id, group_id, project_id)
3. ⚡ CONFIGURATION RESOLUTION ⚡  ← RUNS HERE (Early Stage)
   │
   ├─ Step 3.1: Load Default Config (repositories)
   │   └─ Load: entity_mapping, routes, middleware
   │
   ├─ Step 3.2: Override with Project Config (if exists)
   │   └─ Merge: projects/{project_id}/config/*
   │
   ├─ Step 3.3: Override with Group Config (if exists)
   │   └─ Merge: groups/{group_id}/config/*
   │
   └─ Step 3.4: Override with Tenant Config (if exists)
       └─ Merge: tenants/{tenant_id}/config/*

   Result: FINAL_CONFIG (ready for use)
   ↓
4. Database Connection Resolution (using FINAL_CONFIG)
   ↓
5. REST API Route Resolution (using FINAL_CONFIG)
   ↓
6. Controller/Handler Execution
   ↓
7. Service/Business Logic Execution
   ↓
8. Response
```

### Key Principles

#### 1. Early Resolution
- Configuration **HARUS** di-resolve setelah authentication berhasil
- Configuration di-resolve **SEBELUM** REST API route resolution
- Semua komponen aplikasi menggunakan FINAL_CONFIG yang sudah di-merge

#### 2. Lazy Loading for Custom Configs
- Folder `groups/{group_id}` **hanya ada** jika ada customization
- Folder `tenants/{tenant_id}` **hanya ada** jika ada customization
- Jika tidak ada custom config, sistem menggunakan config dari level parent
- Default fallback: repositories → projects

#### 3. Merge Strategy
```rust
// Pseudocode
let mut final_config = load_config("repositories/auth-module/config");

if project_config_exists(project_id) {
    final_config.merge(load_config("projects/{project_id}/config"));
}

if group_config_exists(group_id) {
    final_config.merge(load_config("groups/{group_id}/config"));
}

if tenant_config_exists(tenant_id) {
    final_config.merge(load_config("tenants/{tenant_id}/config"));
}

return final_config; // This is used throughout the request
```

### Configuration Types

#### 1. Entity-Connection Mapping
**File:** `entity_mapping.toml`

```toml
# repositories/auth-module/config/entity_mapping.toml (DEFAULT)
[connections.core]
entities = ["User", "Tenant", "Role", "Permission"]

[connections.tenant]
default = true

# projects/pantheon-business/config/entity_mapping.toml (OVERRIDE)
[connections.core]
entities = ["User", "Tenant", "Role", "Permission", "Project"]  # Added Project

[connections.cluster.cashier]
pattern = "cashier_{year}"
entities = ["CashierTransaction", "Receipt"]

# tenants/indomaret-001/config/entity_mapping.toml (FINAL OVERRIDE)
[connections.cluster.cashier]
pattern = "cashier_{year}_{month}"  # Custom: monthly instead of yearly
entities = ["CashierTransaction", "Receipt", "IndomaretSpecialLog"]
```

#### 2. Cluster Database Settings
**File:** `cluster.toml`

```toml
# projects/pantheon-business/config/cluster.toml (DEFAULT)
[cluster.cashier]
enabled = true
pattern = "cashier_{year}"
auto_generate = true
days_before = 5

[cluster.scm]
enabled = true
pattern = "scm_{year}"
auto_generate = true
days_before = 5

# tenants/alfamart-001/config/cluster.toml (OVERRIDE)
[cluster.cashier]
enabled = true
pattern = "cashier_{year}_{month}"  # Monthly partition
auto_generate = true
days_before = 3  # Generate 3 days before instead of 5

[cluster.scm]
enabled = false  # Alfamart doesn't use SCM module
```

#### 3. API Route Customization
**File:** `routes.toml`

```toml
# projects/pantheon-business/config/routes.toml (DEFAULT)
[routes.products]
path = "/api/products"
controller = "ProductController"
middleware = ["auth", "tenant_check"]

# tenants/special-client/config/routes.toml (OVERRIDE)
[routes.products]
path = "/api/products"
controller = "CustomProductController"  # Custom controller for this tenant
middleware = ["auth", "tenant_check", "special_logging"]  # Extra middleware
```

#### 4. Middleware Configuration
**File:** `middleware.toml`

```toml
# repositories/auth-module/config/middleware.toml (DEFAULT)
[middleware.auth]
enabled = true
jwt_secret = "${JWT_SECRET}"
token_expiry = 3600

# groups/premium-group/config/middleware.toml (OVERRIDE)
[middleware.auth]
enabled = true
jwt_secret = "${JWT_SECRET}"
token_expiry = 7200  # Premium group gets longer session
```

#### 5. Resource Mapping
**File:** `resource_mapping.toml`

```toml
# repositories/auth-module/config/resource_mapping.toml (DEFAULT)
[resources.User]
view = "UserViewResource"
show = "UserShowResource"

# tenants/api-heavy-client/config/resource_mapping.toml (OVERRIDE)
[resources.User]
view = "LightweightUserViewResource"  # Lighter payload for API-heavy client
show = "UserShowResource"
```

#### 6. Business Logic Settings
**File:** `business_logic.toml`

```toml
# projects/pantheon-business/config/business_logic.toml (DEFAULT)
[pricing]
tax_rate = 0.10
discount_enabled = true
max_discount = 0.50

# tenants/special-franchise/config/business_logic.toml (OVERRIDE)
[pricing]
tax_rate = 0.15  # Different tax rate
discount_enabled = true
max_discount = 0.70  # Can give bigger discounts
```

### Configuration Manager Implementation

```rust
// rust-support/src/config/manager.rs
pub struct ConfigManager {
    base_config: Config,      // from repositories
    project_config: Option<Config>,
    group_config: Option<Config>,
    tenant_config: Option<Config>,
    final_config: Config,     // merged result
}

impl ConfigManager {
    /// Initialize and resolve configuration after authentication
    pub fn new(project_id: &str, group_id: Option<&str>, tenant_id: Option<&str>) -> Self {
        let mut manager = Self {
            base_config: Self::load_base_config(),
            project_config: Self::load_project_config(project_id),
            group_config: group_id.and_then(|id| Self::load_group_config(id)),
            tenant_config: tenant_id.and_then(|id| Self::load_tenant_config(id)),
            final_config: Config::default(),
        };

        manager.resolve();
        manager
    }

    /// Merge all configs following the hierarchy
    fn resolve(&mut self) {
        self.final_config = self.base_config.clone();

        if let Some(ref project_cfg) = self.project_config {
            self.final_config.merge(project_cfg);
        }

        if let Some(ref group_cfg) = self.group_config {
            self.final_config.merge(group_cfg);
        }

        if let Some(ref tenant_cfg) = self.tenant_config {
            self.final_config.merge(tenant_cfg);
        }
    }

    /// Get the final resolved configuration
    pub fn get(&self) -> &Config {
        &self.final_config
    }
}
```

### Configuration in Request Context

```rust
// Example middleware that resolves config
pub async fn config_resolver_middleware(
    req: HttpRequest,
    payload: web::Payload,
) -> Result<HttpResponse> {
    // JWT already validated at this point
    let claims = req.extensions().get::<JwtClaims>().unwrap();

    // ⚡ RESOLVE CONFIGURATION HERE ⚡
    let config_manager = ConfigManager::new(
        &claims.project_id,
        claims.group_id.as_deref(),
        claims.tenant_id.as_deref(),
    );

    // Store in request extensions for later use
    req.extensions_mut().insert(config_manager);

    // Continue to next middleware/handler
    Ok(HttpResponse::Ok().finish())
}

// Usage in controller
pub async fn get_products(
    req: HttpRequest,
) -> Result<HttpResponse> {
    // Get the resolved config
    let config = req.extensions()
        .get::<ConfigManager>()
        .unwrap()
        .get();

    // Use config for database connection, routing, etc.
    let entity_mapping = config.get_entity_mapping();
    let connection = database_manager.get_connection_for_entity("Product", entity_mapping);

    // ... rest of logic
}
```

### Benefits of This Approach

1. **Flexibility** - Each tenant/group dapat customize tanpa affect others
2. **Performance** - Config di-resolve sekali di awal request lifecycle
3. **Maintainability** - Default config di repositories, custom di tenant
4. **Scalability** - Hanya load config yang benar-benar ada
5. **Clear Override Chain** - repositories → projects → groups → tenants
6. **Type Safety** - Compile-time validation dengan Rust type system

## Authentication Flow

```
┌──────────┐                                    ┌──────────┐
│  Client  │                                    │   Rust   │
│  (Nuxt)  │                                    │  Backend │
└────┬─────┘                                    └────┬─────┘
     │                                                │
     │  1. POST /api/auth/login                      │
     │    { username, password }                     │
     ├──────────────────────────────────────────────>│
     │                                                │
     │                                          2. Validate
     │                                          credentials
     │                                                │
     │                                          3. Generate
     │                                          JWT + Refresh
     │                                          Token (HS256)
     │                                                │
     │  4. Return tokens + user info                 │
     │<──────────────────────────────────────────────┤
     │   { access_token, refresh_token, user }       │
     │                                                │
     │  5. Store tokens in cookie/localStorage       │
     │                                                │
     │  6. Subsequent API calls                      │
     │     Authorization: Bearer {access_token}      │
     ├──────────────────────────────────────────────>│
     │                                                │
     │                                          7. Verify JWT
     │                                          8. Resolve tenant
     │                                          9. Map DB connections
     │                                         10. Execute business
     │                                             logic
     │                                                │
     │  11. Return response                          │
     │<──────────────────────────────────────────────┤
     │                                                │
```

## CI/CD Pipeline (GitHub Actions)

### Deployment Environments
1. **Production** - main branch
2. **Staging** - staging branch
3. **Development** - develop branch

### GitHub Actions Workflow
```yaml
# .github/workflows/development.yml
name: Development Pipeline
on:
  push:
    branches: [develop]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - Checkout code
      - Setup Rust
      - Run cargo test (unit tests)
      - Run integration tests

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - Build Docker images
      - Push to registry

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - Deploy to development environment
```

### GitHub Account
- **Email:** hamzahnafalah@gmail.com
- **Repositories:** TBD (akan dibuat)

## Deployment Strategy

### Docker Compose Services
```yaml
version: '3.8'
services:
  postgres-core:
    image: postgres:15
    environment:
      POSTGRES_DB: pantheon

  postgres-tenant-1:
    image: postgres:15
    environment:
      POSTGRES_DB: pantheon_tenant_1

  rabbitmq:
    image: rabbitmq:3-management

  elasticsearch:
    image: elasticsearch:8.x

  rust-backend:
    build: ./pantheon-rust
    depends_on:
      - postgres-core
      - rabbitmq

  nuxt-frontend:
    build: ./pantheon-nuxt
    depends_on:
      - rust-backend

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
```

## Monitoring & Logging

### Tools
- **Application Logs:** Elasticsearch + Kibana
- **Performance Monitoring:** TBD
- **Error Tracking:** TBD

## Security Considerations

1. **JWT Secret Management:** Use environment variables
2. **Database Credentials:** Encrypted and stored in vault
3. **API Rate Limiting:** Implement in API Gateway
4. **CORS Policy:** Configure properly for frontend domains
5. **HTTPS:** Enforce in production
6. **SQL Injection Prevention:** Use parameterized queries
7. **XSS Prevention:** Sanitize inputs in frontend

## Scalability Considerations

1. **Horizontal Scaling:** Docker containers can be scaled
2. **Database Sharding:** Multi-tenant architecture supports it
3. **Caching Layer:** Redis (TBD if needed)
4. **CDN:** For static assets
5. **Load Balancer:** Nginx or cloud-based

## Backup Strategy

1. **Database Backups:**
   - Daily automated backups
   - Point-in-time recovery
   - Backup retention: 30 days

2. **Application Backups:**
   - Configuration files
   - Environment variables
   - Docker images

## Development Environment Setup

### Prerequisites
- Docker & Docker Compose
- Rust (latest stable)
- Node.js 18+ & npm/yarn
- PostgreSQL 15+
- Git

### Quick Start
```bash
# Clone repositories
git clone [pantheon-rust-repo]
git clone [pantheon-nuxt-repo]

# Setup backend
cd pantheon-rust
cp .env.example .env
cargo build
cargo run

# Setup frontend
cd pantheon-nuxt
cp .env.example .env
npm install
npm run dev
```

## Future Enhancements

1. **Microservices Migration:** Gradual transition from monolith
2. **GraphQL API:** Additional API layer
3. **Mobile Apps:** Native iOS/Android
4. **AI/ML Integration:** Predictive analytics
5. **Multi-region Deployment:** Global presence

## Conclusion

Infrastruktur Pantheon dirancang untuk mendukung:
- Multi-tenancy yang scalable
- Customization per tenant/group/project
- High availability dan fault tolerance
- Maintainability dan extensibility
- Security dan compliance

Dokumentasi ini akan terus diupdate seiring perkembangan aplikasi.
