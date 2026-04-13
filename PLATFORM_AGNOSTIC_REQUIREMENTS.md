# Platform-Agnostic Architecture: Requirements

**Document Version**: 1.0  
**Date**: 2026-03-24  
**Status**: Planning Phase - DO NOT BUILD

---

## Executive Summary

Greenfield build of a platform-agnostic Clinical Roundup system. Frontend and backend can be deployed to **any** hosting provider with **any** backend database using standard APIs and protocols. This is a new build informed by the existing SPA's feature set, not a migration of the current codebase.

---

## 1. FUNCTIONAL REQUIREMENTS

### 1.1 Patient Management
- [x] Create patient record (room, name, DOB, MRN, hospital, supervising MD)
- [x] Read/retrieve patient records (by date, by MRN, by ID)
- [x] Update patient record (all fields editable)
- [x] Delete/archive patient records
- [x] List patients for a given date
- [x] Filter by hospital
- [x] Filter by status (To-Do, In Progress, Complete)
- [x] Restore archived patients

### 1.2 Clinical Findings
- [x] Associate findings with patient (CBC, BMP codes)
- [x] Store structured lab values (WCC, Hb, Hct, Plt, Na, K, Cl, etc.)
- [x] Support custom finding text (free-form notes)
- [x] Track finding dates
- [x] Parse legacy string formats into structured JSON

### 1.3 Procedure & Billing Codes
- [x] Assign CPT code (primary procedure)
- [x] Assign secondary CPT codes
- [x] Assign ICD codes (diagnoses)
- [x] Store charge codes
- [x] Mask billing data from non-billing users (RBAC)

### 1.4 Status & Plans
- [x] Track procedure status (To-Do, In Progress, Complete, Pending, Follow-Up)
- [x] Document clinical plan
- [x] Document follow-up requirements
- [x] Update status with timestamps

### 1.5 On-Call Scheduling
- [x] Create/update on-call schedule by date
- [x] Assign provider to date
- [x] Assign hospitals to provider
- [x] Retrieve schedule for date range

### 1.6 Settings Management
- [x] Store system settings (hospitals, procedure types, etc.)
- [x] Retrieve settings
- [x] Support feature flags (local mode, compliance modes, etc.)

### 1.7 Import/Export
- [x] Import patients from CSV
- [x] Export patients to Excel
- [x] Export selected patients to PDF
- [x] Preserve formatting across export formats

### 1.8 Audit & Compliance
- [x] Log all CRUD operations with user/timestamp
- [x] Log data access (who viewed what patient)
- [x] Log data changes (field-level change tracking)
- [x] Log exports (what data, where, when, by whom)
- [x] Retrieve audit logs (filtered by patient, user, action, date range)
- [x] Support compliance modes (relaxed, HIPAA strict, SOX strict)
- [x] Field encryption for PII when required
- [x] Data retention policies (mark for deletion, archive schedules)

### 1.9 Authentication & Authorization
- [x] User login (any OAuth2 provider or custom JWT)
- [x] Session management (create, validate, destroy, timeout)
- [x] Role-based access control (Clinician, Billing, Admin)
- [x] Permission enforcement (read_patients, create_patient, edit_patient, delete_patient, view_billing_codes, export_financial, manage_users, view_audit_logs)
- [x] Multi-user concurrent access

### 1.10 UI/UX
- [x] Responsive design (mobile, tablet, desktop)
- [x] Device detection (iOS, Android, Windows, Mac)
- [x] Offline-first capability (localStorage sync)
- [x] Real-time updates (15-second sync polling)
- [x] Toast notifications
- [x] Adaptive UI for screen sizes
- [x] Print-friendly formatting

### 1.11 Data Synchronization
- [x] Two-way sync between client and server
- [x] Conflict resolution (last-write-wins or server-authoritative)
- [x] Sync status indicator
- [x] Offline mode with localStorage persistence
- [x] Background sync when connection restored

---

## 2. NON-FUNCTIONAL REQUIREMENTS

### 2.1 Performance
- [ ] API response time: < 500ms (95th percentile)
- [ ] Page load time: < 2s (first paint)
- [ ] Sync operation: < 30s from client change to server confirmation
- [ ] Support 100+ concurrent users
- [ ] Support 10,000+ patient records without performance degradation

### 2.2 Reliability
- [ ] 99.5% uptime SLA
- [ ] Data consistency across all replicas
- [ ] Graceful degradation (local mode when cloud unavailable)
- [ ] Automatic retry with exponential backoff for failed requests
- [ ] Connection pooling for database

### 2.3 Security
- [ ] HTTPS only for all API communication
- [ ] OAuth2 / OpenID Connect for authentication (provider-agnostic)
- [ ] JWT tokens with refresh token rotation
- [ ] CORS restrictions to trusted origins only
- [ ] Rate limiting (API calls per user per minute)
- [ ] SQL injection prevention (parameterized queries)
- [ ] XSS protection (CSP headers, input sanitization)
- [ ] CSRF tokens for state-changing operations
- [ ] API key rotation support
- [ ] Field-level encryption for PII when compliance mode enabled
- [ ] No hardcoded credentials (environment variables only)

### 2.4 Compliance
- [ ] HIPAA compliance (field encryption, audit logs, access controls)
- [ ] SOX compliance (dual approval workflows, immutable logs)
- [ ] Data retention policies (configurable per record)
- [ ] Export data in standard formats (CSV, Excel, PDF)
- [ ] Audit trail for all operations

### 2.5 Scalability
- [ ] Horizontal scaling (stateless backend)
- [ ] Database replication support
- [ ] CDN-friendly (static assets cacheable)
- [ ] Connection pooling for concurrent requests
- [ ] Bulk operations support (batch insert, batch update)

### 2.6 Maintainability
- [ ] Clear API documentation (OpenAPI/Swagger)
- [ ] Modular code architecture
- [ ] Dependency injection for testability
- [ ] Unit test coverage > 80%
- [ ] Integration test coverage > 60%
- [ ] Logging at all critical points
- [ ] Error tracking (Sentry, DataDog, or similar)
- [ ] Monitoring dashboards (response time, error rate, uptime)

### 2.7 Deployment
- [ ] Containerized (Docker) for consistent deployments
- [ ] Infrastructure-as-code (Terraform, CloudFormation, or similar)
- [ ] CI/CD pipeline (GitHub Actions, GitLab CI, or similar)
- [ ] Database migrations tracked and versioned
- [ ] Rollback capability for failed deployments
- [ ] Environment parity (dev, staging, production)

---

## 3. PLATFORM NEUTRALITY REQUIREMENTS

### 3.1 Frontend Hosting
- [ ] Must work on: GitHub Pages, Vercel, Netlify, AWS S3+CloudFront, Azure Static Web Apps, any static host
- [ ] No vendor-specific APIs in frontend code
- [ ] All backend communication via standard HTTP/REST (no Microsoft Graph API calls from frontend)
- [ ] Configuration via environment variables or config file
- [ ] API endpoint configurable (not hardcoded to SharePoint)

### 3.2 Backend
- [ ] Database agnostic (API layer abstracts from database)
- [ ] Must support: PostgreSQL, MySQL, MongoDB, Firebase Firestore, DynamoDB, Cosmos DB
- [ ] ORM or query builder for database abstraction
- [ ] No vendor-specific query syntax in business logic
- [ ] Environment-specific configuration (connection strings, credentials)

### 3.3 Authentication
- [ ] Support multiple OAuth2 providers (Azure AD, Google, GitHub, Okta, Auth0)
- [ ] Support JWT with custom validation
- [ ] Support SAML (optional but nice-to-have)
- [ ] Authentication provider configurable at runtime

### 3.4 Data Storage
- [ ] No reliance on SharePoint Lists API
- [ ] No reliance on Microsoft Graph API
- [ ] No Azure-specific services in core logic
- [ ] Vendor-specific features optional (nice-to-have, not required)

### 3.5 File Storage
- [ ] Support: Local filesystem, AWS S3, Azure Blob Storage, Google Cloud Storage
- [ ] Abstracted file interface (can swap providers without code changes)

---

## 4. API CONTRACT REQUIREMENTS

### 4.1 REST API Design
- [ ] RESTful endpoints (not RPC)
- [ ] JSON request/response bodies
- [ ] HTTP status codes (200, 201, 400, 401, 403, 404, 409, 500)
- [ ] Standard error response format
- [ ] Pagination (limit, offset) for list endpoints
- [ ] Filtering support (query parameters)
- [ ] Sorting support (query parameters)
- [ ] Versioning strategy (v1, v2, etc.)

### 4.2 Authentication Endpoints
- [ ] POST /api/v1/auth/login
- [ ] POST /api/v1/auth/logout
- [ ] POST /api/v1/auth/refresh
- [ ] GET /api/v1/auth/me (current user info)

### 4.3 Patient Endpoints
- [ ] GET /api/v1/patients (list)
- [ ] GET /api/v1/patients/:id (retrieve)
- [ ] POST /api/v1/patients (create)
- [ ] PUT /api/v1/patients/:id (update)
- [ ] DELETE /api/v1/patients/:id (delete)
- [ ] POST /api/v1/patients/:id/archive (soft delete)
- [ ] POST /api/v1/patients/:id/restore (restore)

### 4.4 Audit Endpoints
- [ ] GET /api/v1/audit/logs (list logs)
- [ ] GET /api/v1/audit/logs/:patientId (logs for patient)
- [ ] GET /api/v1/audit/logs?action=VIEW&startDate=X&endDate=Y (filtered logs)

### 4.5 Settings Endpoints
- [ ] GET /api/v1/settings
- [ ] PUT /api/v1/settings (admin only)

### 4.6 Import/Export Endpoints
- [ ] POST /api/v1/import/csv (bulk import)
- [ ] GET /api/v1/export/csv?date=YYYY-MM-DD (export CSV)
- [ ] GET /api/v1/export/excel?date=YYYY-MM-DD (export Excel)
- [ ] GET /api/v1/export/pdf?ids=X,Y,Z (export PDF)

### 4.7 On-Call Schedule Endpoints
- [ ] GET /api/v1/oncall (list schedules)
- [ ] GET /api/v1/oncall/:id (retrieve)
- [ ] POST /api/v1/oncall (create)
- [ ] PUT /api/v1/oncall/:id (update)
- [ ] DELETE /api/v1/oncall/:id (delete)

### 4.8 User Management Endpoints
- [ ] GET /api/v1/users (list users, admin only)
- [ ] POST /api/v1/users (create user, admin only)
- [ ] PUT /api/v1/users/:id (update user, admin only)
- [ ] DELETE /api/v1/users/:id (deactivate user, admin only)

### 4.9 Backfeed Endpoints
- [ ] POST /api/v1/backfeed (receive external data feed)
- [ ] GET /api/v1/backfeed?mrn=X (retrieve feed history for patient)

---

## 5. DATA MODEL REQUIREMENTS

### 5.1 Patient Record
```
{
  id: UUID,
  visitKey: string (mrn|date),
  mrn: string,
  name: string,
  dob: ISO8601,
  room: string,
  hospital: string,
  supervisingMd: string,
  date: ISO8601,
  
  procedureStatus: enum (To-Do, In Progress, Complete, Pending, Follow-Up),
  plan: text,
  followUp: text,
  
  cptPrimary: string,
  cptSecondary: [string],
  icdPrimary: string,
  chargeCodesSecondary: string,
  
  findingsCodes: [string],
  findingsValues: {[code]: any},
  findingsDates: {[code]: ISO8601},
  findingsText: text,
  
  createdAt: ISO8601,
  createdBy: string,
  updatedAt: ISO8601,
  updatedBy: string,
  deletedAt: ISO8601 (null if not deleted),
  
}
```

### 5.2 Audit Log Record
```
{
  id: UUID,
  timestamp: ISO8601,
  userId: string,
  action: enum (CREATE, READ, UPDATE, DELETE, ARCHIVE, RESTORE, EXPORT, LOGIN, LOGOUT),
  actionType: enum (patient_access, data_change, export, system),
  affectedRecordId: string,
  affectedRecordType: enum (patient, setting, audit_log),
  fieldChanged: string (for UPDATE actions),
  oldValue: any,
  newValue: any,
  details: {[key]: any},
  ipAddress: string,
  userAgent: string,
}
```

### 5.3 Settings Record
```
{
  id: UUID,
  key: string (unique),
  value: any,
  type: enum (string, number, boolean, json),
  lastModified: ISO8601,
  modifiedBy: string,
}
```

### 5.4 User Record
```
{
  id: UUID,
  email: string (unique),
  passwordHash: string,
  displayName: string,
  role: enum (clinician, billing, admin),
  isActive: boolean,
  lastLogin: ISO8601,
  createdAt: ISO8601,
  createdBy: string,
  updatedAt: ISO8601,
}
```

### 5.5 On-Call Schedule Record
```
{
  id: UUID,
  date: ISO8601,
  provider: string,
  hospitals: [string],
  createdAt: ISO8601,
  createdBy: string,
  updatedAt: ISO8601,
  updatedBy: string,
}
```

### 5.6 Backfeed Record
```
{
  id: UUID,
  mrn: string,
  sourceSystem: string,
  feedType: enum (lab_result, diagnosis, order, discharge),
  payload: {[key]: any},
  processedAt: ISO8601,
  status: enum (pending, processed, failed),
  errorMessage: string (nullable),
  createdAt: ISO8601,
}
```

---

## 6. ACCEPTANCE CRITERIA

### 6.1 Deployment
- [ ] Frontend deployable to GitHub Pages without modification
- [ ] Frontend deployable to any cloud provider (Vercel, Netlify, AWS, Azure, GCP)
- [ ] Backend deployable with different databases (PostgreSQL, MongoDB)
- [ ] Backend deployable on different platforms (AWS, Azure, GCP, on-premises)
- [ ] Zero hardcoded vendor-specific code paths in core logic

### 6.2 Configuration
- [ ] All secrets in environment variables (not committed to repo)
- [ ] Database connection string configurable
- [ ] API endpoint configurable
- [ ] Auth provider configurable
- [ ] All feature flags configurable

### 6.3 Testing
- [ ] Backend API tests pass with PostgreSQL
- [ ] Backend API tests pass with MongoDB
- [ ] Frontend loads and works without SharePoint/Microsoft Graph APIs
- [ ] Local mode works without any cloud dependencies

### 6.4 Documentation
- [ ] OpenAPI/Swagger spec for all API endpoints
- [ ] Deployment guide for each supported platform
- [ ] Database schema documentation
- [ ] Architecture decision log (ADRs)
- [ ] Data import guide (for seeding from external sources)

---

## 7. CONSTRAINTS & ASSUMPTIONS

### 7.1 Constraints
- HIPAA compliance from day one
- Feature parity with existing SPA's capabilities (informed by, not migrated from)
- No vendor lock-in in core logic

### 7.2 Assumptions
- Users have access to at least one relational or NoSQL database
- Users have access to hosting (cloud or on-premises)
- Authentication provider (OAuth2 or custom) available
- File storage available (local filesystem or cloud)
- Internet connectivity required (offline mode optional)

---

## 8. OUT OF SCOPE

- Preference card functionality (future phase, see earlier analysis)
- Migration tooling from existing SharePoint-based SPA (separate effort if needed)
- Mobile native apps (web app works on mobile)
- Real-time WebSocket sync (polling acceptable)
- Advanced analytics/reporting (basic export sufficient)
- Multi-tenancy (single tenant per deployment)
- Internationalization (English only initially)

