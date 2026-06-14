# TWG Grower Portal - Wine Grower Sustainable Certification Portal

## Overview
A web-based portal for The Wine Group (TWG) growers to declare sustainable certifications, upload certificates and PUR documents, and manage their harvest data.

## Key Features
- Grower authentication without Microsoft credentials
- Role-based access control (Grower & Admin roles)
- Harvest master data management
- Sustainable certification declaration workflow
- Certificate and PUR document upload and verification
- Admin dashboard for certificate verification and rejection
- Integration with on-premises JDE SQL Server database

## Technical Stack
- **Frontend**: React.js (Modern, No License Cost)
- **Backend**: ASP.NET Core 8.0
- **Database**: Azure SQL Database
- **Hosting**: Azure App Service
- **Authentication**: Custom Email/Password (Azure SQL)
- **Storage**: Azure Blob Storage (for certificates and PUR documents)

## Architecture
```
Grower Portal (Azure)
├── Frontend (React.js)
├── Backend API (ASP.NET Core)
├── Azure SQL Database (Primary)
│   ├── Users & Authentication
│   ├── Harvest Master Data (Synced from JDE)
│   ├── Certification Data
│   └── Configuration
└── Azure Blob Storage
    ├── Certificates
    └── PUR Documents

On-Premises JDE
└── Source data sync via secure connection
```

## Project Structure
```
TWG_GrowerPortal_sushant/
├── src/
│   ├── frontend/          # React.js application
│   ├── backend/           # ASP.NET Core API
│   ├── database/          # SQL scripts and migrations
│   └── shared/            # Shared models and constants
├── docs/                  # Documentation
├── deployment/            # Azure deployment configs
└── README.md
```

## Getting Started
See individual component README files in their respective directories.

## Database Schema
See `docs/DATABASE_SCHEMA.md` for complete database design.

## Deployment
See `deployment/AZURE_DEPLOYMENT.md` for Azure deployment instructions.
