# Plan 1: Foundation & Infrastructure Setup

## Status: PENDING APPROVAL

## Repository Structure

**GitHub Organization:** `pantheon` (atau `hanafalah` jika organization tidak tersedia)

**Repositories:**
1. **pantheon/pantheon-app** - Backend Rust workspace (monolith dengan multi-repo structure internal)
2. **pantheon/pantheon-nuxt** - Frontend Nuxt 4 dengan BFF
3. **pantheon/pantheon-docs** - Documentation dan Swagger UI

**Local Working Directory:** `/var/www/projects/pantheon/`

**Note:** Folder `pantheon-rust` yang existing akan diganti dengan `pantheon-app` dari GitHub.

## Objektif
Membangun fondasi dan infrastruktur lengkap untuk aplikasi Pantheon dengan arsitektur multi-tenant yang kompleks, termasuk:
- Setup project structure dengan hierarchy repositories → projects → groups → tenants
- Database architecture multi-database dengan core, tenant, dan cluster databases
- Authentication system dengan JWT (HS256)
- Configuration system dengan override hierarchy
- Service Provider pattern untuk module initialization
- Database Manager dengan dynamic connection mapping
- Base classes (Entity, Resource, Controller, Service) dengan resource transformation
- Migration system dari Laravel wellmed project ke Rust repositories
- Complete infrastructure setup (Docker, CI/CD, RabbitMQ, Swagger)

## Scope

### In Scope
1. Setup Rust backend project structure dengan workspace hierarchy (repositories/projects/groups/tenants)
2. Setup Nuxt 4 frontend dengan BFF pattern
3. Setup pantheon-docs sebagai aplikasi terpisah untuk Swagger UI
4. Konfigurasi PostgreSQL multi-database (pantheon, pantheon_tenant_*, dengan multi-schema)
5. Implementasi JWT authentication (HS256) dengan secret: "YXYlGIbJ65VGjQnETWX23iCvssXg7PJu"
6. Database Manager dengan dynamic connection mapping dan tenant resolution
7. Configuration system dengan override hierarchy (repositories → projects → groups → tenants)
8. Service Provider pattern untuk module initialization
9. Base classes dengan traits: BaseEntity (getViewResource, getShowResource, toViewApi, toShowApi), BaseResource, BaseController, BaseService
10. Migration semua repositories dari wellmed/repositories ke Rust crates (kecuali yang excluded)
11. Create all core database tables (users, user_references, domains, tenants, workspaces, licenses, unicodes, api_accesses, jobs, districts, provinces, countries, subdistricts, villages, model_has_licenses, migrations)
12. Create all tenant database tables (addresses, card_identities, consuments, employees, items, variants, permissions, roles, notifications, stocks, transactions, dll - 27 tables)
13. Setup cluster database structure untuk cashier_* dan scm_*
14. Docker containerization untuk semua services
15. CI/CD pipeline dengan 3 environments (production, staging, development)
16. Unit test automation di GitHub Actions
17. Swagger/OpenAPI documentation dengan pantheon-docs app
18. RabbitMQ integration untuk background jobs
19. rust-support repository sebagai base library untuk semua modules

### Out of Scope
1. Business logic implementation untuk modules (Product Management, Finance, Cashier, HR, SCM) - akan di Plan 2
2. Cluster database auto-generation dengan RabbitMQ scheduler (5 hari sebelum tahun baru) - akan di Plan 3
3. Elasticsearch integration untuk search dan analytics - akan di Plan 4
4. PWA & Electron setup untuk frontend - akan di Plan 5
5. MQTT integration (optional if RabbitMQ insufficient) - akan di Plan 6
6. Production deployment dan server provisioning - akan di Plan 7
7. pantheon-hq project untuk SAAS management - akan di Plan 8
8. Advanced features seperti multi-franchise orchestration - akan di Plan 9+

## Technical Requirements

### Backend (Rust)

#### Framework Selection
**Recommended:** Actix-web atau Axum
- Support untuk async/await
- Middleware support
- Good performance
- Active community
- Dokumentasi lengkap

**ORM/Database:** Diesel atau SeaORM
- Multi-database support
- Migration support
- Type-safe queries
- Connection pooling

#### Project Structure
```
pantheon-app/                   # Main Rust workspace (GitHub: pantheon/pantheon-app)
├── Cargo.toml                  # Workspace manifest dengan dependencies ke semua crates
├── .env.example
├── .gitignore
├── README.md
├── docker/
│   ├── Dockerfile
│   ├── docker-compose.yml      # Multi-service: PostgreSQL, RabbitMQ, Rust BE
│   └── .dockerignore
├── config/                     # Core configuration
│   ├── app.toml
│   ├── database.toml           # Multi-database connections
│   ├── jwt.toml                # JWT secret: YXYlGIbJ65VGjQnETWX23iCvssXg7PJu
│   ├── entity_connections.toml # Entity-to-connection mapping
│   └── cluster_config.toml     # Cluster database settings
├── migrations/
│   ├── core/                   # Migrations untuk database pantheon
│   │   ├── 001_create_users_tables.sql
│   │   ├── 002_create_tenants_tables.sql
│   │   ├── 003_create_regional_tables.sql
│   │   └── ...
│   └── tenant/                 # Template migrations untuk pantheon_tenant_*
│       ├── 001_create_base_tables.sql
│       ├── 002_create_permission_tables.sql
│       └── ...
├── rust-support/               # Base library untuk semua modules (CORE DEPENDENCY)
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs
│       ├── provider/           # Service Provider pattern
│       │   ├── mod.rs
│       │   └── service_provider.rs
│       ├── base/               # Base traits dan implementations
│       │   ├── entity.rs       # BaseEntity trait dengan getViewResource, getShowResource, toViewApi, toShowApi
│       │   ├── resource.rs     # BaseResource trait, ViewResource, ShowResource
│       │   ├── controller.rs   # BaseController trait
│       │   └── service.rs      # BaseService trait
│       ├── database/
│       │   ├── manager.rs      # DatabaseManager untuk multi-database
│       │   ├── connection.rs   # ConnectionManager dengan pooling
│       │   ├── resolver.rs     # Connection resolver berdasarkan tenant
│       │   └── migration.rs    # Migration runner
│       ├── config/
│       │   ├── loader.rs       # Config loader dengan hierarchy
│       │   ├── merger.rs       # Config merger (repositories → projects → groups → tenants)
│       │   └── resolver.rs     # Config resolver
│       ├── auth/
│       │   ├── jwt.rs          # JWT generation & validation dengan HS256
│       │   └── middleware.rs   # Auth middleware + tenant resolution
│       └── utils/
│           ├── response.rs     # API response helpers
│           ├── error.rs        # Error handling
│           └── validation.rs   # Input validation
├── repositories/               # All reusable modules (migrated from wellmed)
│   ├── rust-core/              # Core functionality (equivalent to laravel-support)
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── rust-stub/              # Stub generator (equivalent to laravel-stub)
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── rust-package-generator/ # Package generator (equivalent to laravel-package-generator)
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── microtenant/            # Multi-tenant database management
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── tenant_resolver.rs
│   │       └── database_creator.rs
│   ├── module-user/            # User management
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── provider.rs     # Service provider
│   │       ├── entities/
│   │       │   ├── user.rs
│   │       │   └── user_reference.rs
│   │       ├── resources/
│   │       │   ├── user_view_resource.rs
│   │       │   └── user_show_resource.rs
│   │       ├── controllers/
│   │       │   └── user_controller.rs
│   │       └── services/
│   │           └── user_service.rs
│   ├── module-employee/        # Employee management
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-people/          # People management
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-service/         # Service management
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-encoding/        # Encoding utilities
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-event/           # Event management
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-regional/        # Regional data (provinces, districts, etc.)
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── entities/
│   │       │   ├── country.rs
│   │       │   ├── province.rs
│   │       │   ├── district.rs
│   │       │   ├── subdistrict.rs
│   │       │   └── village.rs
│   │       └── ...
│   ├── module-warehouse/       # Warehouse management
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-workspace/       # Workspace management
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-item/            # Item/Product management
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── entities/
│   │       │   ├── item.rs
│   │       │   ├── variant.rs
│   │       │   └── item_has_variant.rs
│   │       └── ...
│   ├── module-transaction/     # Transaction management
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-payment/         # Payment processing
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-organization/    # Organization structure
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-notification/    # Notification system
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-card-identity/   # Card identity management
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-agent/           # Agent management
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-distribution/    # Distribution management
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-funding/         # Funding management
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-handwriting/     # Handwriting/signature
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-license/         # License management
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-manufacture/     # Manufacturing
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-opname-stock/    # Stock opname
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-payer/           # Payer management
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-procurement/     # Procurement
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-profession/      # Profession data
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-summary/         # Summary/reporting
│   │   ├── Cargo.toml
│   │   └── src/...
│   ├── module-support/         # Support utilities
│   │   ├── Cargo.toml
│   │   └── src/...
│   └── module-tax/             # Tax calculation
│       ├── Cargo.toml
│       └── src/...
├── projects/                   # Project-level applications
│   └── pantheon-business/      # Main business application
│       ├── Cargo.toml          # Dependencies ke repositories
│       ├── config/             # Project-level config overrides
│       │   ├── database.toml
│       │   └── entity_connections.toml
│       └── src/
│           ├── main.rs         # Entry point dengan service provider loading
│           ├── routes.rs       # API routes
│           ├── provider.rs     # Project service provider
│           └── modules/        # Project-specific modules
├── groups/                     # Group-level customizations (created on demand)
│   └── pantheon-group/         # Example group
│       ├── Cargo.toml
│       ├── config/             # Group-level config overrides
│       │   └── entity_connections.toml
│       └── src/
│           ├── lib.rs
│           └── provider.rs     # Group service provider
└── tenants/                    # Tenant-level customizations (created on demand)
    └── example-tenant/         # Example tenant
        ├── Cargo.toml
        ├── config/             # Tenant-level config overrides (highest priority)
        │   ├── entity_connections.toml
        │   └── custom_routes.toml
        └── src/
            ├── lib.rs
            ├── provider.rs     # Tenant service provider
            └── custom_modules/ # Tenant-specific overrides
```

**Catatan Penting:**
1. **Service Provider Loading Order:** rust-support → microtenant → rust-stub → rust-package-generator → repositories → projects → groups → tenants
2. **Config Hierarchy:** repositories (default) → projects → groups → tenants (highest priority)
3. **Semua modules di repositories wajib depend ke rust-support**
4. **Groups dan tenants folders hanya dibuat jika ada customization**
5. **Setiap crate punya Cargo.toml sendiri dan punya service provider**

### Frontend (Nuxt 4)

#### Project Structure
```
pantheon-nuxt/                  # Frontend with BFF pattern
├── package.json
├── nuxt.config.ts              # PrimeVue, Iconify, PWA (future) config
├── .env.example
├── .gitignore
├── README.md
├── tsconfig.json
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── assets/
│   └── styles/
│       ├── main.css
│       └── primevue-theme.css  # PrimeVue customization
├── components/
│   ├── auth/
│   │   ├── LoginForm.vue
│   │   └── RegisterForm.vue
│   ├── shared/
│   │   ├── Header.vue
│   │   ├── Sidebar.vue
│   │   ├── Navbar.vue
│   │   └── Footer.vue
│   ├── layout/
│   │   ├── AppTopbar.vue
│   │   └── AppMenu.vue
│   └── common/
│       ├── DataTable.vue
│       ├── Dialog.vue
│       └── Toast.vue
├── composables/
│   ├── useAuth.ts              # Auth logic
│   ├── useApi.ts               # API client
│   ├── useTenant.ts            # Tenant context
│   └── useToast.ts             # Toast notifications
├── layouts/
│   ├── default.vue
│   ├── auth.vue
│   └── dashboard.vue
├── middleware/
│   ├── auth.ts                 # Protected route middleware
│   └── tenant.ts               # Tenant validation middleware
├── pages/
│   ├── index.vue
│   ├── login.vue
│   └── dashboard/
│       ├── index.vue
│       ├── products/
│       │   └── index.vue
│       └── settings/
│           └── index.vue
├── plugins/
│   ├── primevue.ts             # PrimeVue setup
│   └── iconify.ts              # Iconify setup
├── server/                     # BFF (Backend for Frontend) Server
│   ├── api/                    # API endpoints
│   │   ├── auth/
│   │   │   ├── login.post.ts   # Login endpoint
│   │   │   ├── logout.post.ts  # Logout endpoint
│   │   │   ├── refresh.post.ts # Refresh token endpoint
│   │   │   └── me.get.ts       # Get current user
│   │   └── proxy/
│   │       └── [...path].ts    # Proxy ke Rust BE
│   ├── middleware/
│   │   ├── auth.ts             # Server auth middleware
│   │   └── cors.ts             # CORS handling
│   └── utils/
│       ├── jwt.ts              # JWT handling
│       ├── api-client.ts       # HTTP client ke Rust BE
│       └── response.ts         # Response formatter
├── stores/                     # Pinia stores
│   ├── auth.ts                 # Auth state
│   ├── tenant.ts               # Tenant state
│   └── app.ts                  # App global state
└── types/
    ├── auth.ts
    ├── api.ts
    ├── tenant.ts
    └── models.ts
```

### Documentation App (pantheon-docs)

#### Project Structure
```
pantheon-docs/                  # Swagger UI application
├── package.json
├── nuxt.config.ts
├── README.md
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── public/
│   └── openapi.json           # Generated from pantheon-rust
├── pages/
│   ├── index.vue              # Swagger UI page
│   └── api-reference.vue      # API reference
├── components/
│   └── SwaggerUI.vue          # Swagger UI component
└── plans/                     # Project plans
    ├── 1-FOUNDATION-INFRASTRUCTURE-PLAN.md
    ├── 2-BUSINESS-MODULES-PLAN.md
    └── ...
```

### Database Schema

**Database Architecture:**
1. **Core Database:** `pantheon` - Berisi data master dan tenant management
2. **Tenant Databases:** `pantheon_tenant_{tenant_id}` - Satu database per tenant dengan multi-schema
3. **Cluster Schemas:** `cashier_2026`, `cashier_2027`, `scm_2026`, dll - Untuk segmentasi transactional data

#### Core Database: `pantheon`

**Schema: public**
```sql
-- ========================================
-- CORE TABLES (from requirements #74)
-- ========================================

-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    phone VARCHAR(50),
    is_active BOOLEAN DEFAULT true,
    last_login_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    deleted_at TIMESTAMP
);

-- User References (untuk relasi user dengan entities lain)
CREATE TABLE user_references (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    reference_type VARCHAR(100) NOT NULL, -- 'tenant', 'workspace', 'employee', dll
    reference_id UUID NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Domains (untuk multi-domain access)
CREATE TABLE domains (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain VARCHAR(255) UNIQUE NOT NULL,
    tenant_id UUID,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Tenants table
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    database_name VARCHAR(100) UNIQUE NOT NULL,
    is_active BOOLEAN DEFAULT true,
    max_users INT DEFAULT 10,
    max_workspaces INT DEFAULT 5,
    expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    deleted_at TIMESTAMP
);

-- Workspaces (organisasi/cabang dalam tenant)
CREATE TABLE workspaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL,
    address TEXT,
    phone VARCHAR(50),
    email VARCHAR(255),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    deleted_at TIMESTAMP,
    UNIQUE(tenant_id, slug)
);

-- Licenses (untuk license management)
CREATE TABLE licenses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    max_users INT,
    max_workspaces INT,
    duration_days INT,
    price DECIMAL(15,2),
    features JSONB, -- JSON array of features
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Model has Licenses (polymorphic relation)
CREATE TABLE model_has_licenses (
    license_id UUID REFERENCES licenses(id) ON DELETE CASCADE,
    model_type VARCHAR(100) NOT NULL, -- 'tenant', 'workspace', dll
    model_id UUID NOT NULL,
    started_at TIMESTAMP NOT NULL,
    expires_at TIMESTAMP,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (license_id, model_type, model_id)
);

-- Unicodes (untuk generating codes)
CREATE TABLE unicodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type VARCHAR(100) NOT NULL, -- 'invoice', 'transaction', dll
    prefix VARCHAR(20),
    last_number BIGINT DEFAULT 0,
    format VARCHAR(100), -- Format: {PREFIX}{YEAR}{MONTH}{NUMBER}
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(entity_type, prefix)
);

-- API Accesses (untuk tracking API usage dan rate limiting)
CREATE TABLE api_accesses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    endpoint VARCHAR(255) NOT NULL,
    method VARCHAR(10) NOT NULL,
    ip_address VARCHAR(45),
    user_agent TEXT,
    request_body TEXT,
    response_status INT,
    response_time_ms INT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Jobs (untuk background job tracking)
CREATE TABLE jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    queue VARCHAR(100) NOT NULL DEFAULT 'default',
    payload JSONB NOT NULL,
    attempts INT DEFAULT 0,
    reserved_at TIMESTAMP,
    available_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Failed Jobs
CREATE TABLE failed_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    queue VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    exception TEXT,
    failed_at TIMESTAMP DEFAULT NOW()
);

-- ========================================
-- REGIONAL TABLES (from requirements #74)
-- ========================================

-- Countries
CREATE TABLE countries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(10) UNIQUE NOT NULL, -- ID, US, dll
    name VARCHAR(255) NOT NULL,
    dial_code VARCHAR(10),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Provinces
CREATE TABLE provinces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_id UUID REFERENCES countries(id),
    code VARCHAR(20) NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(country_id, code)
);

-- Districts (Kabupaten/Kota)
CREATE TABLE districts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    province_id UUID REFERENCES provinces(id),
    code VARCHAR(20) NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(province_id, code)
);

-- Subdistricts (Kecamatan)
CREATE TABLE subdistricts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    district_id UUID REFERENCES districts(id),
    code VARCHAR(20) NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(district_id, code)
);

-- Villages (Kelurahan/Desa)
CREATE TABLE villages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subdistrict_id UUID REFERENCES subdistricts(id),
    code VARCHAR(20) NOT NULL,
    name VARCHAR(255) NOT NULL,
    postal_code VARCHAR(10),
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(subdistrict_id, code)
);

-- ========================================
-- MIGRATION TRACKING
-- ========================================

-- Migrations (untuk tracking migration versions)
CREATE TABLE migrations (
    id SERIAL PRIMARY KEY,
    version VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    applied_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(version)
);

-- ========================================
-- INDEXES untuk Performance
-- ========================================

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_active ON users(is_active) WHERE deleted_at IS NULL;

CREATE INDEX idx_user_references_user_id ON user_references(user_id);
CREATE INDEX idx_user_references_reference ON user_references(reference_type, reference_id);

CREATE INDEX idx_domains_domain ON domains(domain);
CREATE INDEX idx_domains_tenant_id ON domains(tenant_id);

CREATE INDEX idx_tenants_slug ON tenants(slug);
CREATE INDEX idx_tenants_active ON tenants(is_active) WHERE deleted_at IS NULL;

CREATE INDEX idx_workspaces_tenant_id ON workspaces(tenant_id);
CREATE INDEX idx_workspaces_slug ON workspaces(slug);

CREATE INDEX idx_api_accesses_tenant_id ON api_accesses(tenant_id);
CREATE INDEX idx_api_accesses_user_id ON api_accesses(user_id);
CREATE INDEX idx_api_accesses_created_at ON api_accesses(created_at);

CREATE INDEX idx_jobs_queue ON jobs(queue);
CREATE INDEX idx_jobs_available_at ON jobs(available_at);

CREATE INDEX idx_provinces_country_id ON provinces(country_id);
CREATE INDEX idx_districts_province_id ON districts(province_id);
CREATE INDEX idx_subdistricts_district_id ON subdistricts(district_id);
CREATE INDEX idx_villages_subdistrict_id ON villages(subdistrict_id);
```

#### Tenant Database Template: `pantheon_tenant_{tenant_id}`

**Schema: public (default schema untuk tenant)**
```sql
-- ========================================
-- TENANT TABLES (from requirements #75)
-- ========================================

-- Addresses (alamat untuk people, workspaces, dll)
CREATE TABLE addresses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    addressable_type VARCHAR(100) NOT NULL, -- 'people', 'workspace', dll
    addressable_id UUID NOT NULL,
    address_type VARCHAR(50) DEFAULT 'primary', -- 'primary', 'shipping', 'billing'
    street_address TEXT,
    village_id UUID, -- Reference ke core.villages
    subdistrict_id UUID,
    district_id UUID,
    province_id UUID,
    country_id UUID,
    postal_code VARCHAR(10),
    latitude DECIMAL(10, 8),
    longitude DECIMAL(11, 8),
    is_primary BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    deleted_at TIMESTAMP
);

-- Card Identities (KTP, SIM, Passport, dll)
CREATE TABLE card_identities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id UUID, -- Reference ke peoples
    card_type VARCHAR(50) NOT NULL, -- 'ktp', 'sim', 'passport', dll
    card_number VARCHAR(100) NOT NULL,
    issued_at DATE,
    expires_at DATE,
    issuer VARCHAR(255),
    is_verified BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    deleted_at TIMESTAMP
);

-- Consuments (consumers/customers)
CREATE TABLE consuments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id UUID, -- Reference ke peoples
    code VARCHAR(50) UNIQUE,
    member_since DATE,
    loyalty_points INT DEFAULT 0,
    credit_limit DECIMAL(15,2) DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    deleted_at TIMESTAMP
);

-- Employees (karyawan)
CREATE TABLE employees (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id UUID, -- Reference ke peoples
    employee_code VARCHAR(50) UNIQUE NOT NULL,
    join_date DATE NOT NULL,
    resign_date DATE,
    employment_status VARCHAR(50) DEFAULT 'active', -- 'active', 'resigned', 'terminated'
    position VARCHAR(100),
    department VARCHAR(100),
    salary DECIMAL(15,2),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    deleted_at TIMESTAMP
);

-- Items (products/services)
CREATE TABLE items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    item_type VARCHAR(50) DEFAULT 'product', -- 'product', 'service'
    category VARCHAR(100),
    unit VARCHAR(50) DEFAULT 'pcs',
    base_price DECIMAL(15,2) DEFAULT 0,
    cost_price DECIMAL(15,2) DEFAULT 0,
    has_variants BOOLEAN DEFAULT false,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    deleted_at TIMESTAMP
);

-- Variants (variant produk: ukuran, warna, dll)
CREATE TABLE variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    variant_type VARCHAR(50), -- 'size', 'color', dll
    created_at TIMESTAMP DEFAULT NOW()
);

-- Item has Variants (junction table)
CREATE TABLE item_has_variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_id UUID REFERENCES items(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES variants(id) ON DELETE CASCADE,
    sku VARCHAR(100) UNIQUE NOT NULL,
    price DECIMAL(15,2),
    cost_price DECIMAL(15,2),
    stock INT DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Permissions
CREATE TABLE permissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    module VARCHAR(100), -- 'cashier', 'inventory', dll
    created_at TIMESTAMP DEFAULT NOW()
);

-- Roles
CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    is_system BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Role has Permissions
CREATE TABLE role_has_permissions (
    role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
    permission_id UUID REFERENCES permissions(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (role_id, permission_id)
);

-- Model has Role (polymorphic)
CREATE TABLE model_has_role (
    role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
    model_type VARCHAR(100) NOT NULL, -- 'user', 'employee', dll
    model_id UUID NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (role_id, model_type, model_id)
);

-- Model has Permissions (polymorphic - direct permission assignment)
CREATE TABLE model_has_permissions (
    permission_id UUID REFERENCES permissions(id) ON DELETE CASCADE,
    model_type VARCHAR(100) NOT NULL,
    model_id UUID NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (permission_id, model_type, model_id)
);

-- Model has Organizations (polymorphic - untuk multi-workspace)
CREATE TABLE model_has_organizations (
    organization_id UUID NOT NULL, -- workspace_id
    model_type VARCHAR(100) NOT NULL,
    model_id UUID NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (organization_id, model_type, model_id)
);

-- Model has Warehouses (polymorphic)
CREATE TABLE model_has_warehouses (
    warehouse_id UUID NOT NULL,
    model_type VARCHAR(100) NOT NULL,
    model_id UUID NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (warehouse_id, model_type, model_id)
);

-- Model has Services (polymorphic)
CREATE TABLE model_has_services (
    service_id UUID NOT NULL,
    model_type VARCHAR(100) NOT NULL,
    model_id UUID NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (service_id, model_type, model_id)
);

-- Model has Phones (polymorphic - untuk multiple phone numbers)
CREATE TABLE model_has_phones (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_type VARCHAR(100) NOT NULL,
    model_id UUID NOT NULL,
    phone_type VARCHAR(50) DEFAULT 'mobile', -- 'mobile', 'office', 'home'
    phone_number VARCHAR(50) NOT NULL,
    country_code VARCHAR(10) DEFAULT '+62',
    is_primary BOOLEAN DEFAULT false,
    is_verified BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Model has Encodings (polymorphic - untuk encoding/hashing data)
CREATE TABLE model_has_encodings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_type VARCHAR(100) NOT NULL,
    model_id UUID NOT NULL,
    encoding_type VARCHAR(50) NOT NULL, -- 'barcode', 'qrcode', dll
    encoded_value TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(model_type, model_id, encoding_type)
);

-- Notifications
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notifiable_type VARCHAR(100) NOT NULL,
    notifiable_id UUID NOT NULL,
    notification_type VARCHAR(100) NOT NULL,
    title VARCHAR(255),
    message TEXT,
    data JSONB,
    read_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Peoples (data person/individu)
CREATE TABLE peoples (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100),
    full_name VARCHAR(255),
    gender VARCHAR(20), -- 'male', 'female', 'other'
    birth_date DATE,
    birth_place VARCHAR(255),
    email VARCHAR(255),
    phone VARCHAR(50),
    avatar_url TEXT,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    deleted_at TIMESTAMP
);

-- Rooms (ruangan untuk workspace)
CREATE TABLE rooms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID,
    code VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    floor INT,
    capacity INT,
    room_type VARCHAR(50), -- 'meeting', 'office', 'storage'
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Stocks (inventory stock)
CREATE TABLE stocks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id UUID NOT NULL,
    item_id UUID REFERENCES items(id),
    variant_id UUID REFERENCES item_has_variants(id),
    quantity DECIMAL(15,3) DEFAULT 0,
    reserved_quantity DECIMAL(15,3) DEFAULT 0,
    available_quantity DECIMAL(15,3) DEFAULT 0,
    last_stock_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(warehouse_id, item_id, variant_id)
);

-- Summaries (untuk summary reports)
CREATE TABLE summaries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    summary_type VARCHAR(100) NOT NULL, -- 'daily_sales', 'monthly_revenue', dll
    reference_id UUID,
    summary_date DATE NOT NULL,
    data JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(summary_type, reference_id, summary_date)
);

-- Transactions (transaction header)
CREATE TABLE transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_code VARCHAR(100) UNIQUE NOT NULL,
    transaction_type VARCHAR(50) NOT NULL, -- 'sales', 'purchase', 'transfer'
    transaction_date TIMESTAMP NOT NULL,
    workspace_id UUID,
    customer_id UUID,
    supplier_id UUID,
    total_amount DECIMAL(15,2) DEFAULT 0,
    discount_amount DECIMAL(15,2) DEFAULT 0,
    tax_amount DECIMAL(15,2) DEFAULT 0,
    final_amount DECIMAL(15,2) DEFAULT 0,
    status VARCHAR(50) DEFAULT 'pending', -- 'pending', 'completed', 'cancelled'
    notes TEXT,
    created_by UUID,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Unicodes (untuk generating codes di tenant level)
CREATE TABLE unicodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type VARCHAR(100) NOT NULL,
    prefix VARCHAR(20),
    last_number BIGINT DEFAULT 0,
    format VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(entity_type, prefix)
);

-- User Wallets (untuk e-wallet/credit balance)
CREATE TABLE user_wallets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    wallet_type VARCHAR(50) DEFAULT 'credit', -- 'credit', 'point', dll
    balance DECIMAL(15,2) DEFAULT 0,
    currency VARCHAR(10) DEFAULT 'IDR',
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(user_id, wallet_type)
);

-- Warehouse Items (alternative structure untuk item di warehouse)
CREATE TABLE warehouse_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id UUID NOT NULL,
    item_id UUID REFERENCES items(id),
    min_stock DECIMAL(15,3) DEFAULT 0,
    max_stock DECIMAL(15,3) DEFAULT 0,
    reorder_point DECIMAL(15,3) DEFAULT 0,
    location VARCHAR(100), -- 'A1', 'B2', dll
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(warehouse_id, item_id)
);

-- Master Reports (untuk report configuration)
CREATE TABLE master_reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    report_type VARCHAR(50), -- 'sales', 'inventory', 'financial'
    query_template TEXT,
    parameters JSONB,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Migrations tracking
CREATE TABLE migrations (
    id SERIAL PRIMARY KEY,
    version VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    applied_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(version)
);

-- ========================================
-- INDEXES
-- ========================================

CREATE INDEX idx_addresses_addressable ON addresses(addressable_type, addressable_id);
CREATE INDEX idx_card_identities_person_id ON card_identities(person_id);
CREATE INDEX idx_consuments_person_id ON consuments(person_id);
CREATE INDEX idx_employees_person_id ON employees(person_id);
CREATE INDEX idx_items_code ON items(code);
CREATE INDEX idx_items_active ON items(is_active) WHERE deleted_at IS NULL;
CREATE INDEX idx_stocks_warehouse_item ON stocks(warehouse_id, item_id);
CREATE INDEX idx_transactions_date ON transactions(transaction_date);
CREATE INDEX idx_transactions_workspace ON transactions(workspace_id);
CREATE INDEX idx_notifications_notifiable ON notifications(notifiable_type, notifiable_id);
CREATE INDEX idx_notifications_unread ON notifications(notifiable_type, notifiable_id) WHERE read_at IS NULL;
```

#### Cluster Schemas (in Tenant Database)

**Schema: cashier_2026, cashier_2027, etc.**
```sql
-- ========================================
-- CASHIER CLUSTER TABLES (from requirements)
-- ========================================

-- Billings (tagihan)
CREATE TABLE billings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    billing_code VARCHAR(100) UNIQUE NOT NULL,
    transaction_id UUID NOT NULL,
    customer_id UUID,
    billing_date TIMESTAMP NOT NULL,
    due_date DATE,
    total_amount DECIMAL(15,2) DEFAULT 0,
    paid_amount DECIMAL(15,2) DEFAULT 0,
    remaining_amount DECIMAL(15,2) DEFAULT 0,
    status VARCHAR(50) DEFAULT 'unpaid', -- 'unpaid', 'partial', 'paid', 'overdue'
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Payment Summaries (ringkasan pembayaran)
CREATE TABLE payment_summaries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    summary_date DATE NOT NULL,
    workspace_id UUID,
    cashier_id UUID,
    total_transactions INT DEFAULT 0,
    total_amount DECIMAL(15,2) DEFAULT 0,
    cash_amount DECIMAL(15,2) DEFAULT 0,
    card_amount DECIMAL(15,2) DEFAULT 0,
    ewallet_amount DECIMAL(15,2) DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(summary_date, workspace_id, cashier_id)
);

-- Payment Histories (history pembayaran)
CREATE TABLE payment_histories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    billing_id UUID NOT NULL,
    payment_date TIMESTAMP NOT NULL,
    amount DECIMAL(15,2) NOT NULL,
    payment_method VARCHAR(50) NOT NULL,
    reference_number VARCHAR(100),
    notes TEXT,
    created_by UUID,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Payment Details (detail pembayaran per item)
CREATE TABLE payment_details (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_id UUID NOT NULL,
    item_id UUID NOT NULL,
    quantity DECIMAL(15,3) NOT NULL,
    unit_price DECIMAL(15,2) NOT NULL,
    discount_amount DECIMAL(15,2) DEFAULT 0,
    tax_amount DECIMAL(15,2) DEFAULT 0,
    subtotal DECIMAL(15,2) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Invoices (faktur)
CREATE TABLE invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_number VARCHAR(100) UNIQUE NOT NULL,
    billing_id UUID NOT NULL,
    invoice_date DATE NOT NULL,
    due_date DATE,
    status VARCHAR(50) DEFAULT 'draft', -- 'draft', 'sent', 'paid', 'cancelled'
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Refunds (pengembalian dana)
CREATE TABLE refunds (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    refund_code VARCHAR(100) UNIQUE NOT NULL,
    original_transaction_id UUID NOT NULL,
    refund_date TIMESTAMP NOT NULL,
    refund_amount DECIMAL(15,2) NOT NULL,
    refund_reason TEXT,
    status VARCHAR(50) DEFAULT 'pending', -- 'pending', 'approved', 'rejected', 'completed'
    approved_by UUID,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Split Payments (pembayaran terpisah)
CREATE TABLE split_payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    billing_id UUID NOT NULL,
    payment_method VARCHAR(50) NOT NULL,
    amount DECIMAL(15,2) NOT NULL,
    reference_number VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Wallet Transactions (transaksi e-wallet)
CREATE TABLE wallet_transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    wallet_id UUID NOT NULL,
    transaction_type VARCHAR(50) NOT NULL, -- 'credit', 'debit'
    amount DECIMAL(15,2) NOT NULL,
    balance_before DECIMAL(15,2) NOT NULL,
    balance_after DECIMAL(15,2) NOT NULL,
    reference_type VARCHAR(100),
    reference_id UUID,
    description TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Transaction Items (detail item transaksi)
CREATE TABLE transaction_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_id UUID NOT NULL,
    item_id UUID NOT NULL,
    variant_id UUID,
    quantity DECIMAL(15,3) NOT NULL,
    unit_price DECIMAL(15,2) NOT NULL,
    discount_amount DECIMAL(15,2) DEFAULT 0,
    tax_amount DECIMAL(15,2) DEFAULT 0,
    subtotal DECIMAL(15,2) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_billings_transaction_id ON billings(transaction_id);
CREATE INDEX idx_billings_date ON billings(billing_date);
CREATE INDEX idx_payment_summaries_date ON payment_summaries(summary_date);
CREATE INDEX idx_payment_histories_billing_id ON payment_histories(billing_id);
CREATE INDEX idx_invoices_billing_id ON invoices(billing_id);
CREATE INDEX idx_wallet_transactions_wallet_id ON wallet_transactions(wallet_id);
CREATE INDEX idx_transaction_items_transaction_id ON transaction_items(transaction_id);
```

**Schema: scm_2026, scm_2027, etc.**
```sql
-- ========================================
-- SCM CLUSTER TABLES (from requirements)
-- ========================================

-- Card Stocks (kartu stok)
CREATE TABLE card_stocks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id UUID NOT NULL,
    item_id UUID NOT NULL,
    variant_id UUID,
    stock_date DATE NOT NULL,
    beginning_stock DECIMAL(15,3) DEFAULT 0,
    stock_in DECIMAL(15,3) DEFAULT 0,
    stock_out DECIMAL(15,3) DEFAULT 0,
    ending_stock DECIMAL(15,3) DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(warehouse_id, item_id, variant_id, stock_date)
);

-- Distributions (distribusi barang)
CREATE TABLE distributions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    distribution_code VARCHAR(100) UNIQUE NOT NULL,
    distribution_date TIMESTAMP NOT NULL,
    from_warehouse_id UUID NOT NULL,
    to_warehouse_id UUID NOT NULL,
    status VARCHAR(50) DEFAULT 'pending', -- 'pending', 'in_transit', 'received', 'cancelled'
    notes TEXT,
    created_by UUID,
    approved_by UUID,
    received_by UUID,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Opname Stocks (stock opname)
CREATE TABLE opname_stocks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    opname_code VARCHAR(100) UNIQUE NOT NULL,
    opname_date DATE NOT NULL,
    warehouse_id UUID NOT NULL,
    status VARCHAR(50) DEFAULT 'draft', -- 'draft', 'in_progress', 'completed', 'cancelled'
    notes TEXT,
    created_by UUID,
    approved_by UUID,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Procurements (pengadaan barang)
CREATE TABLE procurements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    procurement_code VARCHAR(100) UNIQUE NOT NULL,
    procurement_date DATE NOT NULL,
    supplier_id UUID,
    warehouse_id UUID NOT NULL,
    total_amount DECIMAL(15,2) DEFAULT 0,
    status VARCHAR(50) DEFAULT 'draft', -- 'draft', 'submitted', 'approved', 'completed', 'cancelled'
    notes TEXT,
    created_by UUID,
    approved_by UUID,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Purchase Orders (PO)
CREATE TABLE purchase_orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    po_number VARCHAR(100) UNIQUE NOT NULL,
    po_date DATE NOT NULL,
    supplier_id UUID NOT NULL,
    delivery_date DATE,
    total_amount DECIMAL(15,2) DEFAULT 0,
    status VARCHAR(50) DEFAULT 'draft', -- 'draft', 'sent', 'confirmed', 'completed', 'cancelled'
    notes TEXT,
    created_by UUID,
    approved_by UUID,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Procurement Lists (detail item procurement)
CREATE TABLE procurement_lists (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    procurement_id UUID NOT NULL,
    item_id UUID NOT NULL,
    variant_id UUID,
    quantity DECIMAL(15,3) NOT NULL,
    unit_price DECIMAL(15,2) NOT NULL,
    discount_amount DECIMAL(15,2) DEFAULT 0,
    tax_amount DECIMAL(15,2) DEFAULT 0,
    subtotal DECIMAL(15,2) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Purchase Requests (permintaan pembelian)
CREATE TABLE purchase_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pr_number VARCHAR(100) UNIQUE NOT NULL,
    request_date DATE NOT NULL,
    requested_by UUID NOT NULL,
    department VARCHAR(100),
    status VARCHAR(50) DEFAULT 'pending', -- 'pending', 'approved', 'rejected', 'completed'
    notes TEXT,
    approved_by UUID,
    approved_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Receive Orders (penerimaan barang)
CREATE TABLE receive_orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ro_number VARCHAR(100) UNIQUE NOT NULL,
    receive_date TIMESTAMP NOT NULL,
    purchase_order_id UUID,
    warehouse_id UUID NOT NULL,
    received_by UUID,
    status VARCHAR(50) DEFAULT 'draft', -- 'draft', 'partial', 'completed'
    notes TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_card_stocks_warehouse_item ON card_stocks(warehouse_id, item_id);
CREATE INDEX idx_card_stocks_date ON card_stocks(stock_date);
CREATE INDEX idx_distributions_from_warehouse ON distributions(from_warehouse_id);
CREATE INDEX idx_distributions_to_warehouse ON distributions(to_warehouse_id);
CREATE INDEX idx_distributions_date ON distributions(distribution_date);
CREATE INDEX idx_opname_stocks_warehouse ON opname_stocks(warehouse_id);
CREATE INDEX idx_opname_stocks_date ON opname_stocks(opname_date);
CREATE INDEX idx_procurements_supplier ON procurements(supplier_id);
CREATE INDEX idx_procurements_warehouse ON procurements(warehouse_id);
CREATE INDEX idx_purchase_orders_supplier ON purchase_orders(supplier_id);
CREATE INDEX idx_purchase_requests_requested_by ON purchase_requests(requested_by);
CREATE INDEX idx_receive_orders_po_id ON receive_orders(purchase_order_id);
```

## Implementation Steps

### Phase 1: Backend Foundation & Core Libraries (Minggu 1-2)

#### Step 1.0: GitHub Organization & Repository Setup

**Note:** Existing folder `pantheon-rust` di local akan diganti dengan `pantheon-app` dari GitHub. Backup dulu jika ada code penting.

```bash
# Backup existing pantheon-rust (jika ada)
cd /var/www/projects/pantheon
if [ -d "pantheon-rust" ]; then
  mv pantheon-rust pantheon-rust.backup-$(date +%Y%m%d)
fi
```

**Verify GitHub Account:**
```bash
# Verify login (sudah login sebagai hanafalah dengan email hamzahnafalah@gmail.com)
gh auth status

# Should show: Logged in to github.com as hanafalah (hamzahnafalah@gmail.com)

# Create organization "pantheon" (jika belum ada)
# Note: Ini akan prompt untuk create via browser
gh org create pantheon --enable-issues --enable-projects
```

**Create Repositories under Organization:**
```bash
# 1. Create pantheon-app repository (main Rust backend)
gh repo create pantheon/pantheon-app \
  --public \
  --description "Pantheon App - Multi-tenant Business Management System (Rust Backend)" \
  --gitignore Rust \
  --license MIT

# 2. Create pantheon-nuxt repository (frontend)
gh repo create pantheon/pantheon-nuxt \
  --public \
  --description "Pantheon App - Frontend with Nuxt 4 and BFF Pattern" \
  --gitignore Node \
  --license MIT

# 3. Create pantheon-docs repository (documentation & swagger UI)
gh repo create pantheon/pantheon-docs \
  --public \
  --description "Pantheon App - Documentation and API Reference" \
  --gitignore Node \
  --license MIT
```

**Clone Repositories ke Local:**
```bash
# Navigate ke working directory
cd /var/www/projects/pantheon

# Clone pantheon-app (backend)
gh repo clone pantheon/pantheon-app
cd pantheon-app

# Setup branches
git checkout -b develop
git push -u origin develop
git checkout -b staging
git push -u origin staging
git checkout main

# Set default branch to develop for development
gh repo edit pantheon/pantheon-app --default-branch develop

cd ..

# Clone pantheon-nuxt (frontend)
gh repo clone pantheon/pantheon-nuxt
cd pantheon-nuxt

# Setup branches
git checkout -b develop
git push -u origin develop
git checkout -b staging
git push -u origin staging
git checkout main

gh repo edit pantheon/pantheon-nuxt --default-branch develop

cd ..

# Clone pantheon-docs
gh repo clone pantheon/pantheon-docs
cd pantheon-docs

# Setup branches
git checkout -b develop
git push -u origin develop
git checkout -b staging
git push -u origin staging
git checkout main

gh repo edit pantheon/pantheon-docs --default-branch develop
```

**Setup Branch Protection Rules:**
```bash
# For pantheon-app
gh api repos/pantheon/pantheon-app/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["test","build"]}' \
  --field enforce_admins=true \
  --field required_pull_request_reviews='{"required_approving_review_count":1}' \
  --field restrictions=null

gh api repos/pantheon/pantheon-app/branches/staging/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["test","build"]}' \
  --field enforce_admins=false \
  --field required_pull_request_reviews='{"required_approving_review_count":1}' \
  --field restrictions=null

# For pantheon-nuxt
gh api repos/pantheon/pantheon-nuxt/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["lint","test","build"]}' \
  --field enforce_admins=true \
  --field required_pull_request_reviews='{"required_approving_review_count":1}' \
  --field restrictions=null

# For pantheon-docs
gh api repos/pantheon/pantheon-docs/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["build"]}' \
  --field enforce_admins=true \
  --field required_pull_request_reviews='{"required_approving_review_count":1}' \
  --field restrictions=null
```

**Alternative: Jika Organization Creation Failed, Use Personal Repos:**
```bash
# Jika tidak bisa create organization, bisa create di personal account
# dengan naming convention: hanafalah/pantheon-app, hanafalah/pantheon-nuxt, dll

gh repo create hanafalah/pantheon-app \
  --public \
  --description "Pantheon App - Multi-tenant Business Management System (Rust Backend)" \
  --gitignore Rust \
  --license MIT

gh repo create hanafalah/pantheon-nuxt \
  --public \
  --description "Pantheon App - Frontend with Nuxt 4 and BFF Pattern" \
  --gitignore Node \
  --license MIT

gh repo create hanafalah/pantheon-docs \
  --public \
  --description "Pantheon App - Documentation and API Reference" \
  --gitignore Node \
  --license MIT

# Then clone with personal account path
gh repo clone hanafalah/pantheon-app
gh repo clone hanafalah/pantheon-nuxt
gh repo clone hanafalah/pantheon-docs
```

**Setup GitHub Secrets (untuk CI/CD):**
```bash
# Untuk pantheon-app
cd /var/www/projects/pantheon/pantheon-app

# Add secrets
gh secret set JWT_SECRET --body "YXYlGIbJ65VGjQnETWX23iCvssXg7PJu"
gh secret set DATABASE_URL --body "postgresql://pantheon_app:password@localhost:5432/pantheon"
gh secret set RABBITMQ_URL --body "amqp://guest:guest@localhost:5672"

# Untuk Docker registry (optional)
gh secret set DOCKER_USERNAME --body "your-docker-username"
gh secret set DOCKER_PASSWORD --body "your-docker-password"

# Untuk pantheon-nuxt
cd /var/www/projects/pantheon/pantheon-nuxt
gh secret set NUXT_PUBLIC_API_URL --body "http://localhost:8000"
```

**Verify Setup:**
```bash
# List repos
gh repo list pantheon --limit 10

# Or for personal account
gh repo list hanafalah --limit 10

# View repo details
gh repo view pantheon/pantheon-app
gh repo view pantheon/pantheon-nuxt
gh repo view pantheon/pantheon-docs
```

#### Step 1.1: Initialize Rust Workspace (pantheon-app)
```bash
cd /var/www/projects/pantheon/pantheon-app
```

- [ ] Create root Cargo.toml sebagai workspace dengan workspace members
- [ ] Setup folder structure: repositories/, projects/, groups/, tenants/
- [ ] Configure .gitignore, .env.example, README.md
- [ ] Setup Docker infrastructure (Dockerfile, docker-compose.yml)
- [ ] Initialize git repository dengan proper .gitattributes (already done via gh repo create)
- [ ] Create initial commit dan push ke develop branch

#### Step 1.2: Implement rust-support Library (CORE DEPENDENCY)
**Base Traits & Structs:**
- [ ] BaseEntity trait dengan methods:
  - `get_view_resource() -> Box<dyn ViewResource>`
  - `get_show_resource() -> Box<dyn ShowResource>`
  - `to_view_api(&self) -> Result<Value>`
  - `to_show_api(&self) -> Result<Value>`
  - `get_connection_name() -> &str`
  - `get_table_name() -> &str`
- [ ] BaseResource trait dengan methods:
  - `transform(&self, entity: &dyn Any) -> Result<Value>`
- [ ] ViewResource struct - untuk list/index views
- [ ] ShowResource struct - extends ViewResource untuk detail views
- [ ] BaseController trait dengan methods:
  - `index()`, `show()`, `store()`, `update()`, `destroy()`
- [ ] BaseService trait dengan dependency injection support

**Service Provider Pattern:**
- [ ] ServiceProvider trait dengan methods:
  - `register(&mut self, app: &mut App)`
  - `boot(&self, app: &App)`
  - `provides() -> Vec<String>` - return service names
- [ ] ServiceProviderRegistry - untuk managing provider loading order
- [ ] Auto-discovery mechanism untuk providers

**Database Infrastructure:**
- [ ] ConnectionManager
  - Multi-database connection pooling (core, tenant_*)
  - Connection configuration loading dari TOML
  - Connection health checks & auto-reconnect
  - Connection pool sizing per database
- [ ] DatabaseManager
  - Tenant resolution dari JWT token
  - Entity-to-connection mapping resolver
  - Dynamic connection switching per request
  - Cluster schema resolver (cashier_2026, scm_2026)
- [ ] MigrationManager
  - Run migrations per database (core, tenant_*)
  - Track migration versions in migrations table
  - Rollback support
  - Migration file discovery

**Configuration System:**
- [ ] ConfigLoader - load TOML configs dengan hierarchy
- [ ] ConfigMerger - merge configs: repositories → projects → groups → tenants
- [ ] ConfigResolver - resolve final config at runtime
- [ ] Support untuk hot-reload configs (untuk development)

**Auth Infrastructure:**
- [ ] JWT utilities dengan HS256 dan secret dari requirements
  - `generate_access_token(user, tenant)` - expired 15 menit
  - `generate_refresh_token(user)` - expired 7 hari
  - `verify_token(token)` - verify signature dan expiry
  - `decode_token(token)` - extract claims
  - `extract_tenant_id(token)` - untuk connection mapping
- [ ] Auth middleware
  - Token validation dari Authorization header
  - Tenant extraction dari token claims
  - User context injection ke request
  - Connection setup berdasarkan tenant
- [ ] Password hashing dengan bcrypt (cost: 12)

**Utilities:**
- [ ] ApiResponse builder (success, error, pagination)
- [ ] ErrorHandler dengan error codes
- [ ] Validator dengan custom rules
- [ ] Logger dengan structured logging

#### Step 1.3: Setup Core Support Repositories
**rust-core (Laravel Support equivalent):**
- [ ] Create Cargo.toml dengan dependency ke rust-support
- [ ] Helper functions untuk common operations
- [ ] Macro definitions untuk code generation

**rust-stub (Laravel Stub equivalent):**
- [ ] Template engine untuk code generation
- [ ] Stub files untuk entity, resource, controller, service, migration
- [ ] CLI commands untuk generating code

**rust-package-generator (Laravel Package Generator):**
- [ ] CLI tool untuk generating new modules
- [ ] Auto-generate Cargo.toml untuk new modules
- [ ] Auto-generate folder structure
- [ ] Auto-register ke workspace

**microtenant (Multi-tenant Management):**
- [ ] TenantResolver - resolve tenant dari request
- [ ] DatabaseCreator - auto-create pantheon_tenant_* databases
- [ ] TenantMigrator - run migrations untuk tenant databases
- [ ] ClusterSchemaManager - manage cluster schemas (cashier_*, scm_*)
- [ ] TenantSeeder - seed default data untuk new tenant

### Phase 2: Repository Migration dari Wellmed (Minggu 2-3)

#### Step 2.1: Migrate Core Modules
**Priority 1 - Foundation Modules:**
- [ ] module-user (Users & User References)
  - Migrate entities: User, UserReference
  - Create ViewResource & ShowResource
  - Implement UserService dengan dependency injection
  - Create UserController dengan BaseController
  - Add service provider
- [ ] module-regional (Countries, Provinces, Districts, Subdistricts, Villages)
  - Migrate all regional entities dari Laravel Models
  - Create seeders untuk Indonesia regional data
  - Add service provider
- [ ] module-workspace (Workspaces & Organizations)
  - Migrate workspace entities
  - Implement multi-workspace support
  - Add service provider
- [ ] module-encoding (Barcode, QR Code generation)
  - Migrate encoding utilities
  - Implement model_has_encodings support
  - Add service provider

**Priority 2 - Business Modules:**
- [ ] module-people (Peoples, Addresses, Card Identities)
  - Migrate people management entities
  - Implement polymorphic addresses
  - Implement polymorphic card_identities
  - Add service provider
- [ ] module-employee (Employees)
  - Migrate employee entities dan relations
  - Link dengan module-people
  - Add service provider
- [ ] module-item (Items, Variants, Item Has Variants)
  - Migrate product/item entities
  - Implement variant management
  - Add service provider
- [ ] module-warehouse (Warehouses, Stocks, Warehouse Items)
  - Migrate warehouse entities
  - Implement stock management
  - Link dengan module-item
  - Add service provider

**Priority 3 - Transaction Modules:**
- [ ] module-transaction (Transactions, Transaction Items)
  - Migrate transaction entities
  - Implement transaction flow
  - Link dengan cluster schemas
  - Add service provider
- [ ] module-payment (Payments, Billings)
  - Migrate payment entities
  - Link dengan cashier cluster schemas
  - Add service provider
- [ ] module-service (Services)
  - Migrate service entities
  - Add service provider
- [ ] module-event (Events & Event Handlers)
  - Migrate event system
  - Add service provider

**Priority 4 - Supporting Modules:**
- [ ] module-notification (Notifications)
  - Migrate notification system
  - Add service provider
- [ ] module-organization (Organizations)
  - Migrate organization structure
  - Add service provider
- [ ] module-card-identity (Extended card identity features)
  - Add service provider
- [ ] module-agent (Agents)
  - Migrate agent entities
  - Add service provider
- [ ] module-distribution (Distributions)
  - Link dengan scm cluster schemas
  - Add service provider
- [ ] module-funding (Funding)
  - Add service provider
- [ ] module-handwriting (Signatures)
  - Add service provider
- [ ] module-license (License Management)
  - Migrate licenses, model_has_licenses
  - Add service provider
- [ ] module-manufacture (Manufacturing)
  - Add service provider
- [ ] module-opname-stock (Stock Opname)
  - Link dengan scm cluster schemas
  - Add service provider
- [ ] module-payer (Payers)
  - Add service provider
- [ ] module-procurement (Procurements, Purchase Orders)
  - Link dengan scm cluster schemas
  - Add service provider
- [ ] module-profession (Professions)
  - Add service provider
- [ ] module-summary (Summary Reports)
  - Implement summary generation
  - Add service provider
- [ ] module-support (Support utilities)
  - Add service provider
- [ ] module-tax (Tax Calculation)
  - Add service provider

**Note:** Setiap module WAJIB:
1. Depend ke rust-support
2. Punya service provider
3. Punya Cargo.toml
4. Follow struktur: entities/, resources/, controllers/, services/
5. Implement BaseEntity untuk semua entities
6. Implement ViewResource & ShowResource

#### Step 2.2: Configure Entity Connection Mapping
- [ ] Create `config/entity_connections.toml` di core
- [ ] Map entities ke connection levels:
  - core: users, user_references, tenants, workspaces, licenses, regional tables
  - hq_*: (untuk future pantheon-hq project)
  - group_*: (untuk group customizations)
  - tenant: default untuk entities tidak terdefinisi (addresses, peoples, employees, items, etc.)
  - cluster cashier_*: billings, payments, invoices, wallet transactions
  - cluster scm_*: procurements, distributions, stock cards
- [ ] Implement connection resolver di rust-support
- [ ] Test connection mapping dengan sample entities

### Phase 3: Database Setup & Migrations (Minggu 3)

#### Step 3.1: PostgreSQL Infrastructure
- [ ] Install PostgreSQL 15+ dengan support untuk multi-database
- [ ] Create database `pantheon` (core database)
- [ ] Create database `pantheon_tenant_template` (template for tenants)
- [ ] Setup PostgreSQL users:
  - pantheon_admin (superuser untuk migrations)
  - pantheon_app (application user dengan limited permissions)
- [ ] Configure connection pooling settings
- [ ] Setup database backup strategy

#### Step 3.2: Core Database Migrations
- [ ] Create migration: 001_create_users_and_references.sql
  - users table
  - user_references table
  - refresh_tokens table (untuk JWT)
- [ ] Create migration: 002_create_tenants_and_workspaces.sql
  - tenants table
  - workspaces table
  - domains table
- [ ] Create migration: 003_create_licenses.sql
  - licenses table
  - model_has_licenses table
- [ ] Create migration: 004_create_regional_tables.sql
  - countries, provinces, districts, subdistricts, villages
- [ ] Create migration: 005_create_utility_tables.sql
  - unicodes table
  - api_accesses table
  - jobs table
  - failed_jobs table
- [ ] Create migration: 006_create_migrations_tracking.sql
  - migrations table
- [ ] Run all core migrations
- [ ] Verify schema creation dengan SQL queries

#### Step 3.3: Tenant Database Migrations (Template)
- [ ] Create migration: 001_create_base_tables.sql
  - peoples, addresses, card_identities
  - consuments, employees
  - rooms, workspaces (tenant-level)
- [ ] Create migration: 002_create_permission_tables.sql
  - permissions, roles
  - role_has_permissions
  - model_has_role, model_has_permissions
- [ ] Create migration: 003_create_item_tables.sql
  - items, variants, item_has_variants
  - warehouse_items
- [ ] Create migration: 004_create_stock_tables.sql
  - stocks, summaries
- [ ] Create migration: 005_create_transaction_tables.sql
  - transactions
  - user_wallets
- [ ] Create migration: 006_create_polymorphic_tables.sql
  - model_has_organizations
  - model_has_warehouses
  - model_has_services
  - model_has_phones
  - model_has_encodings
- [ ] Create migration: 007_create_notification_tables.sql
  - notifications
- [ ] Create migration: 008_create_utility_tables.sql
  - unicodes, master_reports, migrations

#### Step 3.4: Cluster Schema Migrations (Templates)
**Cashier Cluster (cashier_YYYY or cashier_YYYY_MM):**
- [ ] Create migration: 001_create_cashier_tables.sql
  - billings, invoices, refunds
  - payment_summaries, payment_histories, payment_details
  - split_payments
  - wallet_transactions
  - transaction_items

**SCM Cluster (scm_YYYY or scm_YYYY_MM):**
- [ ] Create migration: 001_create_scm_tables.sql
  - card_stocks
  - distributions
  - opname_stocks
  - procurements, procurement_lists
  - purchase_orders, purchase_requests
  - receive_orders

#### Step 3.5: Seed Data
**Core Database Seeds:**
- [ ] Seed Indonesia regional data (provinces, districts, subdistricts, villages)
- [ ] Seed default licenses (Free, Basic, Professional, Enterprise)
- [ ] Seed default tenant untuk testing: "Example Tenant"
- [ ] Seed test users (admin@pantheon.local, user@pantheon.local)

**Tenant Database Seeds:**
- [ ] Seed default permissions untuk semua modules
- [ ] Seed default roles (super-admin, admin, manager, cashier, warehouse-staff, user)
- [ ] Seed role permissions mapping
- [ ] Seed test data: peoples, employees, items, warehouses

#### Step 3.6: Test Database Infrastructure
- [ ] Test connection ke core database
- [ ] Test connection ke tenant database
- [ ] Test cluster schema creation (manual create cashier_2026, scm_2026)
- [ ] Test multi-database queries
- [ ] Test connection pooling under load
- [ ] Test tenant isolation

### Phase 4: Projects Setup & Service Provider Loading (Minggu 4)

#### Step 4.1: Create pantheon-business Project
- [ ] Create `projects/pantheon-business/` folder
- [ ] Create Cargo.toml dengan dependencies:
  - rust-support
  - microtenant
  - rust-core
  - semua module repositories yang dibutuhkan
- [ ] Create `src/main.rs` sebagai entry point
- [ ] Create `src/provider.rs` - ProjectServiceProvider
- [ ] Create `src/routes.rs` - API route definitions
- [ ] Create `config/` folder untuk project-level overrides
  - database.toml
  - entity_connections.toml
  - app.toml

#### Step 4.2: Implement Service Provider Loading
**Di main.rs:**
- [ ] Initialize ServiceProviderRegistry
- [ ] Load providers dalam urutan:
  1. rust-support providers (core infrastructure)
  2. microtenant provider (multi-tenant setup)
  3. rust-stub provider (code generation)
  4. rust-package-generator provider (package management)
  5. All repository providers (from modules)
  6. Project provider (pantheon-business)
  7. Group providers (if exists - runtime loaded)
  8. Tenant providers (if exists - runtime loaded)
- [ ] Call `register()` pada semua providers
- [ ] Call `boot()` pada semua providers
- [ ] Start web server

**Testing:**
- [ ] Test provider loading order
- [ ] Test provider dependencies
- [ ] Test service registration
- [ ] Verify all modules loaded correctly

#### Step 4.3: Implement Configuration Resolution
- [ ] Load base config dari rust-support/config
- [ ] Load repository configs dari repositories/*/config
- [ ] Load project config dari projects/pantheon-business/config
- [ ] Merge configs dengan priority: tenant > group > project > repositories
- [ ] Make merged config available globally via AppState
- [ ] Test config override mechanism

#### Step 4.4: Setup API Routes
- [ ] Define route structure: `/api/v1/{resource}`
- [ ] Implement route registration dari modules
- [ ] Support route override: repositories < projects < groups < tenants
- [ ] Add middleware stack: logging, cors, auth, tenant-resolution
- [ ] Test route resolution

### Phase 5: Auth Module Implementation (Minggu 4)

#### Step 5.1: User Entity & Resources
**In module-user:**
- [ ] User entity implementing BaseEntity
  - Fields: id, email, username, password_hash, first_name, last_name, phone
  - get_connection_name() returns "core" (stored in core database)
  - get_view_resource() returns UserViewResource
  - get_show_resource() returns UserShowResource
- [ ] UserViewResource (untuk list):
  - id, email, username, first_name, last_name, is_active
- [ ] UserShowResource extends UserViewResource (untuk detail):
  - + phone, created_at, updated_at, last_login_at
  - + relationships (tenants, workspaces, roles)
- [ ] UserReference entity
- [ ] RefreshToken entity

#### Step 5.2: Auth Service Implementation
- [ ] UserService dengan methods:
  - `find_by_email(email: &str) -> Result<User>`
  - `find_by_username(username: &str) -> Result<User>`
  - `verify_password(user: &User, password: &str) -> bool`
  - `create_refresh_token(user: &User) -> Result<RefreshToken>`
  - `validate_refresh_token(token: &str) -> Result<RefreshToken>`
  - `invalidate_refresh_token(token: &str) -> Result<()>`
  - `update_last_login(user: &User) -> Result<()>`
- [ ] AuthService dengan methods:
  - `login(username: &str, password: &str) -> Result<AuthResponse>`
  - `refresh(refresh_token: &str) -> Result<AuthResponse>`
  - `logout(refresh_token: &str) -> Result<()>`
  - `get_current_user(user_id: Uuid) -> Result<User>`
- [ ] Use dependency injection via traits

#### Step 5.3: Auth Controller & Routes
**POST /api/v1/auth/login:**
- [ ] Validate request (username/email + password)
- [ ] Call AuthService.login()
- [ ] Generate access_token (JWT, expired 15 min)
- [ ] Generate refresh_token (expired 7 days)
- [ ] Store refresh_token ke database
- [ ] Return response:
  ```json
  {
    "access_token": "...",
    "refresh_token": "...",
    "expires_in": 900,
    "token_type": "Bearer",
    "user": { UserShowResource }
  }
  ```

**POST /api/v1/auth/refresh:**
- [ ] Validate refresh_token dari request body
- [ ] Call AuthService.refresh()
- [ ] Generate new access_token
- [ ] Optionally rotate refresh_token (best practice)
- [ ] Return new tokens

**POST /api/v1/auth/logout:**
- [ ] Require authentication (valid access_token)
- [ ] Invalidate refresh_token dari request body
- [ ] Return success response

**GET /api/v1/auth/me:**
- [ ] Require authentication
- [ ] Get user_id dari JWT claims
- [ ] Call AuthService.get_current_user()
- [ ] Return user dengan ShowResource
- [ ] Include tenant context dari JWT

#### Step 5.4: Auth Middleware Implementation
- [ ] Extract token dari Authorization header: "Bearer {token}"
- [ ] Verify JWT signature dengan secret: "YXYlGIbJ65VGjQnETWX23iCvssXg7PJu"
- [ ] Check token expiry
- [ ] Extract claims: user_id, tenant_id, roles, permissions
- [ ] Set user context ke request extension
- [ ] Extract tenant_id dan setup database connection
- [ ] Call ConnectionManager.switch_to_tenant(tenant_id)
- [ ] Inject tenant context ke request
- [ ] Handle errors: 401 Unauthorized, 403 Forbidden

#### Step 5.5: Testing Auth Flow
- [ ] Unit tests untuk AuthService
- [ ] Unit tests untuk JWT utilities
- [ ] Integration tests:
  - [ ] Test login dengan valid credentials
  - [ ] Test login dengan invalid credentials
  - [ ] Test token refresh
  - [ ] Test token expiry
  - [ ] Test logout
  - [ ] Test protected endpoints dengan valid token
  - [ ] Test protected endpoints tanpa token
  - [ ] Test multi-tenant isolation (user A cannot access tenant B data)

### Phase 6: Frontend dengan Nuxt 4 & BFF (Minggu 5)

#### Step 6.1: Initialize Nuxt 4 Project (pantheon-nuxt)
- [ ] Create new Nuxt 4 project: `npx nuxi init pantheon-nuxt`
- [ ] Install dependencies:
  - primevue@^3.50.0
  - primeicons@^7.0.0
  - @iconify/vue@^4.1.1
  - pinia@^2.1.7
  - @pinia/nuxt@^0.5.1
- [ ] Configure nuxt.config.ts:
  - Enable SSR mode
  - Configure PrimeVue module
  - Configure Pinia module
  - Setup runtime config untuk API base URL
  - Prepare PWA config (disabled for now, akan dienable di Plan 5)
- [ ] Setup TypeScript dengan strict mode
- [ ] Create .env.example:
  ```
  NUXT_PUBLIC_API_URL=http://localhost:8000
  NUXT_API_SECRET=your-secret-here
  ```

#### Step 6.2: PrimeVue Configuration
- [ ] Install PrimeVue dan PrimeIcons
- [ ] Create `plugins/primevue.ts`:
  - Import PrimeVue
  - Import theme CSS dari primevue.org
  - Register global components (Button, InputText, DataTable, Dialog, etc.)
  - Configure PrimeVue options
- [ ] Create `assets/styles/primevue-theme.css`:
  - Custom CSS variables untuk theming
  - Override PrimeVue default styles
  - Add brand colors
- [ ] Create `plugins/iconify.ts`:
  - Setup Iconify
  - Configure icon collections

#### Step 6.3: BFF (Backend for Frontend) Server Implementation
**Server API Routes:**
- [ ] Create `server/api/auth/login.post.ts`:
  - Accept { username, password }
  - Forward request ke Rust BE: POST /api/v1/auth/login
  - Store tokens di httpOnly cookies (security best practice)
  - Return user data tanpa expose tokens ke client
- [ ] Create `server/api/auth/logout.post.ts`:
  - Read refresh_token dari httpOnly cookie
  - Call Rust BE logout endpoint
  - Clear cookies
- [ ] Create `server/api/auth/refresh.post.ts`:
  - Read refresh_token dari cookie
  - Call Rust BE refresh endpoint
  - Update cookies dengan new tokens
- [ ] Create `server/api/auth/me.get.ts`:
  - Read access_token dari cookie
  - Call Rust BE: GET /api/v1/auth/me
  - Return user data
- [ ] Create `server/api/proxy/[...path].ts`:
  - Universal proxy untuk semua API calls
  - Auto-attach access_token dari cookie ke Authorization header
  - Forward request ke Rust BE
  - Handle token refresh on 401 Unauthorized
  - Return response ke client

**Server Middleware:**
- [ ] Create `server/middleware/auth.ts`:
  - Validate cookies exist
  - Check token expiry (optional, bisa di-skip karena akan di-check di BE)
- [ ] Create `server/middleware/cors.ts`:
  - Setup CORS headers untuk development

**Server Utils:**
- [ ] Create `server/utils/jwt.ts`:
  - `extractTokenFromCookie()`
  - `setTokenCookies()` - set httpOnly, secure cookies
  - `clearTokenCookies()`
- [ ] Create `server/utils/api-client.ts`:
  - HTTP client untuk call Rust BE
  - Base URL dari runtime config
  - Error handling
  - Retry logic
- [ ] Create `server/utils/response.ts`:
  - Response formatter helpers
  - Error response helpers

#### Step 6.4: Pinia Stores
**Auth Store (`stores/auth.ts`):**
- [ ] State:
  - user: User | null
  - isAuthenticated: boolean
  - loading: boolean
- [ ] Getters:
  - currentUser()
  - hasRole(role: string)
  - hasPermission(permission: string)
- [ ] Actions:
  - login(username, password)
  - logout()
  - fetchCurrentUser()
  - refreshUser()
- [ ] Use BFF API routes (tidak langsung ke Rust BE)

**Tenant Store (`stores/tenant.ts`):**
- [ ] State:
  - currentTenant: Tenant | null
  - availableTenants: Tenant[]
  - currentWorkspace: Workspace | null
- [ ] Actions:
  - switchTenant(tenantId)
  - switchWorkspace(workspaceId)
  - fetchTenants()

**App Store (`stores/app.ts`):**
- [ ] State:
  - sidebarOpen: boolean
  - breadcrumbs: BreadcrumbItem[]
  - notifications: Notification[]
- [ ] Actions:
  - toggleSidebar()
  - setBreadcrumbs()
  - addNotification()

#### Step 6.5: Composables
**useAuth (`composables/useAuth.ts`):**
- [ ] Wrapper around auth store
- [ ] Helper methods: login(), logout(), isLoggedIn()
- [ ] Auto-fetch user on mount

**useApi (`composables/useApi.ts`):**
- [ ] HTTP client wrapper
- [ ] Call BFF proxy: `/api/proxy/{path}`
- [ ] Error handling dengan toast notifications
- [ ] Loading state management

**useTenant (`composables/useTenant.ts`):**
- [ ] Tenant context management
- [ ] Workspace switching

**useToast (`composables/useToast.ts`):**
- [ ] Toast notification wrapper untuk PrimeVue Toast
- [ ] success(), error(), info(), warning() methods

#### Step 6.6: Layouts
**layouts/default.vue:**
- [ ] Main layout dengan sidebar dan topbar
- [ ] Responsive design
- [ ] Use PrimeVue components

**layouts/auth.vue:**
- [ ] Clean layout untuk login/register pages
- [ ] Centered card design

**layouts/dashboard.vue:**
- [ ] Dashboard layout dengan enhanced sidebar
- [ ] Breadcrumbs
- [ ] User menu dropdown

#### Step 6.7: Pages
**pages/index.vue:**
- [ ] Landing page (redirect ke /dashboard atau /login)

**pages/login.vue:**
- [ ] Login form dengan PrimeVue InputText, Password, Button
- [ ] Form validation
- [ ] Error handling dengan PrimeVue Toast
- [ ] Remember me checkbox (optional)
- [ ] Redirect ke /dashboard after successful login

**pages/dashboard/index.vue:**
- [ ] Dashboard home page
- [ ] Summary cards dengan statistics
- [ ] Charts (akan diimplementasi di Plan 2)
- [ ] Recent activities

**pages/dashboard/settings/index.vue:**
- [ ] User settings page
- [ ] Profile management
- [ ] Password change

#### Step 6.8: Middleware
**middleware/auth.ts:**
- [ ] Check authentication status
- [ ] Redirect ke /login jika tidak authenticated
- [ ] Fetch current user jika belum ada di store

**middleware/tenant.ts:**
- [ ] Validate tenant context exists
- [ ] Redirect jika tenant tidak valid

#### Step 6.9: Components
**components/auth/LoginForm.vue:**
- [ ] Form dengan PrimeVue components
- [ ] Validation dengan VeeValidate (optional) atau native
- [ ] Loading state

**components/layout/AppTopbar.vue:**
- [ ] Logo
- [ ] Breadcrumbs
- [ ] User menu dengan dropdown
- [ ] Notifications icon
- [ ] Tenant/Workspace selector

**components/layout/AppMenu.vue:**
- [ ] Sidebar menu
- [ ] Menu items dari routes
- [ ] Active state styling
- [ ] Collapsible submenu

**components/common/DataTable.vue:**
- [ ] Wrapper untuk PrimeVue DataTable
- [ ] Default pagination
- [ ] Default sorting
- [ ] Actions column
- [ ] Loading overlay

#### Step 6.10: Testing Frontend
- [ ] Test login flow
- [ ] Test logout flow
- [ ] Test protected routes
- [ ] Test BFF proxy
- [ ] Test token refresh
- [ ] Test responsive design
- [ ] Test with multiple browsers

### Phase 7: pantheon-docs Application (Minggu 5)

#### Step 7.1: Setup pantheon-docs Project
- [ ] Create new Nuxt 4 project: `pantheon-docs/`
- [ ] Install swagger-ui dependencies:
  - swagger-ui-dist
  - @nuxtjs/tailwindcss
- [ ] Configure nuxt.config.ts untuk static generation (SSG)
- [ ] Create basic README

#### Step 7.2: Swagger UI Integration
**pages/index.vue:**
- [ ] Implement Swagger UI component
- [ ] Load openapi.json dari public folder
- [ ] Configure Swagger UI options:
  - deepLinking: true
  - displayRequestDuration: true
  - docExpansion: 'list'
  - filter: true

**public/openapi.json:**
- [ ] This file akan di-generate dari pantheon-rust
- [ ] Manual create initial version untuk development
- [ ] Include API version, title, description

#### Step 7.3: API Documentation Pages
**pages/api-reference.vue:**
- [ ] List of all API endpoints
- [ ] Grouped by module
- [ ] Links to Swagger UI

**pages/guides/authentication.vue:**
- [ ] How to authenticate
- [ ] JWT token flow
- [ ] Token refresh

**pages/guides/multi-tenant.vue:**
- [ ] How multi-tenant works
- [ ] Tenant context
- [ ] Database architecture

#### Step 7.4: Plans Management
- [ ] Move existing plans ke pantheon-docs/plans/
- [ ] Create index page untuk plans
- [ ] Link plans to documentation

#### Step 7.5: Docker Setup untuk pantheon-docs
- [ ] Create Dockerfile untuk static serving
- [ ] Add ke docker-compose.yml
- [ ] Expose on port 3001

### Phase 8: Docker Infrastructure (Minggu 6)

#### Step 8.1: Backend Dockerfile (pantheon-app)
**Multi-stage Build:**
- [ ] Stage 1 - Builder:
  - Use rust:1.75-bookworm as base
  - Install build dependencies
  - Copy Cargo.toml workspace
  - Copy all crate Cargo.tomls
  - Build dependencies (cache layer)
  - Copy source code
  - Build release binary
- [ ] Stage 2 - Runtime:
  - Use debian:bookworm-slim
  - Install runtime dependencies (libpq, ca-certificates)
  - Copy binary dari builder stage
  - Copy config files
  - Expose port 8000
  - Set entrypoint
- [ ] Optimize image size (target: < 100MB)

#### Step 8.2: Frontend Dockerfile (pantheon-nuxt)
**Multi-stage Build:**
- [ ] Stage 1 - Builder:
  - Use node:20-alpine
  - Copy package.json, package-lock.json
  - npm install
  - Copy source code
  - npm run build
- [ ] Stage 2 - Runtime:
  - Use node:20-alpine
  - Copy built files dari builder
  - Expose port 3000
  - Set entrypoint: node .output/server/index.mjs
- [ ] Optimize image size

#### Step 8.3: pantheon-docs Dockerfile
- [ ] Build Nuxt app as static
- [ ] Use nginx:alpine untuk serving
- [ ] Copy dist files
- [ ] Configure nginx.conf
- [ ] Expose port 3001

#### Step 8.4: Docker Compose Configuration

**Location:** `pantheon-app/docker/docker-compose.yml` (atau root project folder untuk development)

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  postgres:
    - PostgreSQL 15
    - Ports: 5432
    - Volumes: pgdata, init scripts
    - Environment: credentials, databases
    - Health check

  rabbitmq:
    - RabbitMQ 3.12 with management
    - Ports: 5672 (AMQP), 15672 (Management UI)
    - Volumes: rabbit data
    - Environment: credentials
    - Health check

  pantheon-app:
    - Depends on: postgres, rabbitmq
    - Ports: 8000
    - Environment: database URLs, JWT secret, RabbitMQ URL
    - Volumes: config overrides
    - Restart: unless-stopped
    - Health check

  pantheon-nuxt:
    - Depends on: pantheon-app
    - Ports: 3000
    - Environment: API URL
    - Restart: unless-stopped

  pantheon-docs:
    - Ports: 3001
    - Volumes: openapi.json
    - Restart: unless-stopped
```

**Networks:**
- [ ] Create `pantheon-network` untuk inter-service communication

**Volumes:**
- [ ] `pgdata` - PostgreSQL data persistence
- [ ] `rabbitmq-data` - RabbitMQ persistence

**Environment Files:**
- [ ] Create `.env.example` dengan semua environment variables
- [ ] Create `.env.development`, `.env.staging`, `.env.production`

#### Step 8.5: Docker Compose Variants
- [ ] `docker-compose.yml` - Base configuration
- [ ] `docker-compose.dev.yml` - Development overrides (hot reload, debug)
- [ ] `docker-compose.prod.yml` - Production overrides (optimizations)

#### Step 8.6: Database Initialization Scripts
- [ ] Create `docker/postgres/init/` folder
- [ ] `01-create-databases.sql`:
  - Create pantheon database
  - Create pantheon_tenant_template database
  - Create test tenant database: pantheon_tenant_00000000-0000-0000-0000-000000000001
- [ ] `02-create-users.sql`:
  - Create pantheon_admin user
  - Create pantheon_app user
  - Grant permissions

#### Step 8.7: Testing Docker Setup
- [ ] Test `docker-compose up` semua services start
- [ ] Test database connections
- [ ] Test RabbitMQ connection
- [ ] Test Rust BE health endpoint
- [ ] Test Frontend dapat akses BE
- [ ] Test pantheon-docs dapat serve Swagger UI
- [ ] Test volume persistence (stop & restart services)
- [ ] Test service restart policies

### Phase 9: RabbitMQ Integration (Minggu 6)

#### Step 9.1: RabbitMQ Configuration
- [ ] Add RabbitMQ to docker-compose (already done in Phase 8)
- [ ] Create `rust-support/src/queue/` module:
  - connection.rs - RabbitMQ connection manager
  - publisher.rs - Job publisher
  - consumer.rs - Job consumer/worker
  - job.rs - Job trait dan base implementation

#### Step 9.2: Job Queue Framework
**Job Trait:**
- [ ] Define Job trait:
  ```rust
  pub trait Job: Send + Sync {
      fn handle(&self) -> Result<()>;
      fn queue_name(&self) -> &str;
      fn max_retries(&self) -> u32;
      fn retry_delay(&self) -> Duration;
  }
  ```
- [ ] JobPayload struct dengan serialization
- [ ] JobStatus enum (Pending, Processing, Completed, Failed)

**Queue Manager:**
- [ ] QueueManager untuk publishing jobs
  - `dispatch<T: Job>(job: T)` - publish job to queue
  - `dispatch_delayed<T: Job>(job: T, delay: Duration)` - schedule job
- [ ] Worker untuk consuming jobs
  - Listen pada specific queue
  - Deserialize payload
  - Execute job.handle()
  - Handle success: ack message
  - Handle failure: retry dengan exponential backoff atau move to DLQ
  - Track attempts di database (jobs table)

**Error Handling:**
- [ ] Retry logic dengan exponential backoff
- [ ] Dead Letter Queue (DLQ) untuk failed jobs
- [ ] Move ke failed_jobs table after max retries
- [ ] Logging untuk debugging

#### Step 9.3: Background Job Examples
**Example Jobs:**
- [ ] SendEmailJob - untuk testing
  - Payload: { to, subject, body }
  - Simulate email sending dengan delay
- [ ] ClusterSchemaGeneratorJob - untuk auto-generating cluster schemas (akan fully implemented di Plan 3)
  - Payload: { tenant_id, cluster_type, year }
  - Check if schema exists
  - Create schema jika belum ada
  - Run migrations
- [ ] DataExportJob - untuk exporting data
  - Payload: { tenant_id, export_type, filters }
  - Generate CSV/Excel file
  - Store di storage
  - Notify user

#### Step 9.4: Job Management API
**Endpoints:**
- [ ] GET /api/v1/jobs - List jobs (paginated)
- [ ] GET /api/v1/jobs/:id - Get job details
- [ ] POST /api/v1/jobs/:id/retry - Retry failed job
- [ ] DELETE /api/v1/jobs/:id - Delete job

**Dashboard UI (basic):**
- [ ] Jobs list page
- [ ] Job status badges
- [ ] Retry button untuk failed jobs

#### Step 9.5: Testing Background Jobs
- [ ] Unit tests untuk Job implementations
- [ ] Integration tests:
  - [ ] Publish job dan verify received di worker
  - [ ] Test retry mechanism
  - [ ] Test DLQ functionality
  - [ ] Test job failure handling
  - [ ] Test concurrent job processing

### Phase 10: Swagger/OpenAPI Documentation (Minggu 6)

#### Step 10.1: Setup utoipa in Rust
**Dependencies:**
- [ ] Add to Cargo.toml:
  - utoipa = { version = "4", features = ["actix_extras"] }
  - utoipa-swagger-ui = { version = "6", features = ["actix-web"] }

**OpenAPI Configuration:**
- [ ] Create `rust-support/src/docs/mod.rs`:
  - OpenAPI configuration
  - Info, servers, security schemes
  - Tags untuk grouping endpoints

#### Step 10.2: Annotate APIs dengan utoipa Macros
**Auth Endpoints:**
- [ ] Annotate AuthController methods:
  - `#[utoipa::path(...)]` untuk setiap endpoint
  - Define request body schemas
  - Define response schemas
  - Define error responses
  - Add examples

**Example:**
```rust
#[utoipa::path(
    post,
    path = "/api/v1/auth/login",
    request_body = LoginRequest,
    responses(
        (status = 200, description = "Login successful", body = AuthResponse),
        (status = 401, description = "Invalid credentials", body = ErrorResponse)
    ),
    tag = "Authentication"
)]
async fn login(...)
```

**DTOs:**
- [ ] Annotate request/response structs dengan `#[derive(ToSchema)]`
- [ ] Add descriptions dan examples

#### Step 10.3: Generate openapi.json
- [ ] Add endpoint `/api/openapi.json` untuk serving OpenAPI spec
- [ ] Generate spec dari annotated APIs
- [ ] Include all schemas, paths, components
- [ ] Validate generated spec

**Automated Generation:**
- [ ] Create build script untuk generating openapi.json file
- [ ] Copy generated file ke pantheon-docs/public/
- [ ] Add to CI/CD pipeline

#### Step 10.4: Document All Core APIs
**Auth Module:**
- [ ] POST /api/v1/auth/login
- [ ] POST /api/v1/auth/logout
- [ ] POST /api/v1/auth/refresh
- [ ] GET /api/v1/auth/me

**Future Modules (documented but implemented in Plan 2):**
- [ ] Users API
- [ ] Tenants API
- [ ] Workspaces API
- [ ] Items API
- [ ] Transactions API

#### Step 10.5: Swagger UI Enhancements
**In pantheon-docs:**
- [ ] Add authentication to Swagger UI:
  - Support Bearer token input
  - Persist token in localStorage
  - Auto-attach to requests
- [ ] Add "Try it out" functionality
- [ ] Add code examples (curl, JavaScript, Python)
- [ ] Add response examples

### Phase 11: CI/CD with GitHub Actions (Minggu 7)

#### Step 11.1: GitHub Repository Setup
- [ ] Create GitHub organization or use personal account
- [ ] Create repository: `pantheon`
- [ ] Setup branches:
  - `main` - Production
  - `staging` - Staging environment
  - `develop` - Development environment
- [ ] Branch protection rules:
  - main: Require PR, require status checks, require reviews
  - staging: Require PR, require status checks
  - develop: Require status checks only

#### Step 11.2: Development Workflow (.github/workflows/development.yml)
**Trigger:** Push to `develop` branch
**Jobs:**
1. **Lint Rust:**
   - [ ] Run cargo clippy
   - [ ] Run cargo fmt --check
2. **Test Rust:**
   - [ ] Run cargo test --all
   - [ ] Generate coverage report (optional: cargo-tarpaulin)
3. **Build Rust:**
   - [ ] Run cargo build --release
   - [ ] Cache dependencies
4. **Lint Frontend:**
   - [ ] Run npm run lint
   - [ ] Run npm run type-check
5. **Test Frontend:**
   - [ ] Run npm run test (if tests exist)
6. **Build Frontend:**
   - [ ] Run npm run build
7. **Build Docker Images:**
   - [ ] Build pantheon-rust image
   - [ ] Build pantheon-nuxt image
   - [ ] Tag with commit SHA
   - [ ] Push to Docker registry (optional)
8. **Deploy to Development (optional):**
   - [ ] Deploy to development server
   - [ ] Run health checks

#### Step 11.3: Staging Workflow (.github/workflows/staging.yml)
**Trigger:** Push to `staging` branch atau PR merge ke staging
**Jobs:**
1. **Run Development Workflow Jobs**
2. **Integration Tests:**
   - [ ] Start services dengan docker-compose
   - [ ] Run API integration tests
   - [ ] Run E2E tests (jika ada)
   - [ ] Stop services
3. **Security Scan:**
   - [ ] Run cargo-audit untuk Rust dependencies
   - [ ] Run npm audit untuk Node dependencies
   - [ ] Run container security scan (Trivy)
4. **Deploy to Staging:**
   - [ ] Build and push Docker images dengan tag `staging-{sha}`
   - [ ] Deploy to staging server (placeholder)
   - [ ] Run smoke tests
   - [ ] Notify team on Slack/Discord (optional)

#### Step 11.4: Production Workflow (.github/workflows/production.yml)
**Trigger:**
- Push to `main` branch
- Manual workflow dispatch dengan approval

**Jobs:**
1. **Run Staging Workflow Jobs**
2. **Full Test Suite:**
   - [ ] All unit tests
   - [ ] All integration tests
   - [ ] Performance tests (basic)
3. **Security Checks:**
   - [ ] SAST (Static Application Security Testing)
   - [ ] Dependency vulnerability scan
   - [ ] Container security scan
4. **Build Release:**
   - [ ] Build optimized Rust binary
   - [ ] Build optimized frontend
   - [ ] Generate openapi.json
   - [ ] Build all Docker images
   - [ ] Tag dengan version number dan `latest`
5. **Deploy to Production (placeholder):**
   - [ ] Manual approval required
   - [ ] Deploy to production servers
   - [ ] Run health checks
   - [ ] Run smoke tests
   - [ ] Rollback on failure
6. **Post-Deploy:**
   - [ ] Create GitHub release
   - [ ] Tag commit dengan version
   - [ ] Notify team
   - [ ] Update documentation

#### Step 11.5: Reusable Workflows & Actions
- [ ] Create reusable workflow untuk Rust tests
- [ ] Create reusable workflow untuk Frontend build
- [ ] Create composite action untuk Docker build
- [ ] Setup caching untuk dependencies (Cargo, npm)

#### Step 11.6: Secrets Configuration
**Required Secrets:**
- [ ] DOCKER_USERNAME
- [ ] DOCKER_PASSWORD
- [ ] DATABASE_URL (untuk integration tests)
- [ ] JWT_SECRET
- [ ] STAGING_SERVER_SSH_KEY (jika autodeploy)
- [ ] PRODUCTION_SERVER_SSH_KEY (jika autodeploy)
- [ ] SLACK_WEBHOOK (optional untuk notifications)

#### Step 11.7: Testing CI/CD
- [ ] Test development workflow dengan dummy commit
- [ ] Test branch protection rules
- [ ] Test PR workflow
- [ ] Test staging workflow
- [ ] Test production workflow (dry-run mode)
- [ ] Verify notifications
- [ ] Verify artifacts uploaded

### Phase 12: Comprehensive Testing (Minggu 7-8)

#### Step 12.1: Unit Testing
**Rust Backend:**
- [ ] Test coverage target: > 80% untuk rust-support
- [ ] Test coverage target: > 70% untuk repositories
- [ ] Test BaseEntity implementations
- [ ] Test Resource transformations
- [ ] Test Service logic
- [ ] Test JWT utilities
- [ ] Test Database Manager
- [ ] Test Connection Manager
- [ ] Test Configuration Merging
- [ ] Test Service Provider loading

**Frontend:**
- [ ] Test stores (Pinia)
- [ ] Test composables
- [ ] Test utility functions
- [ ] Test coverage target: > 60%

#### Step 12.2: Integration Testing
**API Integration Tests:**
- [ ] Auth flow tests:
  - [ ] Login dengan valid credentials
  - [ ] Login dengan invalid credentials
  - [ ] Token refresh flow
  - [ ] Logout flow
  - [ ] Access protected endpoint dengan valid token
  - [ ] Access protected endpoint dengan invalid token
  - [ ] Access protected endpoint dengan expired token
  - [ ] Token refresh on 401
- [ ] Multi-tenant tests:
  - [ ] User dari tenant A tidak bisa akses data tenant B
  - [ ] Connection switching berdasarkan tenant
  - [ ] Tenant isolation di database level
- [ ] Database tests:
  - [ ] Connection pooling
  - [ ] Transaction rollback
  - [ ] Migration up/down
  - [ ] Tenant database creation

**Frontend Integration Tests:**
- [ ] Login flow (UI + API)
- [ ] Dashboard navigation
- [ ] API calls melalui BFF
- [ ] Token refresh automatic

#### Step 12.3: End-to-End Testing
**Scenarios:**
- [ ] Complete user journey:
  1. User membuka app
  2. User login
  3. User navigate ke dashboard
  4. User view data (contoh: items list)
  5. User create new item
  6. User logout
- [ ] Multi-tenant scenario:
  1. Admin create tenant
  2. Admin create user untuk tenant
  3. User login ke tenant
  4. User can only see tenant data
- [ ] Error scenarios:
  1. Network error
  2. Server error
  3. Validation error
  4. Permission denied

**Tools:**
- [ ] Use Playwright atau Cypress untuk E2E tests (optional untuk Phase 1)

#### Step 12.4: Performance Testing
**Load Testing:**
- [ ] Setup load testing tool (k6 atau Apache JMeter)
- [ ] Test scenarios:
  - [ ] Login endpoint: 100 concurrent users
  - [ ] List API: 50 concurrent users
  - [ ] Create API: 20 concurrent users
- [ ] Measure:
  - Response time (p50, p95, p99)
  - Throughput (requests/second)
  - Error rate
  - Database connection pool usage
- [ ] Performance targets:
  - Login: < 500ms (p95)
  - List API: < 200ms (p95)
  - Create API: < 300ms (p95)

**Database Performance:**
- [ ] Query performance testing
- [ ] Index effectiveness
- [ ] Connection pool sizing
- [ ] Slow query logging

#### Step 12.5: Security Testing
**Security Checks:**
- [ ] SQL Injection testing (should be prevented by ORM)
- [ ] XSS testing (should be prevented by frontend framework)
- [ ] CSRF testing (verify CSRF tokens jika applicable)
- [ ] JWT token security:
  - [ ] Token cannot be tampered
  - [ ] Token expiry enforced
  - [ ] Refresh token rotation
- [ ] API authorization:
  - [ ] Endpoints require authentication
  - [ ] Permission checks working
  - [ ] Tenant isolation enforced
- [ ] Dependency vulnerability scan (cargo-audit, npm audit)
- [ ] Container security scan (Trivy)

#### Step 12.6: Test Documentation
- [ ] Document test scenarios
- [ ] Document test data setup
- [ ] Document how to run tests
- [ ] Document test coverage reports
- [ ] Add CI test status badges to README

### Phase 13: Documentation & Knowledge Transfer (Minggu 8)

#### Step 13.1: Technical Documentation
**Architecture Documentation:**
- [ ] System architecture diagram
- [ ] Database architecture diagram
- [ ] Multi-tenant architecture explanation
- [ ] Service provider loading flow diagram
- [ ] Configuration hierarchy diagram
- [ ] Connection mapping flow
- [ ] Authentication flow diagram
- [ ] BFF architecture diagram

**Code Documentation:**
- [ ] Rust doc comments untuk public APIs
- [ ] Generate rustdoc: `cargo doc --no-deps --open`
- [ ] JSDoc comments untuk composables dan utils
- [ ] Component props documentation

**API Documentation:**
- [ ] OpenAPI/Swagger documentation (already done in Phase 10)
- [ ] API usage examples
- [ ] Authentication guide
- [ ] Error handling guide
- [ ] Rate limiting guide (untuk future implementation)

#### Step 13.2: Setup & Development Guides
**README.md (Root):**
- [ ] Project overview
- [ ] Tech stack
- [ ] Prerequisites
- [ ] Quick start guide
- [ ] Docker setup instructions
- [ ] Links to detailed docs

**SETUP.md:**
- [ ] Development environment setup
  - Install Rust
  - Install Node.js
  - Install PostgreSQL
  - Install RabbitMQ
  - Install Docker
- [ ] Clone repository
- [ ] Environment configuration
- [ ] Database setup
- [ ] Run migrations
- [ ] Seed data
- [ ] Start services

**DEVELOPMENT.md:**
- [ ] Project structure explanation
- [ ] How to create new module
- [ ] How to create entity
- [ ] How to create migration
- [ ] How to add service provider
- [ ] How to configure entity connection
- [ ] How to test locally
- [ ] How to debug
- [ ] Code style guidelines
- [ ] Git workflow

**DEPLOYMENT.md (placeholder untuk Plan 7):**
- [ ] Deployment strategy overview
- [ ] Environment configuration
- [ ] CI/CD pipeline explanation
- [ ] Rollback procedures
- [ ] Monitoring setup
- [ ] Backup procedures

#### Step 13.3: User Documentation (Basic)
**pantheon-docs/docs/:**
- [ ] Getting Started guide
- [ ] User authentication
- [ ] Multi-tenant concept
- [ ] FAQ
- [ ] Troubleshooting guide

#### Step 13.4: Runbook & Operational Docs
**RUNBOOK.md:**
- [ ] Common operations:
  - Create new tenant
  - Reset user password
  - View logs
  - Restart services
  - Database backup/restore
- [ ] Troubleshooting:
  - Service not starting
  - Database connection issues
  - RabbitMQ connection issues
  - High memory/CPU usage
- [ ] Monitoring:
  - Health check endpoints
  - Metrics to monitor
  - Alert thresholds

#### Step 13.5: Update Existing Docs
- [ ] Update this plan (1-FOUNDATION-INFRASTRUCTURE-PLAN.md) dengan lessons learned
- [ ] Create Plan 2 template based on experience
- [ ] Update project README files
- [ ] Add contribution guidelines

### Phase 14: Review, Refinement & Handover (Minggu 8)

#### Step 14.1: Code Review & Quality
**Code Review:**
- [ ] Review all Rust code untuk:
  - Code organization
  - Error handling
  - Documentation
  - Performance
  - Security
  - Best practices
- [ ] Review Frontend code untuk:
  - Component structure
  - State management
  - Code duplication
  - Performance
  - Accessibility (basic)

**Code Quality Tools:**
- [ ] Run cargo clippy dan fix warnings
- [ ] Run cargo fmt
- [ ] Run eslint dan fix issues
- [ ] Check test coverage
- [ ] Generate dan review code complexity reports

**Refactoring:**
- [ ] Refactor duplicate code
- [ ] Simplify complex functions
- [ ] Improve naming
- [ ] Add missing error handling
- [ ] Add missing documentation

#### Step 14.2: Performance Optimization
**Backend:**
- [ ] Review API response times
- [ ] Optimize slow queries
- [ ] Add database indexes jika perlu
- [ ] Tune connection pool settings
- [ ] Optimize Rust binary size
- [ ] Enable Rust compile-time optimizations

**Frontend:**
- [ ] Optimize bundle size
- [ ] Lazy load components
- [ ] Optimize images (jika ada)
- [ ] Add caching strategies
- [ ] Review Nuxt configuration

**Database:**
- [ ] Review query execution plans
- [ ] Add missing indexes
- [ ] Optimize connection pool per database
- [ ] Tune PostgreSQL configuration

#### Step 14.3: Security Hardening
**Security Checklist:**
- [ ] Review authentication implementation
- [ ] Review authorization implementation
- [ ] Check for sensitive data exposure
- [ ] Verify HTTPS enforcement (untuk production)
- [ ] httpOnly cookies untuk tokens
- [ ] Secure headers (CORS, CSP, X-Frame-Options)
- [ ] Rate limiting (basic implementation)
- [ ] Input validation dan sanitization
- [ ] SQL injection prevention (verified)
- [ ] XSS prevention (verified)
- [ ] Dependency vulnerability scan
- [ ] Secrets management (not in code)
- [ ] Environment-specific configs

**Penetration Testing (Basic):**
- [ ] Test authentication bypass
- [ ] Test authorization bypass
- [ ] Test tenant isolation
- [ ] Test injection attacks
- [ ] Test session management

#### Step 14.4: Monitoring & Observability Setup (Basic)
**Logging:**
- [ ] Structured logging in Rust (tracing crate)
- [ ] Log levels configuration
- [ ] Request logging (request ID, duration, status)
- [ ] Error logging dengan stack traces
- [ ] Audit logging (who did what)

**Health Checks:**
- [ ] Backend health endpoint: GET /health
  - Check database connection
  - Check RabbitMQ connection
  - Return status + dependencies status
- [ ] Frontend health endpoint
- [ ] Database health check
- [ ] RabbitMQ health check

**Metrics (Basic Setup):**
- [ ] Request count
- [ ] Response time
- [ ] Error rate
- [ ] Database query time
- [ ] Connection pool usage

**Future:** Full observability dengan Prometheus + Grafana akan di Plan 7

#### Step 14.5: Final Testing & Validation
**Acceptance Testing:**
- [ ] Verify all requirements dari PANTHEON APP REQUIREMENT.md:
  - [ ] Rust BE working
  - [ ] Nuxt 4 FE with BFF working
  - [ ] JWT authentication (HS256) working
  - [ ] Multi-tenant database working
  - [ ] Entity-resource-controller-service pattern implemented
  - [ ] Service provider pattern working
  - [ ] Configuration hierarchy working
  - [ ] All core tables created
  - [ ] All tenant tables created
  - [ ] Cluster schemas setup (manual for now)
  - [ ] RabbitMQ integration working
  - [ ] Swagger documentation available
  - [ ] Docker setup working
  - [ ] CI/CD pipelines working
  - [ ] All repository modules migrated (except excluded ones)

**Regression Testing:**
- [ ] Run all unit tests
- [ ] Run all integration tests
- [ ] Run all E2E tests
- [ ] Verify all tests passing

**Production Readiness Checklist:**
- [ ] All critical features working
- [ ] All tests passing
- [ ] Documentation complete
- [ ] Security reviewed
- [ ] Performance acceptable
- [ ] Monitoring setup
- [ ] Backup strategy defined
- [ ] Rollback plan defined

#### Step 14.6: Handover & Knowledge Transfer
**Team Training:**
- [ ] Walkthrough sistem architecture
- [ ] Walkthrough code structure
- [ ] Demo key features
- [ ] Explain multi-tenant architecture
- [ ] Explain service provider pattern
- [ ] Explain configuration system
- [ ] Show how to add new features
- [ ] Show how to debug issues
- [ ] Q&A session

**Documentation Review:**
- [ ] Ensure all documentation up-to-date
- [ ] Ensure runbook is clear
- [ ] Ensure setup guide works (test with fresh environment)

**Transition to Plan 2:**
- [ ] Review Plan 2 scope
- [ ] Identify dependencies
- [ ] Identify risks
- [ ] Estimate timeline
- [ ] Allocate resources

## Files to Create/Modify

### New GitHub Repositories (under pantheon organization)
1. **pantheon/pantheon-app** - Backend Rust workspace repository
   - URL: https://github.com/pantheon/pantheon-app
   - Alternative: hanafalah/pantheon-app jika organization tidak tersedia
2. **pantheon/pantheon-nuxt** - Frontend repository
   - URL: https://github.com/pantheon/pantheon-nuxt
   - Alternative: hanafalah/pantheon-nuxt
3. **pantheon/pantheon-docs** - Documentation repository
   - URL: https://github.com/pantheon/pantheon-docs
   - Alternative: hanafalah/pantheon-docs

### Configuration Files (pantheon-app)
1. `pantheon-app/Cargo.toml` (workspace root)
2. `pantheon-app/.env.example`
3. `pantheon-app/.gitignore`
4. `pantheon-app/README.md`
5. `pantheon-app/config/app.toml`
6. `pantheon-app/config/database.toml`
7. `pantheon-app/config/jwt.toml`
8. `pantheon-app/config/entity_connections.toml`
9. `pantheon-app/docker/Dockerfile`
10. `pantheon-app/docker/docker-compose.yml`
11. `pantheon-app/.github/workflows/development.yml`
12. `pantheon-app/.github/workflows/staging.yml`
13. `pantheon-app/.github/workflows/production.yml`

### Configuration Files (pantheon-nuxt)
1. `pantheon-nuxt/nuxt.config.ts`
2. `pantheon-nuxt/package.json`
3. `pantheon-nuxt/.env.example`
4. `pantheon-nuxt/tsconfig.json`
5. `pantheon-nuxt/docker/Dockerfile`
6. `pantheon-nuxt/.github/workflows/development.yml`

### Configuration Files (pantheon-docs)
1. `pantheon-docs/nuxt.config.ts`
2. `pantheon-docs/package.json`
3. `pantheon-docs/docker/Dockerfile`
4. `pantheon-docs/plans/1-FOUNDATION-INFRASTRUCTURE-PLAN.md` (this file)

### Core Implementation Files (pantheon-app/rust-support/)
1. `rust-support/Cargo.toml`
2. `rust-support/src/lib.rs`
3. `rust-support/src/provider/service_provider.rs`
4. `rust-support/src/base/entity.rs`
5. `rust-support/src/base/resource.rs`
6. `rust-support/src/base/controller.rs`
7. `rust-support/src/base/service.rs`
8. `rust-support/src/database/manager.rs`
9. `rust-support/src/database/connection.rs`
10. `rust-support/src/database/resolver.rs`
11. `rust-support/src/config/loader.rs`
12. `rust-support/src/config/merger.rs`
13. `rust-support/src/auth/jwt.rs`
14. `rust-support/src/auth/middleware.rs`
15. `rust-support/src/utils/response.rs`
16. `rust-support/src/utils/error.rs`

### Repository Module Files (pantheon-app/repositories/)
Example untuk module-user:
1. `repositories/module-user/Cargo.toml`
2. `repositories/module-user/src/lib.rs`
3. `repositories/module-user/src/provider.rs`
4. `repositories/module-user/src/entities/user.rs`
5. `repositories/module-user/src/entities/user_reference.rs`
6. `repositories/module-user/src/resources/user_view_resource.rs`
7. `repositories/module-user/src/resources/user_show_resource.rs`
8. `repositories/module-user/src/controllers/user_controller.rs`
9. `repositories/module-user/src/services/user_service.rs`

(Repeat similar structure untuk 35+ repositories)

### Project Files (pantheon-app/projects/pantheon-business/)
1. `projects/pantheon-business/Cargo.toml`
2. `projects/pantheon-business/src/main.rs`
3. `projects/pantheon-business/src/routes.rs`
4. `projects/pantheon-business/src/provider.rs`
5. `projects/pantheon-business/config/database.toml`
6. `projects/pantheon-business/config/entity_connections.toml`

### Database Migration Files (pantheon-app/migrations/)
1. `migrations/core/001_create_users_and_references.sql`
2. `migrations/core/002_create_tenants_and_workspaces.sql`
3. `migrations/core/003_create_licenses.sql`
4. `migrations/core/004_create_regional_tables.sql`
5. `migrations/core/005_create_utility_tables.sql`
6. `migrations/tenant/001_create_base_tables.sql`
7. `migrations/tenant/002_create_permission_tables.sql`
8. `migrations/tenant/003_create_item_tables.sql`
9. `migrations/tenant/004_create_stock_tables.sql`
10. `migrations/tenant/005_create_transaction_tables.sql`
11. `migrations/cluster/cashier/001_create_cashier_tables.sql`
12. `migrations/cluster/scm/001_create_scm_tables.sql`

### Frontend Files (pantheon-nuxt/)
1. `pages/index.vue`
2. `pages/login.vue`
3. `pages/dashboard/index.vue`
4. `layouts/default.vue`
5. `layouts/auth.vue`
6. `components/auth/LoginForm.vue`
7. `components/layout/AppTopbar.vue`
8. `components/layout/AppMenu.vue`
9. `composables/useAuth.ts`
10. `composables/useApi.ts`
11. `stores/auth.ts`
12. `stores/tenant.ts`
13. `server/api/auth/login.post.ts`
14. `server/api/auth/logout.post.ts`
15. `server/api/auth/refresh.post.ts`
16. `server/api/proxy/[...path].ts`
17. `server/utils/jwt.ts`
18. `server/utils/api-client.ts`
19. `middleware/auth.ts`
20. `plugins/primevue.ts`
21. `plugins/iconify.ts`

### Documentation Files (pantheon-docs/)
1. `pages/index.vue` (Swagger UI)
2. `pages/api-reference.vue`
3. `public/openapi.json` (generated dari pantheon-app)
4. `plans/1-FOUNDATION-INFRASTRUCTURE-PLAN.md` (existing, updated)
5. `plans/2-BUSINESS-MODULES-PLAN.md` (future)

### Documentation Files (Project Root)
1. `README.md` (overview linking ke semua repos)
2. `SETUP.md` (setup guide)
3. `DEVELOPMENT.md` (development guide)
4. `RUNBOOK.md` (operational guide)
5. `DEPLOYMENT.md` (deployment guide - placeholder)

## Dependencies

### Rust Dependencies

**Workspace Cargo.toml:**
```toml
[workspace]
members = [
    "rust-support",
    "repositories/*",
    "projects/*",
    "groups/*",
    "tenants/*",
]
resolver = "2"

[workspace.dependencies]
# Web Frameworks (Choose one)
actix-web = "4.4"
actix-cors = "0.7"
actix-files = "0.6"
# Alternative: axum = "0.7"

# Database
diesel = { version = "2.1", features = ["postgres", "uuid", "chrono", "r2d2"] }
diesel_migrations = "2.1"
# Alternative: sea-orm = { version = "0.12", features = ["sqlx-postgres", "runtime-tokio-native-tls", "macros"] }

# Connection Pooling
r2d2 = "0.8"
deadpool-postgres = "0.12" # Alternative pooling

# Async Runtime
tokio = { version = "1.35", features = ["full"] }
futures = "0.3"

# Serialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
toml = "0.8" # For config files

# JWT Authentication
jsonwebtoken = "9.2"

# Password Hashing
bcrypt = "0.15"
argon2 = "0.5" # Alternative, more secure

# UUID
uuid = { version = "1.6", features = ["v4", "serde"] }

# Date & Time
chrono = { version = "0.4", features = ["serde"] }

# Configuration Management
config = "0.14"
dotenv = "0.15"
envy = "0.4" # Environment variable parsing

# Validation
validator = { version = "0.18", features = ["derive"] }

# Error Handling
anyhow = "1.0"
thiserror = "1.0"

# Logging & Tracing
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
tracing-actix-web = "0.7"

# RabbitMQ / AMQP
lapin = "2.3"
async-trait = "0.1"

# OpenAPI / Swagger
utoipa = { version = "4.2", features = ["actix_extras", "chrono", "uuid"] }
utoipa-swagger-ui = { version = "6.0", features = ["actix-web"] }

# HTTP Client (for testing, BFF communication)
reqwest = { version = "0.11", features = ["json"] }

# Testing
mockall = "0.12" # For mocking in tests
fake = "2.9" # For generating test data
proptest = "1.4" # Property-based testing

# Performance
once_cell = "1.19" # Lazy static initialization
parking_lot = "0.12" # Better synchronization primitives

# Utilities
regex = "1.10"
base64 = "0.21"
sha2 = "0.10"
rand = "0.8"
```

**rust-support/Cargo.toml Example:**
```toml
[package]
name = "rust-support"
version = "0.1.0"
edition = "2021"

[dependencies]
# Use workspace dependencies
actix-web = { workspace = true }
diesel = { workspace = true }
tokio = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
uuid = { workspace = true }
chrono = { workspace = true }
jsonwebtoken = { workspace = true }
bcrypt = { workspace = true }
thiserror = { workspace = true }
anyhow = { workspace = true }
tracing = { workspace = true }
lapin = { workspace = true }
config = { workspace = true }
toml = { workspace = true }
validator = { workspace = true }
r2d2 = { workspace = true }
async-trait = { workspace = true }
```

**Repository Module Cargo.toml Example (module-user):**
```toml
[package]
name = "module-user"
version = "0.1.0"
edition = "2021"

[dependencies]
# Must depend on rust-support
rust-support = { path = "../../rust-support" }

# Workspace dependencies
diesel = { workspace = true }
serde = { workspace = true }
uuid = { workspace = true }
chrono = { workspace = true }
validator = { workspace = true }
anyhow = { workspace = true }
```

### Nuxt Dependencies

**package.json:**
```json
{
  "name": "pantheon-nuxt",
  "version": "0.1.0",
  "type": "module",
  "private": true,
  "scripts": {
    "dev": "nuxt dev",
    "build": "nuxt build",
    "generate": "nuxt generate",
    "preview": "nuxt preview",
    "postinstall": "nuxt prepare",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "type-check": "nuxt typecheck"
  },
  "dependencies": {
    "primevue": "^3.50.0",
    "primeicons": "^7.0.0",
    "@iconify/vue": "^4.1.1",
    "@iconify/json": "^2.2.0",
    "pinia": "^2.1.7",
    "@pinia/nuxt": "^0.5.1",
    "@vueuse/core": "^10.7.2",
    "@vueuse/nuxt": "^10.7.2",
    "chart.js": "^4.4.1",
    "vue-chartjs": "^5.3.0"
  },
  "devDependencies": {
    "@nuxt/devtools": "^1.0.8",
    "@nuxtjs/eslint-config-typescript": "^12.1.0",
    "@typescript-eslint/eslint-plugin": "^6.19.0",
    "@typescript-eslint/parser": "^6.19.0",
    "eslint": "^8.56.0",
    "eslint-plugin-vue": "^9.20.0",
    "nuxt": "^3.10.0",
    "typescript": "^5.3.3",
    "vue-tsc": "^1.8.27",
    "@nuxt/test-utils": "^3.10.0",
    "vitest": "^1.2.0",
    "playwright": "^1.41.0"
  }
}
```

**pantheon-docs package.json:**
```json
{
  "name": "pantheon-docs",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "nuxt dev --port 3001",
    "build": "nuxt build",
    "generate": "nuxt generate",
    "preview": "nuxt preview"
  },
  "dependencies": {
    "swagger-ui-dist": "^5.11.0"
  },
  "devDependencies": {
    "@nuxt/devtools": "^1.0.8",
    "nuxt": "^3.10.0",
    "typescript": "^5.3.3"
  }
}
```

### System Dependencies

**Development Environment:**
- Rust 1.75+ (dengan rustup)
- Node.js 20+ (dengan nvm recommended)
- PostgreSQL 15+
- RabbitMQ 3.12+
- Docker 24+ dan Docker Compose v2
- Git 2.40+

**Optional Tools:**
- cargo-watch - untuk auto-reload saat development
- cargo-tarpaulin - untuk code coverage
- cargo-audit - untuk security audit
- diesel_cli - untuk database migrations
- k6 atau Apache JMeter - untuk load testing
- Postman atau Insomnia - untuk API testing

## Risks & Considerations

### Technical Risks

1. **Multi-Tenant Database Complexity**
   - **Risk:** Connection management untuk multi-database (core + N tenant databases) bisa menjadi bottleneck, terutama saat scale
   - **Impact:** High - Could cause performance degradation
   - **Mitigation:**
     - Implement proper connection pooling per database
     - Cache connection instances
     - Monitor connection pool usage
     - Implement connection health checks dan auto-reconnect
     - Consider connection pool size limits per tenant
   - **Contingency:** Use connection proxy (PgBouncer) jika connection limit tercapai

2. **Service Provider Loading Order**
   - **Risk:** Incorrect provider loading order bisa cause runtime errors atau circular dependencies
   - **Impact:** Medium - Could break application startup
   - **Mitigation:**
     - Clearly document provider dependencies
     - Implement dependency graph resolution
     - Add validation untuk provider order
     - Comprehensive startup tests
   - **Contingency:** Manual ordering dengan clear documentation

3. **Configuration Hierarchy Complexity**
   - **Risk:** Config merging dari 4 levels (repositories → projects → groups → tenants) bisa sulit di-debug
   - **Impact:** Medium - Could cause unexpected behavior
   - **Mitigation:**
     - Log final merged config di startup (development mode)
     - Implement config validation
     - Provide config inspection tools
     - Clear documentation dengan examples
   - **Contingency:** Simplify hierarchy jika terlalu complex (skip group level)

4. **Entity-Connection Mapping**
   - **Risk:** Wrong connection mapping bisa cause data corruption atau tenant data leakage
   - **Impact:** Critical - Data security issue
   - **Mitigation:**
     - Comprehensive testing untuk tenant isolation
     - Code review untuk connection mapping
     - Add runtime validation
     - Audit logging untuk data access
     - Use type system untuk enforce correct connections
   - **Contingency:** Rollback dan data audit jika leak terdeteksi

5. **JWT Security**
   - **Risk:** Token theft, replay attacks, atau improper token validation
   - **Impact:** High - Security breach
   - **Mitigation:**
     - Short access token expiry (15 min)
     - Refresh token rotation
     - httpOnly cookies di frontend
     - HTTPS only di production
     - Implement token blacklist untuk logout
     - Regular security audits
   - **Contingency:** Immediate token revocation mechanism

6. **Database Migration Management**
   - **Risk:** Migration failures di multi-database environment (core + tenants) bisa inconsistent state
   - **Impact:** High - Could cause data corruption
   - **Mitigation:**
     - Transaction-based migrations
     - Rollback strategy
     - Test migrations di staging first
     - Backup before migration
     - Track migration versions per database
   - **Contingency:** Restore from backup dan re-apply migrations

7. **Cluster Schema Auto-Generation**
   - **Risk:** Automatic cluster schema generation (cashier_2026, scm_2026) bisa fail atau create wrong schema
   - **Impact:** Medium - Could disrupt business operations
   - **Mitigation:**
     - Manual creation untuk Phase 1
     - Thorough testing sebelum automation (Plan 3)
     - Dry-run mode
     - Notifications before creation
     - Rollback mechanism
   - **Contingency:** Manual schema creation SOP

8. **RabbitMQ Reliability**
   - **Risk:** RabbitMQ downtime bisa cause job loss atau delay
   - **Impact:** Medium - Background jobs delayed
   - **Mitigation:**
     - Persistent messages
     - Dead letter queues
     - Retry mechanism dengan exponential backoff
     - RabbitMQ clustering (future)
     - Monitoring dan alerts
   - **Contingency:** Manual job re-queue dari failed_jobs table

9. **Rust Learning Curve**
   - **Risk:** Team unfamiliar dengan Rust bisa slow down development
   - **Impact:** Medium - Timeline delay
   - **Mitigation:**
     - Start dengan simple modules
     - Pair programming
     - Code review dengan Rust expert
     - Comprehensive documentation
     - Rust training sessions
   - **Contingency:** Simplify architecture jika needed

10. **Repository Migration dari Laravel**
    - **Risk:** Incomplete atau incorrect migration dari wellmed Laravel repositories
    - **Impact:** Medium - Missing features atau bugs
    - **Mitigation:**
      - Review setiap Laravel Model sebelum migrate
      - Create mapping document
      - Test migrated entities
      - Keep Laravel codebase sebagai reference
    - **Contingency:** Iterative migration, prioritas features penting dulu

### Resource Risks

1. **Timeline Optimism**
   - **Risk:** 8 minggu mungkin tidak cukup untuk semua features
   - **Impact:** High - Delayed launch
   - **Mitigation:**
     - Prioritize MVP features
     - Use timeboxing untuk setiap phase
     - Weekly progress review
     - Defer nice-to-have features ke Plan 2
     - Buffer time untuk unexpected issues
   - **Contingency:** Extend timeline atau reduce scope

2. **Single Point of Failure**
   - **Risk:** Key developer tidak available bisa block progress
   - **Impact:** High - Project stalled
   - **Mitigation:**
     - Knowledge sharing sessions
     - Comprehensive documentation
     - Code review untuk knowledge distribution
     - Pair programming
   - **Contingency:** Cross-training team members

3. **Infrastructure Issues**
   - **Risk:** Development environment setup issues bisa delay start
   - **Impact:** Medium - Lost time
   - **Mitigation:**
     - Docker-based development environment
     - Automated setup scripts
     - Clear setup documentation
     - Fallback to cloud development environments
   - **Contingency:** Provide pre-configured VMs

### Business Risks

1. **Scope Creep**
   - **Risk:** Adding features during development
   - **Impact:** High - Timeline delay, technical debt
   - **Mitigation:**
     - Strict adherence to plan
     - Change request process
     - Weekly scope review
     - Defer additional features ke Plan 2+
   - **Contingency:** Re-baseline plan dengan new scope

2. **Requirement Changes**
   - **Risk:** Business requirements change mid-development
   - **Impact:** High - Rework needed
   - **Mitigation:**
     - Clear requirements sign-off
     - Change impact analysis
     - Regular stakeholder demo
     - Flexible architecture untuk future changes
   - **Contingency:** Assess impact dan negotiate timeline/scope

3. **Technology Risk**
   - **Risk:** Chosen technology tidak cocok untuk use case
   - **Impact:** Critical - Might need rewrite
   - **Mitigation:**
     - POC untuk critical components
     - Research best practices
     - Consult experts
     - Plan for abstraction layers
   - **Contingency:** Have alternative technology stack ready

### Operational Risks

1. **Data Migration Risk**
   - **Risk:** Jika ada existing data, migration bisa cause data loss
   - **Impact:** Critical - Business disruption
   - **Mitigation:**
     - Fresh start untuk Phase 1 (no existing data)
     - Future migration plan untuk existing systems
     - Comprehensive backup strategy
   - **Contingency:** Rollback mechanism

2. **Performance at Scale**
   - **Risk:** System tidak perform well dengan banyak tenants
   - **Impact:** High - Poor user experience
   - **Mitigation:**
     - Performance testing di Phase 1
     - Connection pool tuning
     - Database indexing
     - Caching strategy
     - Monitoring dari early stage
   - **Contingency:** Optimization sprint atau infrastructure upgrade

## Success Criteria

### Phase 1: Foundation - Must Have (Critical)

**Infrastructure & Architecture:**
- [ ] ✅ Rust backend running dengan Actix-web/Axum
- [ ] ✅ Nuxt 4 frontend running dengan BFF pattern
- [ ] ✅ pantheon-docs application serving Swagger UI
- [ ] ✅ PostgreSQL multi-database setup (core + tenant template)
- [ ] ✅ RabbitMQ integration working
- [ ] ✅ Docker Compose setup dengan semua services
- [ ] ✅ All services dapat communicate dengan proper

**Core Libraries:**
- [ ] ✅ rust-support library complete dengan:
  - BaseEntity, BaseResource, BaseController, BaseService traits
  - Service Provider pattern implemented
  - Database Manager dengan multi-database support
  - Connection Manager dengan pooling
  - Configuration System dengan hierarchy merging
  - JWT utilities dengan HS256
  - Auth middleware
- [ ] ✅ microtenant library complete
- [ ] ✅ rust-core, rust-stub, rust-package-generator libraries

**Repository Migration:**
- [ ] ✅ Minimal 20 repositories migrated dari wellmed (dari ~35 target)
- [ ] ✅ Setiap repository punya:
  - Service provider
  - Proper Cargo.toml
  - Basic entities migrated
  - ViewResource & ShowResource
- [ ] ✅ Priority modules complete:
  - module-user
  - module-regional
  - module-workspace
  - module-people
  - module-employee
  - module-item
  - module-warehouse
  - module-transaction
  - module-payment

**Database:**
- [ ] ✅ Core database (pantheon) dengan ALL required tables (18 tables dari requirements)
- [ ] ✅ Tenant database template dengan ALL required tables (27 tables dari requirements)
- [ ] ✅ Cluster schema templates (cashier_*, scm_*) defined
- [ ] ✅ Migrations working untuk core dan tenant databases
- [ ] ✅ Entity-connection mapping configured dan working
- [ ] ✅ Seed data untuk:
  - Indonesia regional data
  - Default licenses
  - Test tenant
  - Test users
  - Default permissions & roles

**Authentication & Authorization:**
- [ ] ✅ JWT authentication working (HS256 dengan secret dari requirements)
- [ ] ✅ Login endpoint returning access_token + refresh_token
- [ ] ✅ Refresh token endpoint working
- [ ] ✅ Logout endpoint working
- [ ] ✅ Protected endpoints require valid token
- [ ] ✅ Auth middleware dengan tenant resolution
- [ ] ✅ Multi-tenant isolation verified (user A cannot access tenant B data)

**Frontend:**
- [ ] ✅ Nuxt 4 running dengan PrimeVue styling
- [ ] ✅ Iconify integration working
- [ ] ✅ BFF server working (server/api/ routes)
- [ ] ✅ Login page working end-to-end
- [ ] ✅ Dashboard page accessible after login
- [ ] ✅ Token stored in httpOnly cookies
- [ ] ✅ Auto token refresh on 401
- [ ] ✅ Logout working
- [ ] ✅ Pinia stores working (auth, tenant, app)

**Background Jobs:**
- [ ] ✅ RabbitMQ connection working
- [ ] ✅ Job queue framework implemented
- [ ] ✅ Worker implementation working
- [ ] ✅ At least 1 example job working (SendEmailJob)
- [ ] ✅ Retry mechanism working
- [ ] ✅ Failed jobs tracking

**Documentation:**
- [ ] ✅ OpenAPI/Swagger spec generated
- [ ] ✅ Auth endpoints documented di Swagger
- [ ] ✅ Swagger UI accessible di pantheon-docs
- [ ] ✅ README.md complete dengan setup instructions
- [ ] ✅ SETUP.md complete
- [ ] ✅ DEVELOPMENT.md complete
- [ ] ✅ Architecture diagrams created

**CI/CD:**
- [ ] ✅ GitHub repository setup dengan 3 branches (main, staging, develop)
- [ ] ✅ Development workflow working (lint, test, build)
- [ ] ✅ Staging workflow working (integration tests)
- [ ] ✅ Production workflow configured (placeholder)
- [ ] ✅ Unit tests running in CI
- [ ] ✅ Docker build working in CI

**Testing:**
- [ ] ✅ Unit test coverage > 70% untuk rust-support
- [ ] ✅ Unit test coverage > 60% untuk key repositories
- [ ] ✅ Integration tests untuk auth flow passing
- [ ] ✅ Integration tests untuk multi-tenant isolation passing
- [ ] ✅ E2E test untuk login flow passing
- [ ] ✅ All tests passing in CI

**Performance:**
- [ ] ✅ Login API response < 500ms (p95)
- [ ] ✅ List APIs response < 200ms (p95)
- [ ] ✅ Create APIs response < 300ms (p95)
- [ ] ✅ Frontend initial load < 3s
- [ ] ✅ Database queries optimized dengan indexes

**Security:**
- [ ] ✅ No critical security vulnerabilities (cargo-audit, npm audit)
- [ ] ✅ JWT tokens secure (httpOnly cookies, proper expiry)
- [ ] ✅ Tenant isolation verified
- [ ] ✅ SQL injection protected (via ORM)
- [ ] ✅ XSS protected (via framework)
- [ ] ✅ CORS configured properly

### Phase 1: Foundation - Nice to Have (Optional)

**Enhanced Features:**
- [ ] 📦 All 35 repositories migrated (bisa defer ke Plan 2 jika waktu tidak cukup)
- [ ] 📦 Cluster schema auto-generation (defer ke Plan 3)
- [ ] 📦 Comprehensive E2E test suite (basic E2E must have, comprehensive optional)
- [ ] 📦 Performance benchmarks documented
- [ ] 📦 Load testing results documented
- [ ] 📦 Advanced monitoring setup (basic health checks must have)
- [ ] 📦 Grafana dashboards (defer ke Plan 7)

**Documentation:**
- [ ] 📦 Video walkthrough
- [ ] 📦 Interactive API playground
- [ ] 📦 Code examples untuk setiap module
- [ ] 📦 Troubleshooting flowcharts

**Quality:**
- [ ] 📦 Test coverage > 80% (70% target untuk must have)
- [ ] 📦 Zero clippy warnings (allow some warnings jika blocking)
- [ ] 📦 Code complexity metrics documented
- [ ] 📦 Performance profiling results

### Acceptance Checklist

**Functional Requirements (dari PANTHEON APP REQUIREMENT.md):**
- [ ] ✅ Rust Programming Language untuk BE
- [ ] ✅ Nuxt 4 untuk FE dengan Iconify dan BFF
- [ ] ✅ PrimeVue CSS
- [ ] ✅ Login authentication HS256 dengan secret spesifik
- [ ] ✅ JWT token + refresh token
- [ ] ✅ Multi-tenant dengan custom database per tenant
- [ ] ✅ Multi-database dan multi-schema support
- [ ] ✅ Cluster database structure (manual untuk Phase 1)
- [ ] ✅ Database manager dengan connection mapping
- [ ] ✅ Base entity dengan getViewResource, getShowResource, toViewApi, toShowApi
- [ ] ✅ Resource (ViewResource & ShowResource)
- [ ] ✅ Configuration system dengan override hierarchy
- [ ] ✅ Service Provider pattern
- [ ] ✅ REST API dengan controller/handler dan service
- [ ] ✅ Dependency injection via interface/traits
- [ ] ✅ Swagger API documentation
- [ ] ✅ RabbitMQ untuk background jobs
- [ ] ✅ Docker containerization
- [ ] ✅ CI/CD dengan 3 environments (production, staging, development)
- [ ] ✅ Unit test automation di GitHub Actions
- [ ] ✅ All core database tables created (dari requirement #74)
- [ ] ✅ All tenant database tables created (dari requirement #75)
- [ ] ✅ Cluster database tables defined (cashier_*, scm_*)
- [ ] ✅ Repositories migrated except excluded modules

**Non-Functional Requirements:**
- [ ] ✅ System stable (no crashes during normal operation)
- [ ] ✅ Performance acceptable (meet targets above)
- [ ] ✅ Security standards met (no critical vulnerabilities)
- [ ] ✅ Code maintainable (proper structure, documentation)
- [ ] ✅ Scalable architecture (ready untuk multiple tenants)

### Deliverables

1. **Source Code:**
   - pantheon-app repository (all Rust code - backend workspace)
   - pantheon-nuxt repository (frontend code)
   - pantheon-docs repository (documentation app)

2. **Documentation:**
   - README.md (overview)
   - SETUP.md (environment setup)
   - DEVELOPMENT.md (development guide)
   - RUNBOOK.md (operational guide)
   - Architecture diagrams
   - OpenAPI specification (openapi.json)
   - This plan (updated dengan lessons learned)

3. **Infrastructure:**
   - docker-compose.yml (all services)
   - Dockerfiles untuk semua apps
   - Database migration files (core + tenant)
   - Database seed scripts
   - CI/CD workflows (.github/workflows/)

4. **Tests:**
   - Unit tests (Rust + Frontend)
   - Integration tests
   - E2E tests (basic)
   - Test documentation

5. **Deployment Artifacts:**
   - Docker images (di registry atau local)
   - Environment configuration examples
   - Deployment guide (basic)

## Timeline & Milestones

**Total Duration:** 8 minggu (56 hari kalender, ~40 hari kerja)

### Weekly Breakdown:

**Minggu 1-2: Backend Foundation**
- Setup Rust workspace
- Implement rust-support library
- Setup core support repositories
- Milestone: rust-support complete dengan BaseEntity, Service Provider, Database Manager

**Minggu 2-3: Repository Migration**
- Migrate priority modules dari wellmed
- Configure entity-connection mapping
- Milestone: 20+ repositories migrated dan working

**Minggu 3: Database Setup**
- Setup PostgreSQL infrastructure
- Create all migrations (core + tenant + cluster)
- Run migrations dan seed data
- Milestone: Database infrastructure complete

**Minggu 4: Project Setup & Auth**
- Setup pantheon-business project
- Implement service provider loading
- Implement auth module complete
- Milestone: Authentication working end-to-end

**Minggu 5: Frontend**
- Setup Nuxt 4 dengan BFF
- Implement auth flow di frontend
- Create dashboard layout
- Setup pantheon-docs
- Milestone: Frontend working dengan login/logout

**Minggu 6: Infrastructure**
- Complete Docker setup
- RabbitMQ integration
- Swagger documentation
- Milestone: All infrastructure components working

**Minggu 7: CI/CD & Testing**
- Setup GitHub Actions workflows
- Write comprehensive tests
- Load testing
- Security testing
- Milestone: All tests passing, CI/CD working

**Minggu 8: Documentation & Handover**
- Complete documentation
- Performance optimization
- Security hardening
- Final review
- Milestone: Production ready, Plan 1 complete

### Key Milestones:

| Week | Milestone | Completion Criteria |
|------|-----------|---------------------|
| 2 | Backend Foundation Complete | rust-support working, tests passing |
| 3 | Repositories Migrated | 20+ modules migrated, entities working |
| 3 | Database Complete | All tables created, migrations working |
| 4 | Auth Working | Login/logout API working, JWT verified |
| 5 | Frontend Complete | UI working end-to-end, BFF working |
| 6 | Infrastructure Complete | Docker, RabbitMQ, Swagger all working |
| 7 | Testing Complete | All tests passing, CI/CD working |
| 8 | Plan 1 Complete | All success criteria met, documented |

## Next Steps After Plan 1

### Plan 2: Business Modules Implementation (Est. 6 minggu)
**Scope:**
- Implement business logic untuk:
  - Product Management (CRUD, variants, pricing)
  - Inventory Management (stocks, warehouses, stock cards)
  - Cashier/POS (billings, payments, invoices)
  - Finance Management (transactions, accounting, reports)
  - HR Management (employees, attendance, payroll)
  - SCM (procurement, purchase orders, distributions)
- Complete UI untuk semua modules
- Reports dan dashboards
- Business rules dan validations

### Plan 3: Cluster Database Auto-Generation (Est. 3 minggu)
**Scope:**
- Implement automatic cluster schema generation
- RabbitMQ scheduled jobs (5 hari sebelum tahun baru)
- Migration runner untuk cluster schemas
- Monitoring dan alerting
- Testing dan validation

### Plan 4: Search & Analytics (Est. 4 minggu)
**Scope:**
- Elasticsearch integration
- Full-text search implementation
- Analytics dan reporting
- Data aggregation
- Search UI components

### Plan 5: Mobile & Desktop Support (Est. 4 minggu)
**Scope:**
- PWA implementation (offline support, push notifications)
- Electron wrapper untuk desktop app
- Mobile-responsive improvements
- App packaging dan distribution

### Plan 6: Advanced Features (Est. 6 minggu)
**Scope:**
- MQTT integration (jika needed)
- Real-time features (WebSocket)
- Advanced reporting
- Data export/import
- Backup/restore automation

### Plan 7: Production Deployment & Optimization (Est. 4 minggu)
**Scope:**
- Production server provisioning
- Load balancer setup
- Database replication
- Full monitoring stack (Prometheus, Grafana)
- Log aggregation (ELK stack)
- CDN setup
- SSL/TLS certificates
- Performance optimization
- Security hardening
- Disaster recovery plan

### Plan 8: pantheon-hq Project (Est. 8 minggu)
**Scope:**
- SAAS platform untuk managing tenants
- Tenant provisioning automation
- Billing dan subscription management
- Usage analytics
- Admin dashboard
- Customer portal
- API for third-party integrations

## Resource Requirements

### Development Team:
- **1 Senior Rust Developer** (full-time) - Core infrastructure, database manager, auth
- **1 Mid-level Rust Developer** (full-time) - Repository migration, testing
- **1 Senior Frontend Developer** (full-time) - Nuxt 4, BFF, UI components
- **1 DevOps Engineer** (part-time, ~50%) - Docker, CI/CD, infrastructure
- **1 QA Engineer** (part-time, ~50%) - Testing, test automation
- **1 Tech Lead / Architect** (part-time, ~30%) - Architecture decisions, code review

### Infrastructure:
- **Development Environment:**
  - Development machines (local)
  - Docker Desktop atau Linux dengan Docker
- **Testing Environment:**
  - CI/CD (GitHub Actions - free tier cukup)
  - Optional: Staging server (1 VM, 4GB RAM minimum)
- **Database Development:**
  - PostgreSQL lokal atau Docker
  - Optional: Database server untuk shared development

### Tools & Services:
- **Version Control:** GitHub (repository gratis)
- **CI/CD:** GitHub Actions (included)
- **Container Registry:** Docker Hub (free tier) atau GitHub Container Registry
- **Communication:** Slack/Discord untuk team communication
- **Project Management:** GitHub Projects atau Jira

## Approval Section

### Sign-off Required From:
- [ ] **Product Owner** - Requirements & scope confirmation
- [ ] **Technical Architect** - Architecture approval
- [ ] **DevOps Lead** - Infrastructure feasibility
- [ ] **Engineering Manager** - Resource allocation & timeline

### Pre-Implementation Checklist:
- [ ] All stakeholders reviewed plan
- [ ] Resource allocation confirmed
- [ ] Timeline approved
- [ ] Budget approved (if applicable)
- [ ] Risks acknowledged dan mitigation plans accepted
- [ ] Success criteria agreed upon
- [ ] Development environment setup requirements clear
- [ ] Communication channels established
- [ ] GitHub repository access granted untuk team
- [ ] Required tools dan licenses available

### Approval Status: **PENDING REVIEW**

**Action Items Before Starting:**
1. Review complete plan dengan team
2. Confirm resource availability
3. Setup development environments
4. Create GitHub repository
5. Schedule kickoff meeting
6. Assign roles dan responsibilities

### Post-Approval:
Once approved, will create:
- `1-FOUNDATION-INFRASTRUCTURE-PROGRESS.md` - Weekly progress tracking
- `1-FOUNDATION-INFRASTRUCTURE-RETROSPECTIVE.md` - Lessons learned (after completion)
- GitHub Project board dengan tasks dari plan
- Setup team calendar dengan milestones

---

## Plan Metadata

**Plan ID:** PANTHEON-PLAN-001
**Plan Name:** Foundation & Infrastructure Setup
**Plan Version:** 2.0
**Plan Status:** PENDING APPROVAL
**Plan Created:** 2026-04-02
**Plan Last Updated:** 2026-04-03
**Plan Author:** Development Team
**Estimated Duration:** 8 minggu (56 hari kalender)
**Estimated Effort:** ~6 person-months
**Priority:** Critical (P0)
**Dependencies:** None (foundation plan)
**Risk Level:** High (complex multi-tenant architecture)

**Review Cycle:**
- Weekly progress review setiap Jumat
- Bi-weekly stakeholder demo
- Ad-hoc reviews untuk blocking issues

**Success Measurement:**
- All "Must Have" success criteria met
- All tests passing
- All documentation complete
- Zero critical security vulnerabilities
- Performance targets met
- Ready for Plan 2 implementation

---

**🚀 Ready to proceed pending approval!**
