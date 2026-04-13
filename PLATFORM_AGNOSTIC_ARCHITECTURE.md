# Platform-Agnostic Architecture: System Design

**Document Version**: 1.0  
**Date**: 2026-03-24  
**Status**: Planning Phase - DO NOT BUILD

---

## Executive Summary

Greenfield build of Clinical Roundup as a **vendor-neutral, layered architecture** where:
- **Frontend**: Decoupled from backend, communicates via standard HTTP/REST APIs only
- **Backend**: Abstracted from database, deployable on any cloud or on-premises infrastructure
- **Database**: Swappable (PostgreSQL, MongoDB, etc.) via ORM/abstraction layer
- **Authentication**: Provider-agnostic OAuth2/JWT implementation
- **Storage**: Pluggable file backend (S3, Azure Blob, local filesystem)

---

## 1. TARGET ARCHITECTURE

### Reference: Existing SPA (for feature parity)
The existing Clinical Roundup SPA uses GitHub Pages + SharePoint Lists. This greenfield build is informed by its feature set but is **not a migration** -- it is a clean-slate implementation with no SharePoint dependencies.

### Greenfield Architecture (Platform-Agnostic)
```
┌──────────────────────────────────────┐
│   Frontend (Any Cloud/Host)          │
│  ├─ GitHub Pages                     │
│  ├─ Vercel                           │
│  ├─ Netlify                          │
│  ├─ AWS S3 + CloudFront             │
│  └─ Azure Static Web Apps            │
└──────────────┬───────────────────────┘
               │ HTTP/REST API (standard)
               │ OpenAPI spec
               │ No vendor-specific calls
┌──────────────▼───────────────────────┐
│   Backend API (Stateless)            │
│  ├─ Node.js / Python / Go            │
│  ├─ Azure Functions / AWS Lambda     │
│  ├─ Docker containers                │
│  └─ On-premises servers              │
├─ Route handlers (endpoints)          │
├─ Business logic                      │
├─ Authorization (JWT/OAuth2)          │
├─ Audit logging                       │
└──────────────┬───────────────────────┘
               │ Database abstraction
               │ (ORM/Query builder)
┌──────────────▼───────────────────────┐
│   Data Access Layer (Swappable)      │
│  ├─ PostgreSQL                       │
│  ├─ MySQL                            │
│  ├─ MongoDB                          │
│  ├─ Firebase Firestore              │
│  └─ DynamoDB                         │
└─────────────────────────────────────┘

┌──────────────────────────────────────┐
│   File Storage (Swappable)           │
│  ├─ AWS S3                           │
│  ├─ Azure Blob Storage               │
│  ├─ Google Cloud Storage             │
│  └─ Local filesystem                 │
└─────────────────────────────────────┘

┌──────────────────────────────────────┐
│   Authentication (Swappable)         │
│  ├─ Azure AD                         │
│  ├─ Google                           │
│  ├─ GitHub                           │
│  ├─ Auth0                            │
│  └─ Custom JWT                       │
└─────────────────────────────────────┘

Benefits:
✓ Vendor independence
✓ True separation of concerns
✓ Testable in isolation
✓ Deployable anywhere
✓ Consistent APIs
```

---

## 2. LAYERED ARCHITECTURE

### Layer 1: Frontend (Browser)
**Location**: Separate repo or `/frontend` directory
**Technology**: HTML, CSS, JavaScript (vanilla or framework-agnostic)
**Deployment**: GitHub Pages, Vercel, AWS S3, etc.

> **DECISION REQUIRED: Frontend Framework**
> This greenfield build must decide:
> - **Option A**: Vanilla JS with ES modules (no build step, simplest deployment)
> - **Option B**: Migrate to React/Vue/Svelte (better component structure, state management, but adds build tooling)
> - **Recommendation**: Start with vanilla JS to minimize migration risk, refactor to framework later if needed
>
> All code examples in this document use framework-neutral patterns (fetch/axios, ES modules). Environment variable references like `APP_API_URL` should be loaded from a `config.js` file at runtime rather than baked in at build time, ensuring compatibility with any hosting approach.

**Responsibilities**:
- UI rendering
- User interaction handling
- API calls to backend (via fetch/axios)
- Local state management (React, Vue, or vanilla JS)
- localStorage for offline mode
- Device detection and adaptive layout

**Key Principle**: No business logic, no vendor-specific APIs, no hardcoded endpoints.

**Configuration**:
```javascript
// config.js (loaded from environment)
export const API_BASE_URL = process.env.APP_API_URL || 'https://api.example.com';
export const AUTH_PROVIDER = process.env.APP_AUTH_PROVIDER || 'oauth2';
export const FEATURES = {
  offline_mode: true,
  sync_enabled: true,
  audit_visible: true
};
```

---

### Layer 2: API Gateway / Backend Service
**Location**: `/backend` or separate repo  
**Technology**: Node.js (Express), Python (FastAPI), Go (Gin), or similar  
**Deployment**: Docker container, serverless (Lambda, Cloud Functions), or traditional server

**Responsibilities**:
- HTTP request routing
- Authentication & authorization (JWT validation)
- Input validation & sanitization
- Business logic orchestration
- Database abstraction layer invocation
- Audit logging
- Error handling & response formatting
- Rate limiting
- CORS handling

**Key Principle**: Stateless. Can be scaled horizontally. No database connection pooling at app level (use external).

**Directory Structure**:
```
backend/
├── src/
│  ├── routes/
│  │  ├── auth.js
│  │  ├── patients.js
│  │  ├── audit.js
│  │  ├── settings.js
│  │  └── import-export.js
│  ├── middleware/
│  │  ├── auth.js
│  │  ├── validation.js
│  │  ├── error-handler.js
│  │  └── logging.js
│  ├── services/
│  │  ├── patient-service.js
│  │  ├── audit-service.js
│  │  ├── auth-service.js
│  │  └── export-service.js
│  ├── models/
│  │  ├── patient.js
│  │  ├── audit-log.js
│  │  └── setting.js
│  ├── data-access/
│  │  ├── index.js (exports abstraction)
│  │  ├── postgres-adapter.js
│  │  ├── mongo-adapter.js
│  │  └── abstract-base.js
│  └── app.js (main entry point)
├── tests/
│  ├── unit/
│  ├── integration/
│  └── fixtures/
├── migrations/
│  ├── 001-init-schema.sql
│  └── 002-add-audit-table.sql
├── Dockerfile
├── .env.example
└── package.json
```

---

### Layer 3: Data Access Layer (DAL)
**Location**: `backend/src/data-access/`  
**Technology**: ORM (TypeORM, Prisma, SQLAlchemy, Sequelize) OR Query Builder (Knex.js, SQLAlchemy)

**Responsibilities**:
- Abstract database specifics from business logic
- Support multiple database backends
- Handle connection pooling
- Support transactions
- Provide query builders or ORM models
- Handle migrations
- Ensure consistent query interface

**Key Principle**: Business logic never calls database directly. Always goes through DAL.

**Interface Example**:
```javascript
// data-access/index.js - Abstract interface
class PatientDAO {
  async createPatient(data) { /* impl */ }
  async getPatientById(id) { /* impl */ }
  async updatePatient(id, data) { /* impl */ }
  async deletePatient(id) { /* impl */ }
  async listPatients(filters, pagination) { /* impl */ }
}

// data-access/postgres-adapter.js
class PostgresPatientDAO extends PatientDAO {
  async createPatient(data) {
    return db.query(
      'INSERT INTO patients (name, dob, mrn, ...) VALUES (...)',
      [data.name, data.dob, data.mrn, ...]
    );
  }
}

// data-access/mongo-adapter.js
class MongoPatientDAO extends PatientDAO {
  async createPatient(data) {
    return this.db.collection('patients').insertOne(data);
  }
}

// Runtime selection
const dbType = process.env.DATABASE_TYPE; // 'postgres' or 'mongo'
const PatientDAO = dbType === 'postgres' 
  ? new PostgresPatientDAO() 
  : new MongoPatientDAO();
```

---

### Layer 4: Database (Swappable)
**Options**:
- PostgreSQL (recommended for relational data)
- MySQL
- MongoDB (document-based)
- Firebase Firestore
- AWS DynamoDB
- Cosmos DB

**Database-Neutral Schema**:
```sql
-- PATIENTS
CREATE TABLE patients (
  id UUID PRIMARY KEY,
  visit_key VARCHAR(255) UNIQUE,
  mrn VARCHAR(50),
  name VARCHAR(255),
  dob DATE,
  room VARCHAR(50),
  hospital VARCHAR(255),
  supervising_md VARCHAR(255),
  procedure_status VARCHAR(50),
  plan TEXT,
  follow_up TEXT,
  cpt_primary VARCHAR(10),
  cpt_secondary JSON,
  icd_primary VARCHAR(10),
  charge_codes_secondary TEXT,
  findings_codes JSON,
  findings_values JSON,
  findings_dates JSON,
  findings_text TEXT,
  created_at TIMESTAMP,
  created_by VARCHAR(255),
  updated_at TIMESTAMP,
  updated_by VARCHAR(255),
  deleted_at TIMESTAMP NULL,
  INDEX(mrn),
  INDEX(visit_key),
  INDEX(created_at),
  INDEX(deleted_at)
);

-- AUDIT_LOGS
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY,
  timestamp TIMESTAMP,
  user_id VARCHAR(255),
  action VARCHAR(50),
  action_type VARCHAR(50),
  affected_record_id VARCHAR(255),
  affected_record_type VARCHAR(50),
  field_changed VARCHAR(255),
  old_value TEXT,
  new_value TEXT,
  details JSON,
  ip_address VARCHAR(50),
  user_agent TEXT,
  INDEX(timestamp),
  INDEX(user_id),
  INDEX(affected_record_id),
  INDEX(action)
);

-- SETTINGS
CREATE TABLE settings (
  id UUID PRIMARY KEY,
  key VARCHAR(255) UNIQUE,
  value TEXT,
  type VARCHAR(50),
  last_modified TIMESTAMP,
  modified_by VARCHAR(255)
);

-- USERS
CREATE TABLE users (
  id UUID PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  display_name VARCHAR(255),
  role VARCHAR(50) NOT NULL DEFAULT 'clinician',
  is_active BOOLEAN DEFAULT TRUE,
  last_login TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  created_by VARCHAR(255),
  updated_at TIMESTAMP DEFAULT NOW(),
  INDEX(email),
  INDEX(role)
);

-- ON-CALL SCHEDULE
CREATE TABLE on_call_schedule (
  id UUID PRIMARY KEY,
  date DATE NOT NULL,
  provider VARCHAR(255) NOT NULL,
  hospitals JSON,
  created_at TIMESTAMP DEFAULT NOW(),
  created_by VARCHAR(255),
  updated_at TIMESTAMP DEFAULT NOW(),
  updated_by VARCHAR(255),
  UNIQUE(date, provider)
);

-- BACKFEED (external data ingestion)
CREATE TABLE backfeed (
  id UUID PRIMARY KEY,
  mrn VARCHAR(50) NOT NULL,
  source_system VARCHAR(255),
  feed_type VARCHAR(50),
  payload JSON,
  processed_at TIMESTAMP,
  status VARCHAR(50) DEFAULT 'pending',
  error_message TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  INDEX(mrn),
  INDEX(status),
  INDEX(created_at)
);
```

---

### Layer 5: File Storage (Swappable)
**Abstraction Interface**:
```javascript
// storage/index.js
class StorageAdapter {
  async uploadFile(key, buffer, metadata) { /* impl */ }
  async downloadFile(key) { /* impl */ }
  async deleteFile(key) { /* impl */ }
  async listFiles(prefix) { /* impl */ }
}

// storage/s3-adapter.js
class S3StorageAdapter extends StorageAdapter {
  // Uses AWS SDK
}

// storage/azure-adapter.js
class AzureStorageAdapter extends StorageAdapter {
  // Uses Azure SDK
}

// storage/local-adapter.js
class LocalStorageAdapter extends StorageAdapter {
  // Uses filesystem
}
```

---

### Layer 6: Authentication (Swappable)
**Abstraction Interface**:
```javascript
// auth/index.js
class AuthAdapter {
  async validateToken(token) { /* impl */ }
  async getUserInfo(userId) { /* impl */ }
  async refreshToken(refreshToken) { /* impl */ }
  getAuthorizationUrl(redirectUrl) { /* impl */ }
}

// auth/oauth2-adapter.js
class OAuth2Adapter extends AuthAdapter {
  // Handles OAuth2 flows (Azure AD, Google, GitHub, etc.)
}

// auth/jwt-adapter.js
class JWTAdapter extends AuthAdapter {
  // Validates JWT tokens with custom key
}

// auth/saml-adapter.js
class SAMLAdapter extends AuthAdapter {
  // Handles SAML tokens
}

// Runtime selection
const authType = process.env.AUTH_PROVIDER; // 'oauth2', 'jwt', 'saml'
const AuthAdapter = require(`./auth/${authType}-adapter.js`);
```

---

## 3. API DESIGN (OpenAPI Spec)

### Base Information
```yaml
openapi: 3.0.0
info:
  title: Clinical Roundup API
  version: 1.0.0
  description: Platform-agnostic API for clinical rounding management

servers:
  - url: https://api.example.com/api/v1
    description: Production
  - url: http://localhost:3000/api/v1
    description: Local development

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    Patient:
      type: object
      required:
        - name
        - dob
        - mrn
        - hospital
        - date
      properties:
        id:
          type: string
          format: uuid
        visitKey:
          type: string
        mrn:
          type: string
        name:
          type: string
        dob:
          type: string
          format: date
        room:
          type: string
        hospital:
          type: string
        supervisingMd:
          type: string
        procedureStatus:
          type: string
          enum: [To-Do, In Progress, Complete, Pending, Follow-Up]
        plan:
          type: string
        followUp:
          type: string
        cptPrimary:
          type: string
        cptSecondary:
          type: array
          items:
            type: string
        icdPrimary:
          type: string
        chargeCodesSecondary:
          type: string
        findingsCodes:
          type: array
          items:
            type: string
        findingsValues:
          type: object
        findingsDates:
          type: object
        findingsText:
          type: string
        createdAt:
          type: string
          format: date-time
        createdBy:
          type: string
        updatedAt:
          type: string
          format: date-time
        updatedBy:
          type: string

    AuditLog:
      type: object
      properties:
        id:
          type: string
          format: uuid
        timestamp:
          type: string
          format: date-time
        userId:
          type: string
        action:
          type: string
          enum: [CREATE, READ, UPDATE, DELETE, ARCHIVE, RESTORE, EXPORT, LOGIN, LOGOUT]
        actionType:
          type: string
        affectedRecordId:
          type: string
        affectedRecordType:
          type: string
        fieldChanged:
          type: string
        oldValue:
          type: object
        newValue:
          type: object
        details:
          type: object
        ipAddress:
          type: string
        userAgent:
          type: string

paths:
  /auth/login:
    post:
      tags: [Auth]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                email:
                  type: string
                password:
                  type: string
      responses:
        '200':
          description: Login successful
          content:
            application/json:
              schema:
                type: object
                properties:
                  access_token:
                    type: string
                  refresh_token:
                    type: string
                  user:
                    type: object

  /patients:
    get:
      tags: [Patients]
      security:
        - bearerAuth: []
      parameters:
        - name: date
          in: query
          schema:
            type: string
            format: date
        - name: hospital
          in: query
          schema:
            type: string
        - name: status
          in: query
          schema:
            type: string
        - name: limit
          in: query
          schema:
            type: integer
            default: 50
        - name: offset
          in: query
          schema:
            type: integer
            default: 0
      responses:
        '200':
          description: List of patients
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/Patient'
                  total:
                    type: integer
                  limit:
                    type: integer
                  offset:
                    type: integer
    post:
      tags: [Patients]
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Patient'
      responses:
        '201':
          description: Patient created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Patient'

  /patients/{id}:
    get:
      tags: [Patients]
      security:
        - bearerAuth: []
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Patient details
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Patient'
    put:
      tags: [Patients]
      security:
        - bearerAuth: []
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Patient'
      responses:
        '200':
          description: Patient updated
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Patient'
    delete:
      tags: [Patients]
      security:
        - bearerAuth: []
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
      responses:
        '204':
          description: Patient deleted

  /oncall:
    get:
      tags: [On-Call]
      security:
        - bearerAuth: []
      parameters:
        - name: startDate
          in: query
          schema:
            type: string
            format: date
        - name: endDate
          in: query
          schema:
            type: string
            format: date
      responses:
        '200':
          description: On-call schedules
    post:
      tags: [On-Call]
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                date:
                  type: string
                  format: date
                provider:
                  type: string
                hospitals:
                  type: array
                  items:
                    type: string
      responses:
        '201':
          description: Schedule created

  /users:
    get:
      tags: [Users]
      security:
        - bearerAuth: []
      responses:
        '200':
          description: User list (admin only)
    post:
      tags: [Users]
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                email:
                  type: string
                displayName:
                  type: string
                role:
                  type: string
                  enum: [clinician, billing, admin]
                password:
                  type: string
      responses:
        '201':
          description: User created (admin only)

  /backfeed:
    post:
      tags: [Backfeed]
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                mrn:
                  type: string
                sourceSystem:
                  type: string
                feedType:
                  type: string
                  enum: [lab_result, diagnosis, order, discharge]
                payload:
                  type: object
      responses:
        '201':
          description: Backfeed received

  /import/csv:
    post:
      tags: [Import/Export]
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          multipart/form-data:
            schema:
              type: object
              properties:
                file:
                  type: string
                  format: binary
      responses:
        '200':
          description: Import summary

  /export/excel:
    get:
      tags: [Import/Export]
      security:
        - bearerAuth: []
      parameters:
        - name: date
          in: query
          schema:
            type: string
            format: date
      responses:
        '200':
          description: Excel file download
          content:
            application/vnd.openxmlformats-officedocument.spreadsheetml.sheet:
              schema:
                type: string
                format: binary

  /export/pdf:
    get:
      tags: [Import/Export]
      security:
        - bearerAuth: []
      parameters:
        - name: ids
          in: query
          schema:
            type: string
      responses:
        '200':
          description: PDF file download
          content:
            application/pdf:
              schema:
                type: string
                format: binary

  /audit/logs:
    get:
      tags: [Audit]
      security:
        - bearerAuth: []
      parameters:
        - name: patientId
          in: query
          schema:
            type: string
        - name: userId
          in: query
          schema:
            type: string
        - name: action
          in: query
          schema:
            type: string
        - name: startDate
          in: query
          schema:
            type: string
            format: date
        - name: endDate
          in: query
          schema:
            type: string
            format: date
      responses:
        '200':
          description: Audit logs
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/AuditLog'
```

---

## 4. DEPLOYMENT ARCHITECTURES

### Option A: Containerized (Docker)
```
┌─────────────────────┐
│  GitHub Repository  │
└──────────┬──────────┘
           │ (git push)
┌──────────▼──────────┐
│  CI/CD Pipeline     │
│  (GitHub Actions)   │
├─ Build Docker image│
├─ Run tests         │
├─ Push to registry  │
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│  Container Registry │
│  (Docker Hub)       │
└──────────┬──────────┘
           │
     ┌─────┴──────┬──────────┬────────┐
     │            │          │        │
┌────▼──┐   ┌────▼──┐   ┌────▼──┐   ┌───▼───┐
│  AWS  │   │ Azure │   │ GCP  │   │On-Prem│
│(Fargate)  │(ACI)  │   │(GKE) │   │(K8s)  │
└────────┘   └───────┘   └──────┘   └───────┘
```

### Option B: Serverless (Lambda/Functions)
```
┌─────────────────────┐
│  GitHub Repository  │
└──────────┬──────────┘
           │ (git push)
┌──────────▼──────────┐
│  CI/CD Pipeline     │
│  (GitHub Actions)   │
├─ Build function    │
├─ Run tests         │
└──────────┬──────────┘
           │
     ┌─────┴──────┬──────────┐
     │            │          │
┌────▼──┐   ┌────▼──┐   ┌────▼──┐
│  AWS  │   │ Azure │   │ GCP  │
│Lambda │   │Functions   │Cloud Fns
└────────┘   └───────┘   └──────┘
```

### Option C: Traditional Server (VPS/EC2)
```
┌─────────────────────┐
│  GitHub Repository  │
└──────────┬──────────┘
           │ (git pull / webhook)
┌──────────▼──────────┐
│  Web Server         │
│  (Ubuntu/CentOS)    │
├─ Node.js/Python    │
├─ systemd/supervisor│
├─ Nginx/Apache      │
└────────────────────┘
           │
        (TCP)
│
┌──────────────────┐
│  Database (PG)   │
└──────────────────┘
```

---

## 5. DATA SEEDING & IMPORT

### Initial Data Population
Since this is a greenfield build, there is no migration. However, the system must support bulk data import for initial population:

```
Step 1: Prepare
┌──────────────────┐
│  CSV/Excel/JSON  │
│  source files    │
└────────┬─────────┘
         │
         ▼
      Validate format

Step 2: Import
┌──────────────────┐
│  Import Service  │
│  (Node.js)       │
├─ Validate rows  │
├─ Generate UUIDs │
├─ Hash passwords │
└────────┬─────────┘
         │
         ▼
      Bulk insert

Step 3: Verify
┌──────────────────┐
│  Target DB       │
│  (PostgreSQL/    │
│  MongoDB)        │
└──────────────────┘
```

**Import Sources Supported**:
- CSV files (patient lists, user lists, on-call schedules)
- Excel files (.xlsx)
- JSON files (settings, configuration)
- API-based bulk import endpoint (`POST /api/v1/import/csv`)

---

## 6. CONFIGURATION MANAGEMENT

### Environment Variables (All Environments)
```bash
# Database
DATABASE_TYPE=postgres|mongo|firestore|dynamodb
DATABASE_URL=postgresql://user:pass@host:5432/db
DATABASE_POOL_SIZE=20

# API
API_PORT=3000
API_BASE_URL=https://api.example.com
NODE_ENV=development|staging|production

# Authentication
AUTH_PROVIDER=oauth2|jwt|saml
AUTH_SECRET=<secret>
OAUTH_CLIENT_ID=<id>
OAUTH_CLIENT_SECRET=<secret>

# File Storage
STORAGE_TYPE=s3|azure|gcp|local
S3_BUCKET=my-bucket
S3_REGION=us-east-1
S3_ACCESS_KEY=<key>
S3_SECRET_KEY=<secret>

# Compliance
COMPLIANCE_MODE=relaxed|hipaa_strict|sox_strict
ENCRYPTION_ENABLED=true|false
AUDIT_VERBOSITY=minimal|detailed

# Monitoring
SENTRY_DSN=https://...
LOG_LEVEL=debug|info|warn|error
METRICS_ENABLED=true

# Feature Flags
FEATURES_LOCAL_MODE=true
FEATURES_OFFLINE_SYNC=true
```

---

## 7. TESTING STRATEGY

### Unit Tests
- DAO classes (with mock database)
- Business logic services
- Utility functions
- No external dependencies

### Integration Tests
- API endpoints (with test database)
- DAL with real database (PostgreSQL + MongoDB)
- Authentication flow
- File upload/download

### E2E Tests
- Full workflow (login → create patient → export)
- Multiple deployment targets

### Test Databases
```
# Docker Compose for local testing
version: '3'
services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_PASSWORD: testpass
      POSTGRES_DB: test_db
  
  mongodb:
    image: mongo:5.0
```

---

## 8. MONITORING & OBSERVABILITY

### Logging Strategy
```javascript
// Structured logging (JSON)
logger.info({
  timestamp: new Date().toISOString(),
  level: 'INFO',
  event: 'patient_created',
  userId: user.id,
  patientId: patient.id,
  duration: 245, // ms
  metadata: { hospital: 'NYC Medical Center' }
});
```

### Metrics Collection
- API response time (p50, p95, p99)
- Error rate (5xx, 4xx)
- Request count (per endpoint)
- Database query time
- Cache hit rate

### Distributed Tracing (optional)
- OpenTelemetry SDK
- Trace correlation across frontend/backend
- Performance bottleneck identification

---

## 9. SECURITY CONSIDERATIONS

### Network Security
- HTTPS only
- CORS whitelist
- API rate limiting (100 req/min per user)

### Data Security
- Field-level encryption (for PII in HIPAA mode)
- Parameterized queries (prevent SQL injection)
- Input validation (XSS prevention)
- CSRF tokens for state-changing operations

### Access Control
- JWT with short expiration (15 min)
- Refresh tokens (long-lived, rotated)
- Role-based permissions (clinician, billing, admin)
- Row-level security (users see only their data)

### Secrets Management
- Environment variables
- No hardcoded secrets in code
- Rotate API keys regularly
- Use secret manager (AWS Secrets Manager, Azure Key Vault)

---

## 10. KEY DESIGN DECISIONS

### 10.1 Polling vs WebSocket
**Decision**: HTTP polling (15-second interval), not WebSocket.
**Rationale**: Polling is simpler, works behind all proxies/firewalls, deployable on static hosts and serverless. WebSocket requires persistent connections (not serverless-friendly) and adds complexity. Polling is sufficient for the clinical rounding use case where real-time sub-second updates are not required.

### 10.2 Frontend Framework
**Decision**: Deferred -- vanilla JS initially, framework optional later.
**Rationale**: Vanilla JS with ES modules keeps the build pipeline minimal and deployment trivial (any static host, no Node build step). A framework can be introduced later if component complexity warrants it. Starting simple reduces time-to-launch.

### 10.3 ORM vs Query Builder
**Decision**: Query builder (Knex.js) preferred over full ORM.
**Rationale**: ORMs like Prisma or TypeORM add abstraction that can hide database-specific behavior. Knex.js provides enough abstraction for portability while keeping queries visible and debuggable. Lighter dependency footprint.

### 10.4 Monorepo vs Multi-repo
**Decision**: Monorepo (`/frontend` + `/backend` in same repo).
**Rationale**: Simpler for a small team, atomic commits across frontend and backend, single CI/CD pipeline. Can split later if team grows.

