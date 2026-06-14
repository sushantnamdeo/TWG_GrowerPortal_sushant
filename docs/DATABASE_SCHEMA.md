# Database Schema - TWG Grower Portal

## Overview
This document defines all tables required for the TWG Grower Portal on Azure SQL Database.

---

## 1. Users Table
Stores authentication credentials and user information.

```sql
CREATE TABLE dbo.Users (
    UserId INT IDENTITY(1,1) PRIMARY KEY,
    Email NVARCHAR(255) NOT NULL UNIQUE,
    PasswordHash NVARCHAR(MAX) NOT NULL,
    Role NVARCHAR(50) NOT NULL,  -- 'Admin' or 'Grower'
    IsActive BIT NOT NULL DEFAULT 1,
    LastPasswordChangedDate DATETIME2 NULL,
    LastLoginDate DATETIME2 NULL,
    LastUpdatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    LastUpdatedTime TIME NOT NULL DEFAULT CONVERT(TIME, GETUTCDATE()),
    ModifiedBy NVARCHAR(255) NOT NULL,
    CreatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

CREATE INDEX IX_Users_Email ON dbo.Users(Email);
CREATE INDEX IX_Users_Role ON dbo.Users(Role);
```

---

## 2. GrowerUserRelationship Table
Links users to grower codes they can access.

```sql
CREATE TABLE dbo.GrowerUserRelationship (
    RelationshipId INT IDENTITY(1,1) PRIMARY KEY,
    GrowerCode NVARCHAR(50) NOT NULL,
    UserEmail NVARCHAR(255) NOT NULL,
    RelationshipStatus NVARCHAR(50) NOT NULL DEFAULT 'Active',  -- 'Active' or 'Inactive'
    LastUpdatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    LastUpdatedTime TIME NOT NULL DEFAULT CONVERT(TIME, GETUTCDATE()),
    ModifiedBy NVARCHAR(255) NOT NULL,
    CreatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    FOREIGN KEY (UserEmail) REFERENCES dbo.Users(Email),
    UNIQUE (GrowerCode, UserEmail)
);

CREATE INDEX IX_GrowerUserRel_GrowerCode ON dbo.GrowerUserRelationship(GrowerCode);
CREATE INDEX IX_GrowerUserRel_UserEmail ON dbo.GrowerUserRelationship(UserEmail);
```

---

## 3. GrowerMaster Table
Stores grower information synced from JDE AddressBook.

```sql
CREATE TABLE dbo.GrowerMaster (
    GrowerId INT IDENTITY(1,1) PRIMARY KEY,
    AddressBookNumber INT NOT NULL UNIQUE,
    GrowerName NVARCHAR(255) NOT NULL,
    GrowerCode NVARCHAR(50) NOT NULL UNIQUE,
    GrowerContactEmail NVARCHAR(255) NULL,
    GrowerStatus NVARCHAR(50) NOT NULL DEFAULT 'Active',  -- 'Active' or 'Inactive'
    JDELastSyncDate DATETIME2 NULL,
    LastUpdatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    LastUpdatedTime TIME NOT NULL DEFAULT CONVERT(TIME, GETUTCDATE()),
    ModifiedBy NVARCHAR(255) NOT NULL,
    CreatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

CREATE INDEX IX_GrowerMaster_GrowerCode ON dbo.GrowerMaster(GrowerCode);
CREATE INDEX IX_GrowerMaster_AddressBook ON dbo.GrowerMaster(AddressBookNumber);
```

---

## 4. BlockMaster Table
Stores block information for each grower.

```sql
CREATE TABLE dbo.BlockMaster (
    BlockId INT IDENTITY(1,1) PRIMARY KEY,
    AddressBookNumber INT NOT NULL,
    GrowerCode NVARCHAR(50) NOT NULL,
    BlockNumber NVARCHAR(50) NOT NULL,
    Variety NVARCHAR(255) NULL,
    District NVARCHAR(100) NULL,
    JDELastSyncDate DATETIME2 NULL,
    LastUpdatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    LastUpdatedTime TIME NOT NULL DEFAULT CONVERT(TIME, GETUTCDATE()),
    ModifiedBy NVARCHAR(255) NOT NULL,
    CreatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    FOREIGN KEY (GrowerCode) REFERENCES dbo.GrowerMaster(GrowerCode),
    UNIQUE (GrowerCode, BlockNumber)
);

CREATE INDEX IX_BlockMaster_GrowerCode ON dbo.BlockMaster(GrowerCode);
CREATE INDEX IX_BlockMaster_BlockNumber ON dbo.BlockMaster(BlockNumber);
```

---

## 5. CertificationAgency Table
Stores certification agency information.

```sql
CREATE TABLE dbo.CertificationAgency (
    AgencyId INT IDENTITY(1,1) PRIMARY KEY,
    AgencyCode NVARCHAR(50) NOT NULL UNIQUE,
    AgencyName NVARCHAR(255) NOT NULL,
    Status NVARCHAR(50) NOT NULL DEFAULT 'Active',
    LastUpdatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    LastUpdatedTime TIME NOT NULL DEFAULT CONVERT(TIME, GETUTCDATE()),
    ModifiedBy NVARCHAR(255) NOT NULL,
    CreatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

CREATE INDEX IX_CertAgency_Code ON dbo.CertificationAgency(AgencyCode);
```

---

## 6. HarvestMaster Table
Stores harvest information for blocks in specific years.

```sql
CREATE TABLE dbo.HarvestMaster (
    HarvestId INT IDENTITY(1,1) PRIMARY KEY,
    AddressBookNumber INT NOT NULL,
    GrowerName NVARCHAR(255) NOT NULL,
    GrowerCode NVARCHAR(50) NOT NULL,
    BlockNumber NVARCHAR(50) NOT NULL,
    HarvestYear INT NOT NULL,
    HarvestStatus NVARCHAR(50) NOT NULL DEFAULT 'Current',  -- 'Current' or 'Closed'
    Variety NVARCHAR(255) NULL,
    ContractedTons DECIMAL(10,2) NULL,
    District NVARCHAR(100) NULL,
    CertificateStatus NVARCHAR(50) NOT NULL DEFAULT 'Pending Declaration',
    -- 'Pending Declaration', 'Non Sustainable', 'Pending Upload', 'Pending Verification', 'Verified', 'Rejected'
    PURStatus NVARCHAR(50) NOT NULL DEFAULT 'Pending Upload',
    -- 'Pending Upload', 'Pending Verification', 'Verified', 'Rejected'
    CertificationAgencyCode NVARCHAR(50) NULL,
    CertificateLink NVARCHAR(MAX) NULL,  -- Azure Blob Storage URL
    PURLink NVARCHAR(MAX) NULL,  -- Azure Blob Storage URL
    JDELastSyncDate DATETIME2 NULL,
    LastUpdatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    LastUpdatedTime TIME NOT NULL DEFAULT CONVERT(TIME, GETUTCDATE()),
    ModifiedBy NVARCHAR(255) NOT NULL,
    CreatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    FOREIGN KEY (GrowerCode) REFERENCES dbo.GrowerMaster(GrowerCode),
    FOREIGN KEY (CertificationAgencyCode) REFERENCES dbo.CertificationAgency(AgencyCode),
    UNIQUE (GrowerCode, BlockNumber, HarvestYear)
);

CREATE INDEX IX_HarvestMaster_GrowerCode ON dbo.HarvestMaster(GrowerCode);
CREATE INDEX IX_HarvestMaster_HarvestYear ON dbo.HarvestMaster(HarvestYear);
CREATE INDEX IX_HarvestMaster_CertStatus ON dbo.HarvestMaster(CertificateStatus);
CREATE INDEX IX_HarvestMaster_PURStatus ON dbo.HarvestMaster(PURStatus);
CREATE INDEX IX_HarvestMaster_BlockNumber ON dbo.HarvestMaster(BlockNumber);
```

---

## 7. DocumentUpload Table
Tracks document uploads for certificates and PUR documents.

```sql
CREATE TABLE dbo.DocumentUpload (
    DocumentId INT IDENTITY(1,1) PRIMARY KEY,
    HarvestId INT NOT NULL,
    DocumentType NVARCHAR(50) NOT NULL,  -- 'Certificate' or 'PUR'
    FileName NVARCHAR(255) NOT NULL,
    BlobStoragePath NVARCHAR(MAX) NOT NULL,
    FileSize BIGINT NOT NULL,
    UploadedBy NVARCHAR(255) NOT NULL,
    UploadedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    VerificationStatus NVARCHAR(50) NOT NULL DEFAULT 'Pending Verification',
    -- 'Pending Verification', 'Verified', 'Rejected'
    VerifiedBy NVARCHAR(255) NULL,
    VerificationDate DATETIME2 NULL,
    RejectionReason NVARCHAR(MAX) NULL,
    LastUpdatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    ModifiedBy NVARCHAR(255) NOT NULL,
    FOREIGN KEY (HarvestId) REFERENCES dbo.HarvestMaster(HarvestId)
);

CREATE INDEX IX_DocUpload_HarvestId ON dbo.DocumentUpload(HarvestId);
CREATE INDEX IX_DocUpload_DocumentType ON dbo.DocumentUpload(DocumentType);
CREATE INDEX IX_DocUpload_VerificationStatus ON dbo.DocumentUpload(VerificationStatus);
```

---

## 8. Configuration Table
Stores system configuration data.

```sql
CREATE TABLE dbo.Configuration (
    ConfigId INT IDENTITY(1,1) PRIMARY KEY,
    ConfigKey NVARCHAR(255) NOT NULL UNIQUE,
    ConfigValue NVARCHAR(MAX) NOT NULL,
    Description NVARCHAR(MAX) NULL,
    LastUpdatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    ModifiedBy NVARCHAR(255) NOT NULL,
    CreatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

-- Insert default configuration values
INSERT INTO dbo.Configuration (ConfigKey, ConfigValue, Description, ModifiedBy)
VALUES 
    ('CurrentHarvestYear', '2026', 'Current harvest year for grower declarations', 'System'),
    ('AdminHarvestYear', '2026', 'Current harvest year visible to admins', 'System'),
    ('MaxFileUploadSize', '52428800', 'Maximum file upload size in bytes (50MB)', 'System'),
    ('AllowedFileTypes', 'pdf,docx,xlsx,jpg,png', 'Comma-separated list of allowed file types', 'System');
```

---

## 9. AuditLog Table
Maintains audit trail of all changes.

```sql
CREATE TABLE dbo.AuditLog (
    AuditId INT IDENTITY(1,1) PRIMARY KEY,
    TableName NVARCHAR(255) NOT NULL,
    RecordId INT NOT NULL,
    ActionType NVARCHAR(50) NOT NULL,  -- 'INSERT', 'UPDATE', 'DELETE'
    ChangedBy NVARCHAR(255) NOT NULL,
    ChangeDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    OldValues NVARCHAR(MAX) NULL,
    NewValues NVARCHAR(MAX) NULL,
    IPAddress NVARCHAR(45) NULL
);

CREATE INDEX IX_AuditLog_ChangeDate ON dbo.AuditLog(ChangeDate);
CREATE INDEX IX_AuditLog_ChangedBy ON dbo.AuditLog(ChangedBy);
```

---

## Key Indexes Summary

| Table | Index | Purpose |
|-------|-------|----------|
| Users | Email | Fast authentication lookup |
| HarvestMaster | GrowerCode, HarvestYear, CertificateStatus | Dashboard filtering |
| DocumentUpload | HarvestId, DocumentType, VerificationStatus | Admin verification workflow |
| GrowerUserRelationship | GrowerCode, UserEmail | Permission checks |
| AuditLog | ChangeDate, ChangedBy | Audit trail queries |

---

## Data Synchronization

### From On-Premises JDE to Azure SQL
1. **GrowerMaster** - Daily sync from JDE AddressBook
2. **BlockMaster** - Daily sync from JDE Block Master
3. **HarvestMaster** (read-only fields) - Daily sync of base data
   - AddressBookNumber, GrowerName, GrowerCode, BlockNumber, HarvestYear, Variety, ContractedTons, District

### Stored Procedures for Sync
- `sp_SyncGrowerMasterFromJDE` - Synchronizes grower data
- `sp_SyncBlockMasterFromJDE` - Synchronizes block data
- `sp_SyncHarvestMasterFromJDE` - Synchronizes harvest base data
