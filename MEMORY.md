# Pantheon Memory - Business & Technical Context

## Business Overview

### Company Name
**Pantheon Nusantara Kreasi**

### Application Name
**Pantheon App**

### Business Mission
Pantheon merupakan sebuah wadah/platform yang menampung banyak bisnis yang akan dijalankan. Aplikasi ini dirancang sebagai solusi multi-bisnis yang terintegrasi dengan kemampuan multi-tenant.

### Core Business Values
1. **Scalability** - Dapat menampung banyak bisnis/tenant
2. **Customizability** - Setiap tenant dapat memiliki kustomisasi sendiri
3. **Integration** - Satu platform untuk berbagai kebutuhan bisnis
4. **Efficiency** - Mengelola projects, products, keuangan, dan tenant secara terpusat

## Business Modules

### 1. Pantheon Business (Primary Project)
Project utama yang mengelola operasional bisnis dengan modul:

#### a. Product Management
- Manajemen katalog produk
- SKU dan variants
- Pricing dan inventory
- Product categorization

#### b. Financial Management
- Accounting dan bookkeeping
- Revenue tracking
- Expense management
- Financial reporting
- Multi-currency support

#### c. Franchise Management
- Franchise registration
- Franchise monitoring
- Performance tracking
- Territory management

#### d. Cashier/POS Management
- Point of Sale operations
- Transaction recording
- Payment processing
- Receipt generation
- Daily reporting

#### e. HR/Employee Management (Kepegawaian)
- Employee records
- Attendance tracking
- Payroll management
- Performance evaluation
- Leave management

### 2. Pantheon HQ (Headquarters/SAAS Platform)
Project untuk mengelola tingkat HQ dengan fitur:

#### a. Project Installation
- Setup new business projects
- Configuration management
- Initial data seeding

#### b. Membership Management
- Client membership (PT level seperti Indomaret, Alfamart)
- Franchise level membership
- Subscription management

#### c. SAAS Services
- Automated group provisioning
- Tenant provisioning
- Billing and invoicing
- License management

#### d. Multi-level Organization
- **PT Level:** Corporate clients (contoh: Indomaret Group, Alfamart Group)
- **Group Level:** Business groups under corporate
- **Franchise Level:** Individual franchise locations

## Technical Architecture

### Technology Stack

#### Backend
- **Language:** Rust
- **Architecture:** Monolith with Multi-Repository
- **Pattern:** Repository Pattern, Dependency Injection
- **API:** RESTful API with Swagger Documentation
- **Auth:** JWT with HS256 algorithm

#### Frontend
- **Framework:** Nuxt 4
- **Architecture:** BFF (Backend for Frontend)
- **UI Library:** PrimeVue
- **Icons:** Iconify
- **Platform:** Web (PWA), Desktop (Electron)

#### Database
- **DBMS:** PostgreSQL
- **Strategy:** Multi-database, Multi-schema, Cluster database
- **Databases:**
  - `pantheon` - Core database
  - `pantheon_tenant_{id}` - Tenant databases

#### Message Queue
- **Primary:** RabbitMQ
- **Alternative:** MQTT
- **Purpose:** Background jobs, cluster DB generation

#### Search
- **Engine:** Elasticsearch
- **Purpose:** Full-text search, analytics, logging

## Architecture Patterns

### 1. Multi-Tenant Architecture

#### Database Level Separation
```
pantheon (Core DB)
├── public schema (core tables)
├── hq_{tenant_id} schemas (HQ level)
└── pantheon_group_{tenant_id} schemas (Group level)

pantheon_tenant_{id} (Tenant DBs)
├── public schema (default tenant tables)
├── cashier_2026, cashier_2027 (Cluster by year)
└── scm_2026, scm_2027 (SCM clusters)
```

#### Connection Hierarchy
1. **Core Connection** - `pantheon.public.*`
2. **HQ Connection** - `pantheon.hq_{tenant_id}.*`
3. **Group Connection** - `pantheon.pantheon_group_{tenant_id}.*`
4. **Tenant Connection** - `pantheon_tenant_{id}.public.*`
5. **Cluster Connection** - `pantheon_tenant_{id}.{cluster_schema}.*`

### 2. Code Organization Hierarchy

```
Projects → Groups → Tenants → Repositories
```

#### Override Chain
Default logic dapat di-override berdasarkan hierarchy:
```
Default (Repositories) → Project → Group → Tenant
```

**Contoh:**
- API endpoint di repositories (default)
- Di-override oleh project jika ada business logic khusus project
- Di-override oleh group jika ada business logic khusus group
- Di-override oleh tenant jika ada business logic khusus tenant

### 3. Base Classes Pattern

Semua komponen memiliki base class di `rust-support`:

#### Base Entity
```rust
trait BaseEntity {
    fn get_view_resource() -> ResourceType;
    fn get_show_resource() -> ResourceType;
    fn to_view_api(&self) -> JsonValue;
    fn to_show_api(&self) -> JsonValue;
    fn get_connection() -> ConnectionType;
}
```

#### Base Resource
```rust
trait BaseResource {
    fn transform(&self) -> JsonValue;
}

// ViewResource - untuk listing
// ShowResource - untuk detail (extends ViewResource)
```

#### Base Controller
```rust
trait BaseController {
    fn index() -> Response;
    fn show(id) -> Response;
    fn store(request) -> Response;
    fn update(id, request) -> Response;
    fn destroy(id) -> Response;
}
```

#### Base Service
```rust
trait BaseService {
    fn find_all() -> Vec<Entity>;
    fn find_by_id(id) -> Option<Entity>;
    fn create(data) -> Result<Entity>;
    fn update(id, data) -> Result<Entity>;
    fn delete(id) -> Result<bool>;
}
```

### 4. Database Manager

Komponen kunci yang mengelola:

#### Connection Pool Management
- Manage multiple database connections
- Pool sizing per connection type
- Connection health checking

#### Tenant Resolution
- Resolve tenant dari JWT token
- Map tenant ke database connection
- Auto-switch connection based on tenant

#### Entity-to-Connection Mapping
```rust
ConnectionConfig {
    "User": Core,
    "Tenant": Core,
    "Product": Tenant,
    "CashierTransaction": Cluster("cashier_2026"),
    // ... other mappings
}
```

#### Migration Manager
- Run migrations per database
- Track migration versions
- Support rollback

### 5. Cluster Database Strategy

#### Purpose
Segmentasi data transaksional untuk:
- Performa query yang lebih baik
- Easier archival
- Partitioned backups

#### Cluster Categories
1. **cashier_*** - POS transactions
2. **scm_*** - Supply chain management

#### Cluster Types
1. **By Year:** `cashier_2026`, `scm_2027`
2. **By Month:** `cashier_2026_01`, `cashier_2026_02`

#### Auto-generation
- Triggered 5 days before period change
- Automated via RabbitMQ background job
- Schema copied from template
- Automatic migration execution

## Authentication & Authorization

### JWT Implementation

#### Token Structure
```json
{
  "header": {
    "alg": "HS256",
    "typ": "JWT"
  },
  "payload": {
    "sub": "user_id",
    "tenant_id": "tenant_id",
    "group_id": "group_id",
    "project_id": "project_id",
    "roles": ["admin", "cashier"],
    "permissions": ["read:products", "write:transactions"],
    "iat": 1234567890,
    "exp": 1234571490
  },
  "signature": "..."
}
```

#### Secret Key
```
YXYlGIbJ65VGjQnETWX23iCvssXg7PJu
```

#### Token Types
1. **Access Token** - Short-lived (1 hour)
2. **Refresh Token** - Long-lived (7 days)

### Authentication Flow
1. User login dengan credentials
2. Backend verifikasi credentials
3. Generate access token + refresh token
4. Return tokens ke client
5. Client store tokens (localStorage/cookie)
6. Subsequent requests menggunakan access token
7. Jika expired, use refresh token untuk get new access token

### Tenant Resolution
Setelah JWT verified:
1. Extract tenant_id dari token payload
2. Database Manager resolve connection untuk tenant
3. Map entities ke appropriate database connections
4. Execute business logic dengan connection yang tepat

## API Design Principles

### RESTful Conventions
```
GET    /api/products        - List all products
GET    /api/products/:id    - Show product detail
POST   /api/products        - Create product
PUT    /api/products/:id    - Update product
DELETE /api/products/:id    - Delete product
```

### Response Format
```json
{
  "success": true,
  "data": {...},
  "message": "Operation successful",
  "meta": {
    "pagination": {...},
    "timestamp": "2026-04-02T12:00:00Z"
  }
}
```

### Error Format
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      {"field": "email", "message": "Invalid email format"}
    ]
  },
  "meta": {
    "timestamp": "2026-04-02T12:00:00Z"
  }
}
```

## Dependency Injection Pattern

### Controller → Service
```rust
struct ProductController {
    service: Box<dyn ProductService>
}

impl ProductController {
    fn new(service: Box<dyn ProductService>) -> Self {
        Self { service }
    }

    fn index(&self) -> Response {
        let products = self.service.find_all();
        Response::json(products)
    }
}
```

### Interface-based Design
- Controllers depend on service interfaces, not implementations
- Services can be swapped without changing controllers
- Easy testing dengan mock services

## Project Structure Example

### Pantheon Business Structure
```
projects/pantheon-business/
├── groups/
│   └── pantheon-group/
│       └── tenants/
│           └── example-tenant/
└── repositories/
    ├── product-module/
    ├── finance-module/
    ├── franchise-module/
    ├── cashier-module/
    └── hr-module/
```

### Example: Franchise Business Scenario

**Project:** pantheon-franchise
**Group:** indomaret-group
**Tenant:** indomaret-branch-001

**Alternative Group:** alfamart-group
**Tenant:** alfamart-branch-001

Kedua group berbeda tapi menggunakan project yang sama (pantheon-franchise).

## Configuration Management

### Configuration Hierarchy & Resolution

#### Override Chain
```
repositories (default) → projects → groups → tenants (most specific)
```

Configuration mengikuti hierarchy yang sama dengan code organization:
- **repositories** - Default configuration untuk semua
- **projects** - Override repositories config
- **groups** - Override projects config (hanya jika ada customization)
- **tenants** - Override groups config (hanya jika ada customization) - **FINAL**

#### Key Principles

1. **Early Resolution**
   - Configuration **HARUS** di-resolve di tahap awal request lifecycle
   - Sequence: Authentication → **Configuration Resolution** → Database Connection → REST API
   - Configuration di-resolve SETELAH authentication berhasil
   - Configuration di-resolve SEBELUM REST API route handler

2. **Lazy Loading**
   - Folder `groups/{group_id}/config` **hanya ada** jika ada customization
   - Folder `tenants/{tenant_id}/config` **hanya ada** jika ada customization
   - Jika tidak ada custom config, fallback ke parent level
   - Default fallback path: repositories → projects

3. **Merge Strategy**
   - Load base config dari repositories
   - Merge dengan project config (if exists)
   - Merge dengan group config (if exists)
   - Merge dengan tenant config (if exists)
   - Result: FINAL_CONFIG yang digunakan untuk request

#### Configuration Structure
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
│       └── config/              # Always exists
│           ├── entity_mapping.toml
│           ├── cluster.toml
│           └── business_logic.toml
├── groups/
│   └── {group_id}/              # Only if customization needed
│       └── config/
│           └── custom_settings.toml
└── tenants/
    └── {tenant_id}/             # Only if customization needed
        └── config/
            └── tenant_specific.toml
```

### Configuration Types

#### 1. Entity-Connection Mapping
Mapping entities ke database connections dengan override capability.

#### 2. Cluster Database Settings
Konfigurasi cluster database (pattern, auto-generation, timing).

#### 3. API Route Customization
Custom routes, controllers, dan middleware per level.

#### 4. Resource Mapping
ViewResource dan ShowResource customization per entity.

#### 5. Business Logic Settings
Settings seperti tax rates, discount rules, pricing logic.

#### 6. Middleware Configuration
JWT settings, rate limiting, custom middleware.

### Configuration Levels
1. **Application Config** - `config/app.toml`
2. **Database Config** - `config/database.toml`
3. **Project Config** - Per project
4. **Group Config** - Per group (overrides project)
5. **Tenant Config** - Per tenant (overrides group)

### Entity-Connection Configuration
```toml
[connections.core]
entities = ["User", "Tenant", "Group", "Role", "Permission"]

[connections.hq]
entities = ["HQMembership", "Subscription", "License"]

[connections.group]
entities = ["GroupSettings", "GroupConfig"]

[connections.tenant]
default = true  # Entities not listed elsewhere use tenant connection

[connections.clusters.cashier]
pattern = "cashier_{year}"
entities = ["CashierTransaction", "CashierReceipt"]

[connections.clusters.scm]
pattern = "scm_{year}"
entities = ["Inventory", "StockMovement"]
```

## Migration Strategy

### Migration Levels
1. **Core Migrations** - Run on `pantheon` database
2. **HQ Migrations** - Run on HQ schemas
3. **Group Migrations** - Run on group schemas
4. **Tenant Migrations** - Run on each tenant database
5. **Cluster Migrations** - Run when cluster created

### Versioning
Menggunakan sistem seperti Composer untuk versioning:
```toml
[package]
name = "pantheon-business"
version = "1.0.0"

[dependencies]
rust-support = "1.0.0"
product-module = "1.2.0"
finance-module = "1.1.0"
```

## Development Workflow

### Git Branching Strategy
```
main (production)
├── staging
│   └── develop
│       ├── feature/new-feature
│       ├── bugfix/fix-issue
│       └── hotfix/critical-fix
```

### Commit Convention
```
feat: Add new feature
fix: Fix bug
docs: Update documentation
refactor: Code refactoring
test: Add tests
chore: Maintenance tasks
```

### CI/CD Pipeline
1. **Development Branch**
   - Auto run unit tests
   - Build Docker images
   - Deploy to dev environment

2. **Staging Branch**
   - Run integration tests
   - Deploy to staging
   - Performance testing

3. **Main Branch**
   - Full test suite
   - Security scan
   - Deploy to production

## Key Design Decisions

### 1. Why Rust for Backend?
- Performance dan efficiency
- Memory safety
- Concurrency support
- Strong type system
- Growing ecosystem

### 2. Why Multi-Repository Monolith?
- Modularity dengan code sharing
- Easier deployment (vs microservices)
- Shared business logic
- Gradual migration path ke microservices

### 3. Why Multi-Database Multi-Schema?
- True tenant isolation
- Performance per tenant
- Easier backup/restore per tenant
- Custom schema per tenant
- Scalability

### 4. Why Cluster Database?
- Query performance pada large datasets
- Easier data archival
- Partitioned backups
- Historical data management

### 5. Why BFF Pattern?
- Optimize API for frontend needs
- Security layer
- API aggregation
- Platform-specific responses

### 6. Why Early Configuration Resolution?
- **Performance:** Config di-resolve sekali per request, tidak berulang
- **Consistency:** Semua komponen menggunakan config yang sama
- **Predictability:** Config sudah ready sebelum business logic execute
- **Flexibility:** Setiap tenant/group dapat customize tanpa affect others
- **Maintainability:** Default config di repositories, custom di tenant
- **Type Safety:** Compile-time validation dengan Rust

### 7. Why Lazy Config Loading (Groups/Tenants)?
- **Disk Space:** Hanya simpan config yang benar-benar custom
- **Clarity:** Mudah identify mana tenant/group yang punya customization
- **Performance:** Tidak perlu load config yang tidak ada
- **Simplicity:** Default behavior adalah inherit dari parent
- **Scalability:** Thousands of tenants tanpa thousands of config files

## Business Rules

### Multi-Tenant Rules
1. Tenant tidak bisa akses data tenant lain
2. Setiap tenant punya database sendiri
3. Core data (users, tenants, groups) di database utama
4. Cluster database auto-generate 5 hari sebelum periode

### Authentication Rules
1. JWT token required untuk semua API (except login)
2. Token expired perlu refresh
3. Tenant resolution dari token
4. Role-based access control

### Database Connection Rules
1. Entity config determine connection
2. Jika tidak ada config, default ke tenant connection
3. Cluster connection based on timestamp/period
4. Connection pool per tenant/database

## Performance Considerations

### Query Optimization
- Index pada foreign keys
- Index pada frequently queried columns
- Pagination untuk large datasets
- Cluster database untuk transactional data

### Caching Strategy
- Redis untuk session/cache (optional)
- Query result caching
- API response caching
- Static asset caching

### Connection Pooling
- Pool size per database type
- Reuse connections
- Health check connections
- Timeout settings

## Security Considerations

### Application Security
- Input validation di DTO
- SQL injection prevention (parameterized queries)
- XSS prevention (sanitize outputs)
- CSRF protection
- Rate limiting

### Database Security
- Separate credentials per database
- Encrypted connections (SSL)
- Least privilege access
- Regular security audits

### API Security
- JWT validation
- CORS configuration
- API rate limiting
- Request size limits

## Testing Strategy

### Unit Tests
- Test individual functions/methods
- Mock dependencies
- Run on every commit

### Integration Tests
- Test component interactions
- Test database operations
- Test API endpoints

### End-to-End Tests
- Test complete user flows
- Test multi-tenant scenarios
- Test cluster database generation

## Monitoring & Logging

### Application Logs
- Structured logging
- Log levels (trace, debug, info, warn, error)
- Elasticsearch integration
- Log rotation

### Performance Monitoring
- API response times
- Database query times
- Resource utilization
- Error rates

### Business Metrics
- Active tenants
- Transactions per tenant
- Revenue tracking
- User activity

## Future Roadmap

### Phase 1 - Foundation (Current)
- Setup infrastructure
- Implement authentication
- Multi-tenant database
- Basic CRUD operations

### Phase 2 - Core Modules
- Product management
- Financial management
- Cashier/POS
- Basic reporting

### Phase 3 - Advanced Features
- Franchise management
- HR/Employee management
- Advanced analytics
- Mobile apps

### Phase 4 - SAAS Platform
- Pantheon HQ
- Auto-provisioning
- Billing system
- Multi-level organization

### Phase 5 - Scale & Optimize
- Microservices migration
- Multi-region deployment
- AI/ML features
- Advanced integrations

## Conclusion

Pantheon adalah aplikasi yang dirancang untuk:
1. **Scalability** - Menampung banyak bisnis/tenant
2. **Flexibility** - Customizable per level (project/group/tenant)
3. **Performance** - Multi-database, cluster strategy
4. **Security** - Tenant isolation, JWT auth
5. **Maintainability** - Clean architecture, base classes, DI

Memory ini akan terus diupdate seiring perkembangan aplikasi dan business requirements.

---

**Last Updated:** 2026-04-02
**Version:** 1.0.0
**Maintainer:** Pantheon Development Team
