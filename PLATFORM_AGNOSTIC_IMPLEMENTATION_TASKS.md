# Platform-Agnostic Architecture: Implementation Tasks

**Document Version**: 1.0  
**Date**: 2026-03-24  
**Status**: Planning Phase - DO NOT BUILD

---

## Overview

Detailed task breakdown for a **greenfield build** of the platform-agnostic Clinical Roundup system. This is a new codebase informed by the existing SPA's features, not a migration. Tasks organized by phase with dependencies and estimated effort.

**Legend**:
- ⏱️ Time estimate (in days)
- 🔗 Dependencies
- 👥 Team size (1-3 people)

---

## PHASE 0: PLANNING & SETUP (2-3 days)

### Task 0.1: Repository Structure Setup
**Description**: Create new directory structure for platform-agnostic codebase  
**Deliverable**: New repo layout ready for development  
**Effort**: ⏱️ 0.5 days  

**Sub-tasks**:
- [ ] Create `/backend` directory
- [ ] Create `/frontend` directory (move from root HTML)
- [ ] Create `/database-migrations` directory
- [ ] Create `/docker` directory with Dockerfile
- [ ] Create `/docs` directory for API docs, deployment guides
- [ ] Create `.env.example` file with all configuration options
- [ ] Create `docker-compose.yml` for local development (PostgreSQL, MongoDB)
- [ ] Update root `.gitignore` to exclude `.env`, node_modules, build artifacts

**Files Created**:
```
.
├── backend/
│  ├── src/
│  ├── tests/
│  ├── migrations/
│  ├── package.json
│  └── Dockerfile
├── frontend/
│  ├── src/
│  ├── public/
│  └── package.json
├── database-migrations/
│  ├── postgres/
│  └── mongo/
├── docs/
│  ├── API.md
│  ├── DEPLOYMENT.md
│  └── DEVELOPMENT.md
├── docker/
│  └── docker-compose.yml
├── .env.example
└── README.md
```

---

### Task 0.2: Project Documentation
**Description**: Create comprehensive development guides  
**Deliverable**: README, setup guide, architecture decision log  
**Effort**: ⏱️ 0.5 days  

**Sub-tasks**:
- [ ] Create `README.md` explaining platform-agnostic architecture
- [ ] Create `DEVELOPMENT.md` for local setup (Docker, environment vars)
- [ ] Create `CONTRIBUTING.md` with coding standards
- [ ] Create `ARCHITECTURE_DECISIONS.md` (ADR format) documenting major choices
- [ ] Create `TROUBLESHOOTING.md` for common issues

---

### Task 0.3: CI/CD Pipeline Setup
**Description**: Configure automated testing and deployment  
**Deliverable**: GitHub Actions workflow  
**Effort**: ⏱️ 1 day  

**Sub-tasks**:
- [ ] Create `.github/workflows/test.yml` (runs on PR)
  - Lint code (ESLint, Prettier)
  - Run unit tests
  - Run integration tests (PostgreSQL, MongoDB)
  - Report coverage
- [ ] Create `.github/workflows/build.yml` (on merge to main)
  - Build Docker image
  - Run full test suite
  - Push to Docker registry (optional)
- [ ] Create `.github/workflows/deploy.yml` (on release tag)
  - Deploy to staging
  - Run smoke tests
  - Deploy to production (with manual approval)

---

### Task 0.4: Development Environment Setup
**Description**: Prepare local development tools and configuration  
**Deliverable**: docker-compose.yml, local setup scripts  
**Effort**: ⏱️ 0.5 days  

**Sub-tasks**:
- [ ] Create `docker-compose.yml` with:
  - PostgreSQL container (port 5432)
  - MongoDB container (port 27017)
  - Redis container for caching (port 6379, optional)
- [ ] Create `docker-compose.override.yml` for local development with volume mounts
- [ ] Create `scripts/setup-local.sh` that:
  - Runs `docker-compose up`
  - Seeds test data
  - Creates initial user accounts
- [ ] Create `scripts/reset-db.sh` to wipe and reinitialize databases

---

## PHASE 1: BACKEND FOUNDATION (7-10 days)

### Task 1.1: Initialize Backend Project
**Description**: Set up Node.js/Express project with dependencies  
**Deliverable**: Working Express server with health check endpoint  
**Effort**: ⏱️ 1 day  
**Dependencies**: Task 0.1  

**Sub-tasks**:
- [ ] Create `backend/package.json` with core dependencies:
  - `express` - HTTP framework
  - `joi` - Input validation
  - `jsonwebtoken` - JWT handling
  - `bcrypt` - Password hashing
  - `cors` - CORS handling
  - `dotenv` - Environment variable loading
  - `winston` - Logging
  - `uuid` - UUID generation
- [ ] Create `backend/src/app.js` with:
  - Express app initialization
  - Middleware stack (CORS, logging, error handling)
  - Health check endpoint (`GET /health`)
- [ ] Create `backend/.env.example`
- [ ] Create `backend/Dockerfile`
- [ ] Verify server starts: `npm start` → listens on port 3000

**Test**:
```bash
curl http://localhost:3000/health
# Expected: { "status": "ok" }
```

---

### Task 1.2: Database Abstraction Layer (PostgreSQL)
**Description**: Implement swappable database adapter pattern  
**Deliverable**: Working PostgreSQL adapter for all patient operations  
**Effort**: ⏱️ 3 days  
**Dependencies**: Task 1.1  

**Sub-tasks**:
- [ ] Install `pg` and `knex.js` (query builder for database abstraction)
- [ ] Create `backend/src/data-access/abstract-base.js`
  - Base class with abstract methods
  - Method signatures for: create, read, update, delete, list, count, filter
  - Error handling standard
- [ ] Create `backend/src/data-access/postgres-adapter.js`
  - Extend AbstractBase
  - Implement all patient CRUD operations
  - Use Knex.js for query building
  - Connection pooling configuration
  - Transaction support
- [ ] Create `backend/src/data-access/index.js`
  - Runtime adapter selection based on `DATABASE_TYPE`
  - Singleton pattern (one connection pool per process)
  - Connection pooling (20 max connections)
- [ ] Create database initialization script
  - `backend/scripts/init-postgres.js`
  - Creates tables, indexes, constraints
- [ ] Create mock/test adapter
  - `backend/src/data-access/memory-adapter.js`
  - In-memory storage for unit tests (no database needed)

**Database Schema** (PostgreSQL):
```sql
-- Run via init-postgres.js
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE patients (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  visit_key VARCHAR(255) UNIQUE NOT NULL,
  mrn VARCHAR(50) NOT NULL,
  name VARCHAR(255) NOT NULL,
  dob DATE NOT NULL,
  room VARCHAR(50),
  hospital VARCHAR(255),
  supervising_md VARCHAR(255),
  procedure_status VARCHAR(50) DEFAULT 'To-Do',
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
  created_at TIMESTAMP DEFAULT NOW(),
  created_by VARCHAR(255),
  updated_at TIMESTAMP DEFAULT NOW(),
  updated_by VARCHAR(255),
  deleted_at TIMESTAMP,
  INDEX(mrn),
  INDEX(visit_key),
  INDEX(created_at),
  INDEX(deleted_at)
);

CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  timestamp TIMESTAMP DEFAULT NOW(),
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

CREATE TABLE settings (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  key VARCHAR(255) UNIQUE NOT NULL,
  value TEXT,
  type VARCHAR(50),
  last_modified TIMESTAMP DEFAULT NOW(),
  modified_by VARCHAR(255)
);
```

**Tests**:
- [ ] Unit tests for PostgreSQL adapter (with real database)
- [ ] CRUD operations for patients
- [ ] Filter/search operations
- [ ] Transaction rollback on error

---

### Task 1.3: Database Abstraction Layer (MongoDB)
**Description**: Implement MongoDB adapter for feature parity  
**Deliverable**: Working MongoDB adapter for all operations  
**Effort**: ⏱️ 2.5 days  
**Dependencies**: Task 1.2  

**Sub-tasks**:
- [ ] Install `mongodb` driver
- [ ] Create `backend/src/data-access/mongo-adapter.js`
  - Extend AbstractBase
  - Implement all CRUD operations using MongoDB API
  - Connection pooling
  - Transaction support (MongoDB 4.0+)
- [ ] Create database initialization script
  - `backend/scripts/init-mongo.js`
  - Creates collections and indexes
- [ ] Create MongoDB schema validation (schema validation feature)

**MongoDB Collections**:
```javascript
{
  _id: ObjectId,
  visitKey: string,
  mrn: string,
  name: string,
  dob: Date,
  room: string,
  hospital: string,
  supervisingMd: string,
  procedureStatus: string,
  plan: string,
  followUp: string,
  cptPrimary: string,
  cptSecondary: [string],
  icdPrimary: string,
  chargeCodesSecondary: string,
  findingsCodes: [string],
  findingsValues: object,
  findingsDates: object,
  findingsText: string,
  createdAt: Date,
  createdBy: string,
  updatedAt: Date,
  updatedBy: string,
  deletedAt: Date | null
}
```

**Tests**:
- [ ] Feature parity tests (same tests as PostgreSQL adapter)
- [ ] MongoDB-specific operations (aggregation pipeline)

---

### Task 1.3b: User Management & Storage
**Description**: Create users table, CRUD operations, and password hashing
**Deliverable**: User storage, creation, lookup, and role assignment
**Effort**: ⏱️ 1 day
**Dependencies**: Task 1.2

**Sub-tasks**:
- [ ] Add `users` table to PostgreSQL schema (id, email, password_hash, display_name, role, is_active, last_login, created_at)
- [ ] Add `users` collection to MongoDB schema
- [ ] Create `backend/src/data-access/user-dao.js`
  - createUser(email, passwordHash, displayName, role)
  - getUserByEmail(email)
  - getUserById(id)
  - updateUser(id, data)
  - deactivateUser(id)
  - listUsers(filters, pagination)
- [ ] Implement password hashing (bcrypt, 12 rounds)
- [ ] Create `backend/scripts/seed-users.js`
  - test-clinician@example.com (clinician)
  - test-billing@example.com (billing)
  - test-admin@example.com (admin)
- [ ] Create `backend/src/routes/users.js`
  - `GET /api/v1/users` (admin only)
  - `POST /api/v1/users` (admin only)
  - `PUT /api/v1/users/:id` (admin only)
  - `DELETE /api/v1/users/:id` (admin only, soft deactivate)

**Tests**:
- [ ] Create user with hashed password
- [ ] Lookup user by email
- [ ] Deactivate user
- [ ] Admin-only access enforcement

---

### Task 1.4: Authentication Service (JWT)
**Description**: Implement JWT-based authentication  
**Deliverable**: Login endpoint, token validation middleware  
**Effort**: ⏱️ 2 days  
**Dependencies**: Task 1.1  

**Sub-tasks**:
- [ ] Create `backend/src/services/auth-service.js`
  - Login (username/password → JWT)
  - Token validation (verify signature, expiration)
  - Refresh token logic
  - Logout (blacklist support)
- [ ] Create `backend/src/middleware/auth.js`
  - Extract token from Authorization header
  - Validate token
  - Inject user info into request
  - Return 401 on invalid token
- [ ] Create `backend/src/routes/auth.js`
  - `POST /api/v1/auth/login` (username, password)
  - `POST /api/v1/auth/refresh` (refresh_token)
  - `POST /api/v1/auth/logout` (invalidate token)
  - `GET /api/v1/auth/me` (current user info)
- [ ] Create test fixtures (test users with passwords)

**Endpoints**:
```
POST /api/v1/auth/login
{
  "email": "user@example.com",
  "password": "password123"
}
→ {
  "access_token": "eyJhbGc...",
  "refresh_token": "eyJhbGc...",
  "expires_in": 900,
  "user": { "id", "email", "roles" }
}

POST /api/v1/auth/refresh
{
  "refresh_token": "eyJhbGc..."
}
→ {
  "access_token": "eyJhbGc...",
  "expires_in": 900
}

GET /api/v1/auth/me
Authorization: Bearer eyJhbGc...
→ {
  "id": "user-id",
  "email": "user@example.com",
  "roles": ["clinician"]
}
```

**Tests**:
- [ ] Login with correct credentials
- [ ] Login with incorrect credentials (401)
- [ ] Token validation (valid, expired, invalid signature)
- [ ] Refresh token logic
- [ ] Missing Authorization header (401)

---

### Task 1.5: Authorization Service (RBAC)
**Description**: Implement role-based access control  
**Deliverable**: Permission checking middleware, role definitions  
**Effort**: ⏱️ 1.5 days  
**Dependencies**: Task 1.4  

**Sub-tasks**:
- [ ] Create `backend/src/services/rbac-service.js`
  - Role definitions (clinician, billing, admin)
  - Permission matrix
  - Check if user has permission
- [ ] Create `backend/src/middleware/rbac.js`
  - Middleware factory: `requirePermission('read_patients')`
  - Check user roles against required permission
  - Return 403 if insufficient permission
- [ ] Create role seeding script
  - `backend/scripts/seed-roles.js`
  - Default users: test-clinician, test-billing, test-admin
- [ ] Integrate with patient routes
  - GET /patients → requires 'read_patients'
  - POST /patients → requires 'create_patient'
  - PUT /patients/:id → requires 'edit_patient'
  - DELETE /patients/:id → requires 'delete_patient'

**Role Matrix**:
```
Clinician:
  - read_patients
  - create_patient
  - edit_own_patient
  - view_findings
  - export_patients

Billing:
  - read_patients
  - view_billing_codes
  - edit_billing_codes
  - export_financial
  - view_audit_logs (limited)

Admin:
  - read_patients
  - create_patient
  - edit_patient
  - delete_patient
  - manage_users
  - export_financial
  - view_audit_logs (full)
```

**Tests**:
- [ ] Clinician can create patient
- [ ] Clinician cannot delete patient (403)
- [ ] Billing user cannot access clinical data
- [ ] Admin can do everything
- [ ] Missing role returns 403

---

### Task 1.6: Audit Logging Service
**Description**: Implement comprehensive audit trail  
**Deliverable**: Audit logger that logs all operations  
**Effort**: ⏱️ 1.5 days  
**Dependencies**: Task 1.2, Task 1.4  

**Sub-tasks**:
- [ ] Create `backend/src/services/audit-service.js`
  - Log CREATE (what was created)
  - Log READ (who viewed what)
  - Log UPDATE (old value, new value, which fields)
  - Log DELETE (what was deleted)
  - Log EXPORT (what was exported, where)
  - Log LOGIN/LOGOUT
  - Structured logging format (JSON)
- [ ] Create `backend/src/routes/audit.js`
  - `GET /api/v1/audit/logs` (list with filters)
  - `GET /api/v1/audit/logs/:patientId` (patient-specific logs)
  - Filters: userId, action, startDate, endDate
  - Pagination support
- [ ] Integrate audit logging into all patient endpoints
  - Log on CREATE
  - Log on READ (optional, can be verbose)
  - Log on UPDATE
  - Log on DELETE
  - Log on EXPORT
- [ ] Create test fixtures with audit logs

**Audit Log Schema**:
```javascript
{
  id: UUID,
  timestamp: ISO8601,
  userId: string,
  action: 'CREATE|READ|UPDATE|DELETE|EXPORT|LOGIN|LOGOUT',
  actionType: 'patient_access|data_change|export|system',
  affectedRecordId: string,
  affectedRecordType: 'patient|setting|audit_log',
  fieldChanged: string, // for UPDATE
  oldValue: any,        // for UPDATE
  newValue: any,        // for UPDATE
  details: {            // context-specific
    ip: string,
    userAgent: string
  }
}
```

**Tests**:
- [ ] Create patient logs correctly
- [ ] Update logs capture old/new values
- [ ] Delete logs capture deleted data
- [ ] Filter logs by patient
- [ ] Filter logs by action
- [ ] Pagination works

---

## PHASE 2: FRONTEND BUILD (5-7 days)

### Task 2.0: Frontend Architecture Decision
**Description**: Choose frontend approach (vanilla JS vs framework)
**Deliverable**: Documented decision, project scaffolding
**Effort**: ⏱️ 0.5 days
**Dependencies**: Phase 1 (backend complete)

**Decision required**:
- **Option A**: Vanilla JS (no build step, simplest deployment, closest to existing SPA's patterns)
- **Option B**: React/Vue/Svelte (component structure, state management, routing, but adds build tooling)
- **Recommendation**: Vanilla JS with ES modules for simplicity; framework is optional later

**Sub-tasks**:
- [ ] Document decision in `ARCHITECTURE_DECISIONS.md`
- [ ] Scaffold `/frontend` directory accordingly
- [ ] If framework chosen: initialize with Vite (fast, framework-agnostic bundler)
- [ ] If vanilla: create module structure (`/frontend/src/api/`, `/frontend/src/ui/`, `/frontend/src/utils/`)

---

### Task 2.1: API Client Layer
**Description**: Build the HTTP client that talks to the backend REST API
**Deliverable**: API client with auth token injection, error handling
**Effort**: ⏱️ 1 day
**Dependencies**: Task 2.0

**Sub-tasks**:
- [ ] Create `frontend/src/api/client.js`
  - Fetch/Axios wrapper with default headers
  - API_BASE_URL from config (not hardcoded)
  - Automatic Bearer token injection from localStorage
  - Error handling (401 → redirect to login, 403 → show message, 500 → toast)
  - Request/response logging (debug mode)
- [ ] Create `frontend/src/config.js`
  - Load API URL, feature flags from runtime config
  - No build-time baking of environment variables

**API Client**:
```javascript
// frontend/src/api/client.js
const API_BASE = window.__APP_CONFIG__?.apiUrl || 'http://localhost:3000/api/v1';

async function apiFetch(path, options = {}) {
  const token = localStorage.getItem('access_token');
  const headers = {
    'Content-Type': 'application/json',
    ...(token ? { 'Authorization': `Bearer ${token}` } : {}),
    ...options.headers,
  };

  const response = await fetch(`${API_BASE}${path}`, { ...options, headers });

  if (response.status === 401) {
    localStorage.removeItem('access_token');
    window.location.href = '/login.html';
    return;
  }

  if (!response.ok) {
    const error = await response.json().catch(() => ({ message: 'Unknown error' }));
    throw new Error(error.error?.message || error.message || `HTTP ${response.status}`);
  }

  return response.status === 204 ? null : response.json();
}

export default apiFetch;
```

**Tests**:
- [ ] All API calls use REST endpoints
- [ ] Token injection works
- [ ] 401 redirects to login
- [ ] Config is runtime-loaded, not build-time

---

### Task 2.2: Build Frontend Authentication
**Description**: JWT-based authentication UI
**Deliverable**: Login form, token storage, session management
**Effort**: ⏱️ 1.5 days
**Dependencies**: Task 1.4, Task 2.1

**Sub-tasks**:
- [ ] Create `frontend/src/api/auth.js`
  - Login (email, password → tokens)
  - Store tokens in localStorage
  - Retrieve tokens
  - Clear tokens (logout)
  - Get current user info
- [ ] Create `frontend/src/pages/Login.js`
  - Email input
  - Password input
  - Login button
  - Error handling
  - Redirect to dashboard on success
- [ ] Create `frontend/src/context/AuthContext.js`
  - Provide user, isAuthenticated, login, logout
  - Initialize from localStorage on app load
- [ ] Update main app to wrap with AuthContext
- [ ] Create `frontend/src/components/PrivateRoute.js`
  - Guard routes that require authentication
  - Redirect to login if not authenticated
- [ ] Create logout functionality
  - Clear tokens from localStorage
  - Redirect to login

**Tests**:
- [ ] Login with valid credentials
- [ ] Login with invalid credentials shows error
- [ ] Token stored in localStorage after login
- [ ] Private routes redirect to login if not authenticated
- [ ] Logout clears tokens and redirects

---

### Task 2.3: Build Patient Management UI
**Description**: Patient CRUD interface using REST API
**Deliverable**: Patient list, detail, create, update, delete views
**Effort**: ⏱️ 2 days
**Dependencies**: Task 1.2, Task 2.1  

**Sub-tasks**:
- [ ] Create `frontend/src/api/patients.js`
  - getPatients(filters)
  - getPatient(id)
  - createPatient(data)
  - updatePatient(id, data)
  - deletePatient(id)
  - archivePatient(id)
  - restorePatient(id)
- [ ] Build patient list view
  - Fetch from REST API via getPatients()
  - Filter by date, hospital, status
  - Sort by room, name, status
- [ ] Build patient detail view
  - Fetch from REST API via getPatient(id)
  - Display all fields (findings, codes, plan, follow-up)
- [ ] Build patient create/edit form
  - POST/PUT to REST endpoints
  - Input validation (client-side)
  - Inline findings entry (CBC, BMP structured fields)
- [ ] Build patient delete/archive flow
  - DELETE to REST endpoint with confirmation dialog
- [ ] Add loading states and error handling

**API Calls**:
```javascript
// frontend/src/api/patients.js
export const getPatients = (filters = {}) => {
  return client.get('/patients', { params: filters });
};

export const getPatient = (id) => {
  return client.get(`/patients/${id}`);
};

export const createPatient = (data) => {
  return client.post('/patients', data);
};

export const updatePatient = (id, data) => {
  return client.put(`/patients/${id}`, data);
};

export const deletePatient = (id) => {
  return client.delete(`/patients/${id}`);
};
```

**Tests**:
- [ ] Patient list loads from API
- [ ] Patient detail loads from API
- [ ] Create patient POSTs to API
- [ ] Update patient PUTs to API
- [ ] Delete patient DELETEs from API
- [ ] Error handling (404, 500, etc.)

---

### Task 2.4: Implement Import/Export
**Description**: Create import/export endpoints and UI  
**Deliverable**: CSV import, Excel/PDF export working  
**Effort**: ⏱️ 1.5 days  
**Dependencies**: Task 1.1, Task 2.1  

**Backend**:
- [ ] Create `backend/src/routes/import-export.js`
  - `POST /api/v1/import/csv` (multipart file upload)
  - `GET /api/v1/export/csv?date=YYYY-MM-DD`
  - `GET /api/v1/export/excel?date=YYYY-MM-DD`
  - `GET /api/v1/export/pdf?ids=id1,id2`
- [ ] Create `backend/src/services/import-service.js`
  - Parse CSV file
  - Validate each row
  - Bulk insert into database
  - Return import summary (created, failed, skipped)
- [ ] Create `backend/src/services/export-service.js`
  - Query patients by date or IDs
  - Format as CSV
  - Generate Excel with formatting
  - Generate PDF with page breaks

**Frontend**:
- [ ] Create `frontend/src/api/importexport.js`
  - importCSV(file)
  - exportCSV(date)
  - exportExcel(date)
  - exportPDF(patientIds)
- [ ] Update UI with import/export buttons
- [ ] File input for CSV import
- [ ] Download handling for exports
- [ ] Progress/status feedback

**Tests**:
- [ ] CSV import with valid data
- [ ] CSV import with invalid data (skipped)
- [ ] CSV export includes all columns
- [ ] Excel export with formatting
- [ ] PDF export with page breaks

---

### Task 2.5: Audit Log Viewing
**Description**: Create audit log viewer in UI  
**Deliverable**: Audit log page with filtering  
**Effort**: ⏱️ 1 day  
**Dependencies**: Task 1.6, Task 2.1  

**Sub-tasks**:
- [ ] Create `frontend/src/api/audit.js`
  - getAuditLogs(filters)
  - getPatientAuditLogs(patientId)
- [ ] Create `frontend/src/pages/AuditLogs.js`
  - List audit logs in table
  - Filters: date range, user, action
  - Pagination
  - Admin/billing users only
- [ ] Add audit log viewer to patient detail page
  - Show all changes to this patient
- [ ] Format log display (readable)
  - Timestamp, user, action, what changed

**Tests**:
- [ ] Audit logs load correctly
- [ ] Filters work
- [ ] Only admin/billing can view logs
- [ ] Patient audit logs show only relevant entries

---

## PHASE 3: API ENDPOINTS (4-5 days)

### Task 3.1: Patient Endpoints
**Description**: Implement all patient CRUD endpoints  
**Deliverable**: All patient operations via REST  
**Effort**: ⏱️ 2 days  
**Dependencies**: Task 1.2, Task 1.5  

**Endpoints**:
```
GET    /api/v1/patients
POST   /api/v1/patients
GET    /api/v1/patients/:id
PUT    /api/v1/patients/:id
DELETE /api/v1/patients/:id
POST   /api/v1/patients/:id/archive
POST   /api/v1/patients/:id/restore
```

**Sub-tasks**:
- [ ] Create `backend/src/routes/patients.js`
- [ ] Implement GET /patients
  - Filters: date, hospital, status
  - Pagination: limit, offset
  - Sorting: by date, by name, by status
  - RBAC: read_patients permission
- [ ] Implement POST /patients
  - Input validation (Joi schema)
  - RBAC: create_patient permission
  - Audit logging
- [ ] Implement GET /patients/:id
  - RBAC: read_patients permission
  - Audit logging (optional)
- [ ] Implement PUT /patients/:id
  - Input validation
  - RBAC: edit_patient permission
  - Audit logging (field changes)
  - Conflict detection (lastUpdated timestamp)
- [ ] Implement DELETE /patients/:id
  - RBAC: delete_patient permission
  - Audit logging
  - Soft delete (set deleted_at) by default
- [ ] Implement POST /patients/:id/archive
- [ ] Implement POST /patients/:id/restore

**Validation Schema** (Joi):
```javascript
const patientSchema = Joi.object({
  name: Joi.string().required(),
  dob: Joi.date().required(),
  mrn: Joi.string().required(),
  room: Joi.string(),
  hospital: Joi.string(),
  supervisingMd: Joi.string(),
  procedureStatus: Joi.string().valid('To-Do', 'In Progress', 'Complete', 'Pending', 'Follow-Up'),
  plan: Joi.string(),
  followUp: Joi.string(),
  cptPrimary: Joi.string(),
  // ... more fields
});
```

**Tests**:
- [ ] CRUD operations
- [ ] Validation errors (400)
- [ ] Permission checks (403)
- [ ] Not found (404)
- [ ] Conflict on update (409)
- [ ] Audit logs created

---

### Task 3.2: Settings Endpoints
**Description**: System settings API  
**Deliverable**: GET/PUT settings endpoints  
**Effort**: ⏱️ 0.5 days  
**Dependencies**: Task 1.2, Task 1.5  

**Endpoints**:
```
GET  /api/v1/settings
PUT  /api/v1/settings
```

**Sub-tasks**:
- [ ] Create `backend/src/routes/settings.js`
- [ ] GET /settings → returns all settings (admin only)
- [ ] PUT /settings → updates settings (admin only)
- [ ] Audit logging for changes

---

### Task 3.3: On-Call Schedule Endpoints
**Description**: On-call scheduling API  
**Deliverable**: CRUD for on-call schedules  
**Effort**: ⏱️ 1 day  
**Dependencies**: Task 1.2, Task 1.5  

**Endpoints**:
```
GET    /api/v1/oncall
POST   /api/v1/oncall
GET    /api/v1/oncall/:id
PUT    /api/v1/oncall/:id
DELETE /api/v1/oncall/:id
```

**Sub-tasks**:
- [ ] Create `backend/src/routes/oncall.js`
- [ ] Implement GET /oncall
  - Filters: startDate, endDate, provider
  - Pagination: limit, offset
  - RBAC: read_patients permission (all clinical users)
- [ ] Implement POST /oncall
  - Input validation (date, provider, hospitals required)
  - RBAC: admin or clinician
  - Unique constraint: one provider per date (or allow multiple)
  - Audit logging
- [ ] Implement PUT /oncall/:id
  - Input validation
  - Audit logging
- [ ] Implement DELETE /oncall/:id
  - RBAC: admin only
  - Audit logging

**Schema** (Joi):
```javascript
const onCallSchema = Joi.object({
  date: Joi.date().required(),
  provider: Joi.string().required(),
  hospitals: Joi.array().items(Joi.string()).min(1).required(),
});
```

**Tests**:
- [ ] Create on-call schedule
- [ ] List schedules by date range
- [ ] Update schedule
- [ ] Delete schedule
- [ ] Duplicate date+provider returns 409
- [ ] Non-admin cannot delete (403)

---

### Task 3.4: Backfeed Endpoints
**Description**: External data ingestion API (lab results, diagnoses, etc.)
**Deliverable**: Backfeed receive and query endpoints
**Effort**: ⏱️ 1 day
**Dependencies**: Task 1.2, Task 1.5

**Context**: Backfeed is the mechanism for receiving external data (lab results, diagnoses, orders, discharge summaries) and linking them to patient records by MRN. This is a core integration point for clinical workflows.

**Endpoints**:
```
POST   /api/v1/backfeed (receive external feed)
GET    /api/v1/backfeed?mrn=X (query feed history)
GET    /api/v1/backfeed/:id (retrieve single feed record)
```

**Sub-tasks**:
- [ ] Create `backend/src/routes/backfeed.js`
- [ ] Implement POST /backfeed
  - Input validation (mrn, sourceSystem, feedType, payload required)
  - RBAC: admin or system service account
  - Store in backfeed table with status 'pending'
  - Optionally auto-process (link to patient record)
  - Audit logging
- [ ] Implement GET /backfeed
  - Filters: mrn, feedType, status, startDate, endDate
  - Pagination
- [ ] Create `backend/src/services/backfeed-service.js`
  - processBackfeed(feedRecord) -- link payload to patient by MRN
  - markProcessed(feedId)
  - markFailed(feedId, errorMessage)

**Tests**:
- [ ] Receive backfeed record
- [ ] Query by MRN returns correct records
- [ ] Process links to patient
- [ ] Invalid MRN handled gracefully

---

### Task 3.5: Error Handling & Standard Responses
**Description**: Consistent error response format  
**Deliverable**: Global error handler, error response format  
**Effort**: ⏱️ 0.5 days  
**Dependencies**: Task 1.1  

**Standard Response Format**:
```javascript
// Success
{
  "success": true,
  "data": { /* ... */ },
  "meta": { "timestamp": "2026-03-24T10:00:00Z" }
}

// Error
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input",
    "details": [
      { "field": "email", "message": "Email is required" }
    ]
  },
  "meta": { "timestamp": "2026-03-24T10:00:00Z" }
}
```

**Sub-tasks**:
- [ ] Create `backend/src/middleware/error-handler.js`
- [ ] Create `backend/src/utils/error-codes.js`
  - VALIDATION_ERROR
  - UNAUTHORIZED
  - FORBIDDEN
  - NOT_FOUND
  - CONFLICT
  - INTERNAL_ERROR
- [ ] Wrap all route handlers with try-catch
- [ ] Return consistent error responses

---

## PHASE 4: TESTING & QUALITY (3-4 days)

### Task 4.1: Unit Tests
**Description**: Test individual functions in isolation  
**Deliverable**: > 80% code coverage for services  
**Effort**: ⏱️ 1.5 days  
**Dependencies**: Phase 3  

**Sub-tasks**:
- [ ] Test `services/auth-service.js`
  - Token generation
  - Token validation
  - Refresh logic
- [ ] Test `services/audit-service.js`
  - Log creation
  - Field change tracking
- [ ] Test `services/rbac-service.js`
  - Permission checking
  - Role assignment
- [ ] Test `data-access/abstract-base.js`
  - CRUD operations
  - Filtering
  - Pagination
- [ ] Test validation schemas (Joi)
- [ ] Setup test fixtures and mocks

**Tools**: Jest, Sinon for mocks

---

### Task 4.2: Integration Tests
**Description**: Test API endpoints with real database  
**Deliverable**: > 60% coverage for integration points  
**Effort**: ⏱️ 1.5 days  
**Dependencies**: Task 4.1  

**Sub-tasks**:
- [ ] Test patient endpoints (GET, POST, PUT, DELETE)
- [ ] Test with both PostgreSQL and MongoDB
- [ ] Test authentication flow
- [ ] Test RBAC (clinician, billing, admin)
- [ ] Test audit logging
- [ ] Test import/export
- [ ] Test error handling (400, 401, 403, 404, 500)
- [ ] Use test database (Docker containers)

**Tools**: Supertest (HTTP assertions), test databases

---

### Task 4.3: E2E Tests (Optional)
**Description**: Full workflow testing  
**Deliverable**: Critical user journeys tested  
**Effort**: ⏱️ 1 day  
**Dependencies**: Phase 2, Phase 3  

**Scenarios**:
- [ ] Login → Create patient → View list → Update → Delete
- [ ] Login → Import CSV → Verify data → Export Excel
- [ ] Login → Create patient → View audit log → Confirm changes logged

**Tools**: Cypress or Playwright

---

### Task 4.4: Code Quality & Linting
**Description**: Enforce code standards  
**Deliverable**: ESLint/Prettier configured, passing checks  
**Effort**: ⏱️ 0.5 days  
**Dependencies**: Phase 3  

**Sub-tasks**:
- [ ] Configure ESLint (airbnb config)
- [ ] Configure Prettier
- [ ] Add pre-commit hooks (husky)
- [ ] Add to CI/CD pipeline
- [ ] Fix existing code to pass

---

## PHASE 5: DOCUMENTATION (2-3 days)

### Task 5.1: API Documentation
**Description**: OpenAPI/Swagger specification  
**Deliverable**: Complete API docs at `/api/docs`  
**Effort**: ⏱️ 1 day  
**Dependencies**: Phase 3  

**Sub-tasks**:
- [ ] Install `swagger-ui-express` and `swagger-jsdoc`
- [ ] Write OpenAPI spec in YAML
- [ ] Document all endpoints with examples
- [ ] Document error responses
- [ ] Document authentication
- [ ] Serve at `/api/docs`
- [ ] Generate HTML/JSON

---

### Task 5.2: Deployment Guides
**Description**: How to deploy on different platforms  
**Deliverable**: Step-by-step guides for AWS, Azure, GCP, on-premises  
**Effort**: ⏱️ 1 day  
**Dependencies**: All phases  

**Guides to Create**:
- [ ] `docs/DEPLOY_AWS.md` (ECS Fargate + RDS PostgreSQL)
- [ ] `docs/DEPLOY_AZURE.md` (App Service + SQL Database)
- [ ] `docs/DEPLOY_GCP.md` (Cloud Run + Cloud SQL)
- [ ] `docs/DEPLOY_HEROKU.md` (simple deployment)
- [ ] `docs/DEPLOY_ONPREMISES.md` (Docker + Linux VM)

**Each Guide Should Include**:
- Infrastructure setup
- Environment variables
- Database initialization
- CI/CD setup
- Monitoring setup
- Troubleshooting

---

### Task 5.3: Developer Documentation
**Description**: How to develop locally  
**Deliverable**: Complete local setup guide  
**Effort**: ⏱️ 0.5 days  
**Dependencies**: Task 0.1  

**Content**:
- [ ] `docs/LOCAL_SETUP.md`
  - Prerequisites (Docker, Node, Git)
  - Clone repo
  - `docker-compose up`
  - Run migrations
  - Seed test data
  - Start backend (`npm run dev`)
  - Start frontend (`npm run dev`)
- [ ] `docs/TROUBLESHOOTING.md`
  - Common issues and fixes
  - How to reset databases
  - How to clear cache
- [ ] `docs/CODE_STRUCTURE.md`
  - Directory layout explained
  - Where to add new features
  - File naming conventions

---

## PHASE 6: DATA SEEDING & LAUNCH (2-3 days)

### Task 6.1: Data Seeding & Import Tools
**Description**: Build tooling for initial data population
**Deliverable**: Scripts to seed users, settings, and import patient data
**Effort**: ⏱️ 1 day
**Dependencies**: Task 1.2, Task 1.3b

**Sub-tasks**:
- [ ] Create `scripts/seed-database.js`
  - Create default admin user
  - Seed system settings (hospitals, procedure types, compliance mode)
  - Seed sample on-call schedules (optional, for demo)
- [ ] Create `scripts/import-patients-csv.js`
  - Import patient records from CSV/Excel
  - Validate each row against schema
  - Report: created, skipped, failed
- [ ] Create `scripts/import-users-csv.js`
  - Bulk create users from CSV
  - Auto-generate temporary passwords
  - Assign roles
- [ ] Document all seed/import scripts in `docs/DATA_IMPORT.md`

**Validation Checklist**:
- [ ] Admin user can log in after seeding
- [ ] Settings load correctly in frontend
- [ ] CSV import handles edge cases (missing fields, duplicates)
- [ ] Import summary report is accurate

---

### Task 6.2: Staging Environment & Smoke Testing
**Description**: Deploy to staging, run full smoke test suite
**Deliverable**: Staging environment verified and functional
**Effort**: ⏱️ 1 day
**Dependencies**: All prior phases

**Sub-tasks**:
- [ ] Deploy backend to staging (Docker or serverless)
- [ ] Deploy frontend to staging (static host)
- [ ] Seed staging database with test data
- [ ] Run E2E smoke tests against staging
- [ ] Verify all CRUD operations
- [ ] Verify auth flow (login, token refresh, logout)
- [ ] Verify RBAC (clinician vs billing vs admin)
- [ ] Verify import/export
- [ ] Verify audit logs
- [ ] Performance baseline (response times under load)

---

### Task 6.3: Production Deployment & Go-Live
**Description**: Deploy to production, onboard users
**Deliverable**: System live and accessible
**Effort**: ⏱️ 1 day
**Dependencies**: Task 6.2

**Sub-tasks**:
- [ ] Final code review
- [ ] Deploy backend to production
- [ ] Deploy frontend to production
- [ ] Run seed script (admin user, settings)
- [ ] Configure DNS / custom domain
- [ ] Verify health checks
- [ ] Create initial user accounts
- [ ] Notify users and share access
- [ ] Monitor first 24 hours (error rates, response times)
- [ ] Document known limitations and post-launch roadmap

---

## PHASE 7: OPTIMIZATION & HARDENING (1-2 days - Optional)

### Task 7.1: Performance Optimization
**Effort**: ⏱️ 1 day  

**Sub-tasks**:
- [ ] Database query optimization (indices, explain plans)
- [ ] Caching layer (Redis for frequently accessed data)
- [ ] API response compression (gzip)
- [ ] Frontend lazy loading
- [ ] CDN for static assets

---

### Task 7.2: Security Hardening
**Effort**: ⏱️ 1 day  

**Sub-tasks**:
- [ ] Rate limiting per IP
- [ ] CORS whitelist review
- [ ] HTTPS certificates (Let's Encrypt)
- [ ] Security headers (CSP, HSTS, X-Frame-Options)
- [ ] SQL injection prevention (already using parameterized queries)
- [ ] XSS prevention (input sanitization)
- [ ] Secrets management (no hardcoded keys)
- [ ] Penetration testing (optional)

---

## SUMMARY TIMELINE

```
Phase 0: Planning & Setup          2-3 days
Phase 1: Backend Foundation        8-11 days (includes user management)
Phase 2: Frontend Build            5-7 days
Phase 3: API Endpoints             5-6 days (includes backfeed)
Phase 4: Testing & Quality         3-4 days
Phase 5: Documentation             2-3 days
Phase 6: Data Seeding & Launch     2-3 days
Phase 7: Optimization (optional)   1-2 days

TOTAL:   28-39 days (6-8 weeks with 1 full-time engineer)
         15-22 days (2 engineers working in parallel)
```

---

## Critical Path

**Minimal viable order** (MVP launch):
1. Task 0.1: Repo structure
2. Task 1.1-1.5 + 1.3b: Backend core + user management
3. Task 2.0-2.3: Frontend build (API client, auth, patient UI)
4. Task 3.1: Patient endpoints
5. Task 4.2: Integration tests
6. Task 5.1: API docs
7. Task 6.1-6.2: Data seeding + staging
8. Task 6.3: Go-live

**Minimum effort**: ~22 days, 1 engineer

---

## Dependencies Matrix

```
Phase 0 (Planning)
  ↓
Phase 1 (Backend)
  ↓
Phase 2 (Frontend) + Phase 3 (API) [parallel]
  ↓
Phase 4 (Testing)
  ↓
Phase 5 (Documentation)
  ↓
Phase 6 (Data Seeding & Launch)
  ↓
Phase 7 (Optimization) [optional]
```

