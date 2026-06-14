# Workflow Specifications - TWG Grower Portal

## Overview
Detailed specifications for Grower and Admin user workflows.

---

## Grower Workflow

### Screen 1: Grower Dashboard

**URL**: `/grower/dashboard`
**Authentication**: Required (Grower or Admin role)

#### Layout
```
┌──────────────────────────────────────────────────────────────┐
│  Welcome, {LoggedInEmail}                     Logout | Help  │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Current Harvest Year: {CurrentHarvestYear}                  │
│                                                               │
│  Select Grower: [────────────────────────────────────────]▼  │
│                 [Search growers by code or name]             │
│                                                               │
├─ HARVEST RECORDS ─────────────────────────────────────────────┤
│                                                               │
│  [Search] [Filter] [Sort]                                    │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Block │ Variety │ Tons │ Cert Status │ PUR Status │ Link│  │
│  ├────────────────────────────────────────────────────────┤  │
│  │ P-5235│ PIG     │ 1080 │ Verified    │ Verified   │ ✓  │  │
│  │ P-6374│ SAB     │ 1200 │ Pending...  │ Verified   │ ✓  │  │
│  │ P-6375│ CAS     │ 1200 │ Non Sust    │ Pending..  │ □  │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
│  [Declare Sustainable] [Declare Non Sustainable] [Upload]   │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

#### Components

1. **Logged-In User Email**
   - Display current user email
   - Top right corner

2. **Logout Button**
   - Clears session
   - Redirects to login

3. **Current Harvest Year Display**
   - Read from Configuration table
   - Not editable by grower
   - Example: "Current Harvest Year: 2026"

4. **Grower Selector Dropdown**
   - Populated from GrowerUserRelationship table
   - Filter: WHERE UserEmail = @currentUserEmail AND RelationshipStatus = 'Active'
   - Show: GrowerCode + GrowerName
   - Has search/filter box to type and filter
   - If user has only one grower → Auto-select
   - If user is Admin → Show all growers (auto-search available)
   - On selection change → Refresh grid below

5. **Harvest Grid**
   - Data source: HarvestMaster table
   - Filters:
     - WHERE GrowerCode = @selectedGrowerCode
     - AND HarvestYear = @currentHarvestYear
   - Columns:
     - **Block** (BlockNumber) - Searchable
     - **Variety** - Searchable
     - **Contracted Tons** - Sortable
     - **District** - Searchable
     - **Certificate Status** - Filterable (dropdown)
     - **PUR Status** - Filterable (dropdown)
     - **Links** - Click to view documents
   - Each column header has search icon → Opens filter
   - Pagination: 25 records per page

6. **Action Buttons**
   - **Declare Sustainable** → Navigate to Screen 2
   - **Declare Non Sustainable** → Navigate to Screen 3
   - **Upload Document** → Modal to select type and upload

---

### Screen 2: Declare Blocks as Sustainable

**URL**: `/grower/declare-sustainable`
**Navigation**: From Dashboard "Declare Sustainable" button

#### Layout
```
┌────────────────────────────────────────────────────────────┐
│ Back to Dashboard                                           │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  Declare Blocks as Sustainable                             │
│  Grower: {SelectedGrowerCode} - {GrowerName}               │
│  Harvest Year: {CurrentHarvestYear}                        │
│                                                             │
│  Certification Agency *                                    │
│  [─────────────────────────────────────────────────────]▼  │
│  [Select agency or search...]                              │
│  - California Sustainable Winegrowing Alliance (CSWA)      │
│  - Lodi Rules (LR)                                         │
│  - Napa Green (NP)                                         │
│  - Fish Friendly Farming (FFF)                             │
│  - Sustainability in Practice (SIP)                        │
│                                                             │
├─ PENDING DECLARATION BLOCKS ──────────────────────────────┤
│  [Search] [Select All] [Clear Selection]                  │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ ☐ │ Block  │ Variety │ Tons   │ District │          │ │
│  ├──────────────────────────────────────────────────────┤ │
│  │ ☑ │ P-5235 │ PIG     │ 1080   │ 12       │          │ │
│  │ ☐ │ P-6375 │ CAS     │ 1200   │ 11       │          │ │
│  │ ☑ │ P-4594 │ FRC     │ 250    │ 13       │          │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                             │
│  Selected Blocks: 2 of 3                                   │
│                                                             │
│  [Cancel] [Declare as Sustainable]                        │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

#### Specifications

1. **Certification Agency Selection**
   - Mandatory field (marked with *)
   - Dropdown from CertificationAgency table
   - Filter: WHERE Status = 'Active'
   - Display: AgencyName (AgencyCode)
   - Searchable
   - Cannot proceed without selection

2. **Blocks Grid**
   - Data source: HarvestMaster
   - Filters:
     - WHERE GrowerCode = @selectedGrowerCode
     - AND HarvestYear = @currentHarvestYear
     - AND CertificateStatus = 'Pending Declaration'
   - Columns: Checkbox, Block, Variety, Tons, District
   - Multiple selection allowed
   - Checkbox for select all
   - Count of selected blocks shown below grid

3. **Declare Button**
   - Only enabled if:
     - At least 1 block selected
     - Certification agency selected
   - On click → Show confirmation dialog

#### Confirmation Dialog

```
┌─────────────────────────────────────────────────────┐
│ Confirm Sustainable Declaration                     │
├─────────────────────────────────────────────────────┤
│                                                      │
│ You are declaring the following blocks to be        │
│ sustainable for certification agency:              │
│ {SelectedAgencyName}                                │
│                                                      │
│ Blocks:                                              │
│ • P-5235 (PIG) - 1080 tons                          │
│ • P-4594 (FRC) - 250 tons                           │
│                                                      │
│ This action cannot be undone.                       │
│ Are you sure?                                       │
│                                                      │
│ [Cancel] [Confirm]                                  │
│                                                      │
└─────────────────────────────────────────────────────┘
```

#### Backend Actions
On confirmation:
1. For each selected block:
   - Update HarvestMaster.CertificateStatus = 'Pending Upload'
   - Update HarvestMaster.CertificationAgencyCode = @selectedAgencyCode
   - Update HarvestMaster.LastUpdatedDate = NOW()
   - Update HarvestMaster.LastUpdatedTime = NOW()
   - Update HarvestMaster.ModifiedBy = @currentUserEmail

2. Log audit record for each block update

3. Show success message: "2 blocks declared as sustainable"

4. Redirect back to dashboard

---

### Screen 3: Declare Blocks as Non Sustainable

**URL**: `/grower/declare-non-sustainable`
**Navigation**: From Dashboard "Declare Non Sustainable" button

#### Layout
```
┌────────────────────────────────────────────────────────────┐
│ Back to Dashboard                                           │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  Declare Blocks as Non Sustainable                         │
│  Grower: {SelectedGrowerCode} - {GrowerName}               │
│  Harvest Year: {CurrentHarvestYear}                        │
│                                                             │
├─ PENDING DECLARATION BLOCKS ──────────────────────────────┤
│  [Search] [Select All] [Clear Selection]                  │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ ☐ │ Block  │ Variety │ Tons   │ District │          │ │
│  ├──────────────────────────────────────────────────────┤ │
│  │ ☑ │ P-5235 │ PIG     │ 1080   │ 12       │          │ │
│  │ ☐ │ P-6375 │ CAS     │ 1200   │ 11       │          │ │
│  │ ☑ │ P-4594 │ FRC     │ 250    │ 13       │          │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                             │
│  Selected Blocks: 2 of 3                                   │
│                                                             │
│  [Cancel] [Declare as Non Sustainable]                    │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

#### Specifications

1. **Blocks Grid**
   - Same as Screen 2, but no Certification Agency selection
   - Data filters same as Screen 2
   - Multiple selection allowed

2. **Declare Button**
   - Only enabled if at least 1 block selected
   - On click → Show confirmation dialog

#### Confirmation Dialog

```
┌─────────────────────────────────────────────────────┐
│ Confirm Non Sustainable Declaration                 │
├─────────────────────────────────────────────────────┤
│                                                      │
│ You are declaring the following blocks to be        │
│ non sustainable.                                    │
│                                                      │
│ Blocks:                                              │
│ • P-5235 (PIG) - 1080 tons                          │
│ • P-4594 (FRC) - 250 tons                           │
│                                                      │
│ This action cannot be undone.                       │
│ Are you sure?                                       │
│                                                      │
│ [Cancel] [Confirm]                                  │
│                                                      │
└─────────────────────────────────────────────────────┘
```

#### Backend Actions
On confirmation:
1. For each selected block:
   - Update HarvestMaster.CertificateStatus = 'Non Sustainable'
   - Clear HarvestMaster.CertificationAgencyCode = NULL
   - Update HarvestMaster.LastUpdatedDate = NOW()
   - Update HarvestMaster.LastUpdatedTime = NOW()
   - Update HarvestMaster.ModifiedBy = @currentUserEmail

2. Log audit record for each block update

3. Show success message: "2 blocks declared as non sustainable"

4. Redirect back to dashboard

---

### Screen 4: Document Upload Modal

**Triggered from**: Dashboard or any view
**Type**: Modal overlay

#### Layout
```
┌─────────────────────────────────────────────────────┐
│ Upload Document                                   ✕ │
├─────────────────────────────────────────────────────┤
│                                                      │
│  Document Type *                                    │
│  ◉ Certificate    ◯ PUR Document                   │
│                                                      │
│  Select Block *                                     │
│  [─────────────────────────────────────────────]▼   │
│  [Search and select block...]                       │
│  - P-5235 (PIG) - Pending Upload                    │
│  - P-6375 (CAS) - Pending Upload                    │
│                                                      │
│  File Upload *                                      │
│  [Browse Files] or Drag & Drop                      │
│  Max size: 50 MB                                    │
│  Allowed: PDF, DOCX, XLSX, JPG, PNG                │
│                                                      │
│  [Cancel] [Upload]                                  │
│                                                      │
└─────────────────────────────────────────────────────┘
```

#### Specifications

1. **Document Type**
   - Radio button selection
   - Options: "Certificate" or "PUR Document"
   - Mandatory field

2. **Block Selection**
   - Dropdown of harvest blocks for selected grower
   - Filter based on document type:
     - If Certificate: Show blocks with CertificateStatus = 'Pending Upload'
     - If PUR: Show all blocks
   - Mandatory field

3. **File Upload**
   - Drag and drop support
   - Browse button to select file
   - Max size: 50 MB (from configuration)
   - Allowed types: PDF, DOCX, XLSX, JPG, PNG
   - Show file size and format validation
   - Mandatory field

4. **Upload Button**
   - Enabled only when all fields populated
   - On click:
     - Upload file to Azure Blob Storage
     - Generate SAS URL
     - Create DocumentUpload record
     - Update HarvestMaster.CertificateLink or .PURLink
     - Show success message
     - Close modal

---

## Admin Workflow

### Screen 1: Admin Dashboard

**URL**: `/admin/dashboard`
**Authentication**: Required (Admin role only)

#### Layout
```
┌──────────────────────────────────────────────────────────────┐
│  Welcome, {LoggedInEmail}  [Configuration] [Users]  Logout   │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Current Harvest Year: {AdminHarvestYear}                    │
│                                                               │
│  Select Grower: [────────────────────────────────────────]▼  │
│                 [Search growers by code or name]             │
│                                                               │
├─ HARVEST RECORDS ─────────────────────────────────────────────┤
│                                                               │
│  [Search] [Filter by Status] [Filter by Agency]              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │Block│Var│Tons│Cert Status│PUR Status│Agency│Cert│PUR│  │
│  ├────────────────────────────────────────────────────────┤  │
│  │P-523│PIG│1080│Verified   │Verified  │CSWA  │✓   │✓  │  │
│  │P-637│SAB│1200│Pending... │Verified  │LR    │📄  │✓  │  │
│  │P-637│CAS│1200│Non Sust   │Pending...|      │    │📄  │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

#### Features

1. **Navigation Links**
   - "Configuration" link → Admin Configuration page
   - "Users" link → User Management page

2. **Grower Selector**
   - Same as Grower dashboard
   - For admins, shows all growers
   - Searchable

3. **Harvest Grid**
   - Same columns as grower view
   - Plus clickable links for Certificate and PUR
   - On click Certificate/PUR link → Open verification view

---

### Screen 2: Document Verification

**URL**: `/admin/verify-document/{documentId}`
**Navigation**: Click on Certificate or PUR link in admin dashboard

#### Layout
```
┌────────────────────────────────────────────────────────────┐
│ Back to Dashboard                                           │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  Document Verification                                     │
│  Type: Certificate                                         │
│  Block: P-5235 (PIG)                                       │
│  Grower: JJB FARMS (LTR001)                                │
│  Uploaded: June 10, 2026 by penny.cynthia@gmail.com        │
│                                                             │
├─ DOCUMENT PREVIEW ────────────────────────────────────────┤
│                                                             │
│  [PDF Viewer / Image Viewer / Document Viewer]            │
│  ┌────────────────────────────────────────────────────┐   │
│  │                                                     │   │
│  │   [Document Content Displayed Here]                │   │
│  │                                                     │   │
│  │   [Download PDF Link]                              │   │
│  │                                                     │   │
│  └────────────────────────────────────────────────────┘   │
│                                                             │
├─ VERIFICATION DECISION ───────────────────────────────────┤
│                                                             │
│  Status: Pending Verification                             │
│                                                             │
│  ◉ Verify (Accept)     ◯ Reject                           │
│                                                             │
│  Rejection Reason (if selecting Reject):                  │
│  [Multiline text area]                                     │
│                                                             │
│  [Cancel] [Submit Decision]                               │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

#### Specifications

1. **Document Information**
   - Display document type (Certificate or PUR)
   - Show associated block and grower
   - Show upload date and uploader email
   - Display current verification status

2. **Document Preview**
   - Embed PDF viewer for PDF files
   - Image viewer for JPG/PNG
   - Document viewer for DOCX/XLSX
   - Download link to original file

3. **Verification Decision**
   - Radio button: Verify (Accept) or Reject
   - If Reject selected → Show text area for rejection reason
   - Submit button
   - Cancel button returns to dashboard

4. **Confirmation Dialog**
```
┌─────────────────────────────────────────────────────┐
│ Confirm Verification Decision                       │
├─────────────────────────────────────────────────────┤
│                                                      │
│ You are verifying the following:                    │
│                                                      │
│ Type: Certificate                                   │
│ Block: P-5235 (PIG)                                 │
│ Grower: JJB FARMS (LTR001)                          │
│ Decision: APPROVED                                  │
│                                                      │
│ This will update all related harvest records.       │
│ Are you sure?                                       │
│                                                      │
│ [Cancel] [Confirm]                                  │
│                                                      │
└─────────────────────────────────────────────────────┘
```

#### Backend Actions
On confirmation:

**If Verify (Accept)**:
1. Update DocumentUpload record:
   - VerificationStatus = 'Verified'
   - VerifiedBy = @currentUserEmail
   - VerificationDate = NOW()

2. Find all HarvestMaster records with this harvest ID:
   - If DocumentType = 'Certificate':
     - Update HarvestMaster.CertificateStatus = 'Verified'
   - If DocumentType = 'PUR':
     - Update HarvestMaster.PURStatus = 'Verified'
   - Update LastUpdatedDate, LastUpdatedTime, ModifiedBy

3. Log audit records

4. Show success message

**If Reject**:
1. Update DocumentUpload record:
   - VerificationStatus = 'Rejected'
   - VerifiedBy = @currentUserEmail
   - VerificationDate = NOW()
   - RejectionReason = @adminReason

2. Find all HarvestMaster records:
   - If DocumentType = 'Certificate':
     - Update CertificateStatus = 'Pending Upload' (ready for re-upload)
   - If DocumentType = 'PUR':
     - Update PURStatus = 'Pending Upload'

3. Log audit records

4. Show success message

---

### Screen 3: User Management

**URL**: `/admin/users`
**Navigation**: From Admin Dashboard "Users" link

#### Layout
```
┌──────────────────────────────────────────────────────────────┐
│ Back to Dashboard                          [Add New User]    │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  User Management                                             │
│                                                               │
│  [Search by email] [Filter by role] [Show active only]       │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │Email│Role│Status│Last Login│Password Chg│Actions│       │
│  ├─────────────────────────────────────────────────────────┤ │
│  │penny│Grow│Activ│Jun 10...|May 28 2026│Edit|Reset│      │
│  │jacob│Grow│Inact│Never    │Jun 10..|Edit|Reset│          │
│  │sush.│Adm │Activ│Jun 13...|Jun 05 2026│Edit|Reset│       │
│  │mark.│Grow│Activ│Jun 13...|Jun 10..|Edit|Reset│          │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

#### Specifications

1. **User List Grid**
   - Data from Users table
   - Columns: Email, Role, Status, Last Login, Last Password Change, Actions
   - Searchable by email
   - Filterable by role (Admin/Grower)
   - Checkbox "Show active only"
   - Pagination: 25 users per page

2. **Add New User Button**
   - Opens create user modal
   - Fields: Email, Role, Send temp password to email

3. **Edit Action**
   - Edit user email (if not primary login)
   - Change role
   - Set active/inactive status

4. **Reset Password Action**
   - Generates temporary password
   - Shows one-time to admin
   - Admin must provide to user
   - Logged with admin email

---

### Screen 4: Configuration

**URL**: `/admin/configuration`
**Navigation**: From Admin Dashboard "Configuration" link

#### Layout
```
┌────────────────────────────────────────────────────────────┐
│ Back to Dashboard                                           │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  Configuration Settings                                    │
│                                                             │
│  Current Harvest Year for Growers *                        │
│  [2026]                                                     │
│                                                             │
│  Current Harvest Year for Admin *                          │
│  [2026]                                                     │
│                                                             │
│  Max File Upload Size (MB) *                               │
│  [50]                                                       │
│                                                             │
│  Allowed File Types *                                      │
│  [pdf, docx, xlsx, jpg, png]                               │
│                                                             │
│  Password Expiration Days                                  │
│  [90]                                                       │
│                                                             │
│  [Cancel] [Save Changes]                                   │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

#### Specifications

1. **Configuration Fields**
   - Editable text inputs
   - Validate numeric fields
   - Show description for each field
   - Required fields marked with *

2. **Save Changes**
   - Update Configuration table
   - Log audit record
   - Show success message

---

## Status Transition Workflows

### Certificate Status Flow
```
Pending Declaration
        ↓
        ├─→ Grower declares sustainable
        │         ↓
        │   Pending Upload
        │         ↓
        │   Grower uploads certificate
        │         ↓
        │   Pending Verification
        │    ↙            ↘
        │  Admin         Admin
        │  Accepts       Rejects
        │    ↓              ↓
        │  Verified    Pending Upload (re-upload)
        │
        └─→ Grower declares non sustainable
                  ↓
              Non Sustainable (final)
```

### PUR Status Flow
```
Pending Upload (default for all blocks)
        ↓
  Grower uploads PUR
        ↓
 Pending Verification
  ↙              ↘
Admin          Admin
Accepts        Rejects
  ↓              ↓
Verified   Pending Upload (re-upload)
```
