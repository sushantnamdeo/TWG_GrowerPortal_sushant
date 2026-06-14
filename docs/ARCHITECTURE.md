# Technical Architecture - TWG Grower Portal

## System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        INTERNET USERS                            │
│              (Growers - External Access)                         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                    HTTP/HTTPS
                           │
          ┌────────────────▼────────────────┐
          │     Azure App Service           │
          │  (React Frontend + ASP.NET API) │
          │  ┌──────────────────────────┐   │
          │  │   React.js Application   │   │
          │  └──────────────────────────┘   │
          │  ┌──────────────────────────┐   │
          │  │  ASP.NET Core 8.0 API    │   │
          │  │  - Authentication        │   │
          │  │  - Business Logic        │   │
          │  │  - File Upload Handler   │   │
          │  └──────────────────────────┘   │
          └────────────────┬────────────────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
    ┌───────▼──────┐ ┌────▼─────┐ ┌─────▼──────┐
    │ Azure SQL DB │ │  Blob    │ │ On-Prem    │
    │ (Primary)    │ │ Storage  │ │ JDE DB     │
    │              │ │ (Certs & │ │ (Read)     │
    │ - Users      │ │  PUR)    │ │            │
    │ - Harvest    │ └──────────┘ └────────────┘
    │ - Config     │
    └──────────────┘
```

## Components

### 1. Frontend (React.js)
- **Framework**: React 18+
- **UI Components**: Material-UI or Bootstrap
- **State Management**: Redux or Context API
- **HTTP Client**: Axios
- **Authentication**: JWT tokens (stored in secure cookies)
- **Features**:
  - Responsive design
  - No license cost
  - Easy to deploy to Azure
  - Modern, maintainable codebase

### 2. Backend API (ASP.NET Core 8.0)
- **Framework**: ASP.NET Core 8.0
- **Database ORM**: Entity Framework Core
- **API**: RESTful API with Swagger/OpenAPI documentation
- **Authentication**: Custom JWT-based authentication
- **Authorization**: Role-based access control (RBAC)
- **File Upload**: Azure Blob Storage integration
- **Logging**: Application Insights
- **Features**:
  - Stateless architecture (scalable)
  - Built-in dependency injection
  - Comprehensive error handling
  - Audit logging

### 3. Azure SQL Database
- **Tier**: Standard or Premium (based on performance needs)
- **Backup**: Automated daily backups
- **Security**: 
  - Firewall rules
  - SSL/TLS encryption in transit
  - Transparent Data Encryption (TDE) at rest
  - Row-Level Security (RLS) for data isolation
- **Tables**: As defined in DATABASE_SCHEMA.md

### 4. Azure Blob Storage
- **Purpose**: Store certificates and PUR documents
- **Organization**: Separate containers for certificates and PURs
- **Naming Convention**: `{growercode}/{harvestyear}/{documenttype}/{filename}`
- **Access**: Private with SAS tokens for time-limited access
- **Backup**: Geo-redundant storage (GRS)

### 5. On-Premises JDE Integration
- **Connection**: Secure VPN or ExpressRoute connection
- **Data Sync**: Daily scheduled sync (Azure Data Factory or Scheduled API calls)
- **Direction**: Read-only from JDE to Azure
- **Sync Data**:
  - GrowerMaster (AddressBook data)
  - BlockMaster (Block data)
  - HarvestMaster base fields

### 6. Azure Key Vault
- **Purpose**: Secure storage of sensitive data
- **Contents**:
  - Database connection strings
  - Blob storage connection strings
  - API keys
  - JDE database credentials

---

## Authentication & Authorization Flow

### Authentication Process

1. **User navigates to portal** (anonymous access)
2. **Login screen displayed** (no Microsoft auth required)
3. **User enters email and password**
4. **Backend validates credentials**:
   - Look up user in Users table
   - Compare entered password with stored hash (bcrypt/PBKDF2)
5. **On success**:
   - Generate JWT token (expires in 24 hours)
   - Return token to frontend
   - Frontend stores in secure HTTP-only cookie
6. **On failure**:
   - Return 401 Unauthorized
   - Display error message

### Authorization Process

1. **JWT token included in every request** (Authorization header)
2. **Backend validates token** on every request
3. **Check user role** (Admin or Grower)
4. **For Grower role**: Check GrowerUserRelationship table
   - Ensure user has access to requested grower data
5. **Admin role** has access to all grower data
6. **Unauthorized access** returns 403 Forbidden

---

## API Endpoints

### Authentication
- `POST /api/auth/login` - Login with email and password
- `POST /api/auth/logout` - Logout
- `POST /api/auth/refresh-token` - Refresh JWT token
- `POST /api/auth/change-password` - Change user password
- `GET /api/auth/current-user` - Get current logged-in user info

### Grower Operations
- `GET /api/growers` - List growers accessible to current user
- `GET /api/growers/{growerCode}/harvests` - Get harvest records for grower
- `GET /api/harvests/{harvestId}/details` - Get harvest details
- `POST /api/harvests/declare-sustainable` - Declare blocks as sustainable
- `POST /api/harvests/declare-non-sustainable` - Declare blocks as non-sustainable
- `POST /api/documents/upload` - Upload certificate or PUR document
- `GET /api/documents/{documentId}/download` - Download document

### Admin Operations
- `GET /api/admin/users` - List all users
- `POST /api/admin/users` - Create new user
- `PUT /api/admin/users/{userId}` - Update user
- `PUT /api/admin/users/{userId}/password` - Reset user password
- `DELETE /api/admin/users/{userId}` - Deactivate user
- `POST /api/admin/grower-relationships` - Create/update grower-user relationship
- `GET /api/admin/harvests` - List all harvests (with filters)
- `POST /api/admin/documents/{documentId}/verify` - Verify document
- `POST /api/admin/documents/{documentId}/reject` - Reject document
- `GET /api/admin/configuration` - Get configuration values
- `PUT /api/admin/configuration` - Update configuration values
- `GET /api/admin/audit-logs` - View audit logs

---

## Deployment Architecture

### Development Environment
```
Local Machine
├── React Dev Server (port 3000)
├── ASP.NET Core Dev Server (port 5000)
└── Local SQL Database (or Azure SQL)
```

### Staging Environment
```
Azure
├── App Service (Staging slot)
├── Azure SQL Database (Staging)
└── Blob Storage (Staging)
```

### Production Environment
```
Azure
├── App Service (Production)
│   ├── Autoscaling (2-10 instances based on load)
│   └── Traffic Manager (if multi-region)
├── Azure SQL Database (Standard/Premium)
│   ├── Geo-redundancy enabled
│   ├── Automated backups
│   └── Firewall rules
├── Blob Storage
│   ├── Geo-redundant storage (GRS)
│   └── CDN for faster access
└── Application Insights (Monitoring & Logging)
```

---

## Security Considerations

### Authentication & Passwords
1. **Password Requirements**:
   - Minimum 12 characters
   - Mix of uppercase, lowercase, numbers, and special characters
   - Not reuse last 5 passwords

2. **Password Storage**:
   - Use bcrypt with salt (cost factor 12)
   - Hash stored in Azure SQL
   - Plaintext never stored or transmitted

3. **Password Reset**:
   - Admin can reset user passwords
   - Users can change own passwords
   - New password immediately replaces old
   - Audit log tracks all changes

### Data Security
1. **In Transit**:
   - HTTPS/TLS 1.2+ only
   - All API calls encrypted
   - Blob Storage SAS tokens with expiry

2. **At Rest**:
   - Azure SQL: Transparent Data Encryption (TDE)
   - Blob Storage: Server-side encryption
   - Key Vault: HSM-backed keys

3. **Access Control**:
   - Row-Level Security (RLS) in SQL
   - API-level authorization checks
   - Firewall rules restrict access
   - VPN/ExpressRoute for JDE connection

### Audit & Compliance
1. **Audit Logging**:
   - All data changes logged
   - User actions tracked
   - Timestamps and user IDs recorded
   - Retention: 2 years

2. **Monitoring**:
   - Application Insights tracks all API calls
   - Performance metrics
   - Error tracking and alerting
   - User activity dashboards

---

## Scalability

### Horizontal Scaling
- **App Service**: Auto-scale to 2-10 instances based on CPU and memory
- **Azure SQL**: Read replicas for reporting queries
- **Blob Storage**: Automatically scales (no configuration needed)

### Performance Optimization
1. **Caching**:
   - Redis cache for frequently accessed data
   - Browser caching for static assets
   - Azure CDN for Blob Storage access

2. **Database Optimization**:
   - Proper indexing (as defined in schema)
   - Connection pooling
   - Query optimization
   - Partitioning for large tables

3. **API Optimization**:
   - Pagination for list endpoints
   - Lazy loading of data
   - Compression of responses

---

## Disaster Recovery

1. **Backup Strategy**:
   - Azure SQL: Automated backups (7 days retained)
   - Blob Storage: GRS replication
   - Configuration: Version control in GitHub

2. **Recovery Procedures**:
   - RTO (Recovery Time Objective): 4 hours
   - RPO (Recovery Point Objective): 1 hour
   - Documented runbooks for common scenarios

3. **High Availability**:
   - App Service: Always On, multiple instances
   - Azure SQL: Premium tier with HA
   - Geographic redundancy

---

## Future Enhancement Possibilities

1. **Mobile App** (iOS/Android)
2. **Advanced Analytics** (Power BI integration)
3. **Multi-language Support**
4. **Two-Factor Authentication** (optional)
5. **Single Sign-On (SSO)** integration
6. **Blockchain** for certificate verification
7. **AI-based** document validation
