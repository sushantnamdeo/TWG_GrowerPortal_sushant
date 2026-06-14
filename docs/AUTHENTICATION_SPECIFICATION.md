# Authentication & Authorization Specification

## Overview
Custom email/password-based authentication without Microsoft credentials. Role-based access control for Admins and Growers.

---

## User Authentication Flow

### Step 1: User Access
- User navigates to portal URL
- Anonymous access allowed to login page only
- All other routes require authentication

### Step 2: Login Request
```json
POST /api/auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "UserPassword123!"
}
```

### Step 3: Credentials Validation
1. Query Users table by email
   - If not found → Return 401 (Invalid email or password)
   - If found → Continue to step 4

2. Check IsActive flag
   - If false → Return 401 (Account deactivated)
   - If true → Continue to step 5

3. Verify password hash
   - Use bcrypt.compare(inputPassword, storedHash)
   - If mismatch → Return 401 (Invalid email or password)
   - If match → Continue to step 6

4. Check if password expired
   - If password older than 90 days → Return 403 (Password expired, must reset)
   - Otherwise → Continue to step 7

### Step 4: JWT Token Generation
- Generate JWT with claims:
  ```json
  {
    "sub": "user@example.com",
    "userId": 123,
    "email": "user@example.com",
    "role": "Grower",
    "iat": 1234567890,
    "exp": 1234654290,  // 24 hours
    "iss": "TWGGrowerPortal",
    "aud": "GrowerPortalAPI"
  }
  ```
- Sign with private key (stored in Azure Key Vault)

### Step 5: Response
```json
200 OK
Content-Type: application/json
Set-Cookie: authToken=<JWT>; HttpOnly; Secure; SameSite=Strict; Max-Age=86400

{
  "success": true,
  "message": "Login successful",
  "user": {
    "email": "user@example.com",
    "role": "Grower",
    "displayName": "User Name"
  },
  "expiresIn": 86400
}
```

### Step 6: Update Last Login
- Update Users table LastLoginDate to current UTC time
- Log audit record

---

## Authorization

### Role-Based Access Control (RBAC)

#### Admin Role
- **Permissions**:
  - View all grower data
  - Manage all users
  - Verify/reject documents
  - Modify configuration
  - View audit logs
  - Access to all grower codes (automatic)

- **UI Access**:
  - Admin Dashboard
  - User Management
  - Configuration Management
  - Audit Logs
  - Certificate Verification

#### Grower Role
- **Permissions**:
  - View own grower data only
  - Declare sustainability for own blocks
  - Upload certificates and PUR documents
  - Change own password
  - View own upload history

- **UI Access** (based on GrowerUserRelationship):
  - Grower Dashboard
  - Own harvest data
  - Document upload interface

### Authorization Check Flow

```
1. Request arrives with JWT token
   ↓
2. Validate JWT signature
   ↓
3. Check token expiry
   ↓
4. Extract user role from claims
   ↓
5. If Admin → Allow all operations
   ↓
6. If Grower → Check GrowerUserRelationship
   - Query: SELECT * FROM GrowerUserRelationship 
            WHERE UserEmail = @email 
            AND GrowerCode = @requestedGrowerCode
   - If found & Active → Allow
   - If not found or Inactive → Return 403
   ↓
7. Check row-level security (RLS) policies
   ↓
8. Grant or deny access
```

---

## Password Management

### Password Requirements
- **Minimum Length**: 12 characters
- **Character Types**: Must include:
  - At least 1 uppercase letter (A-Z)
  - At least 1 lowercase letter (a-z)
  - At least 1 digit (0-9)
  - At least 1 special character (!@#$%^&*)
- **History**: Cannot reuse last 5 passwords
- **Expiration**: 90 days (optional, configurable)

### Password Change Flow

#### User-Initiated Change
```json
POST /api/auth/change-password
Authorization: Bearer <JWT>
Content-Type: application/json

{
  "currentPassword": "OldPassword123!",
  "newPassword": "NewPassword456@",
  "confirmPassword": "NewPassword456@"
}
```

**Validation**:
1. Verify JWT token is valid
2. Extract user email from JWT
3. Verify current password matches
4. Validate new password meets requirements
5. Check new password not in history
6. Hash new password with bcrypt
7. Update Users table
8. Invalidate all existing tokens for this user
9. Return success message
10. Log audit record

#### Admin-Initiated Password Reset
```json
POST /api/admin/users/{userId}/password
Authorization: Bearer <JWT>
Content-Type: application/json

{
  "newPassword": "TempPassword123!"
}
```

**Validation**:
1. Verify requester is Admin
2. Verify target user exists
3. Generate strong temporary password
4. Hash password
5. Update Users table
6. Set flag requiring password change on next login
7. Return temporary password (only shown once to Admin)
8. Log audit record with Admin email

### Password Storage
- **Algorithm**: bcrypt with cost factor 12
- **Salt**: Automatically generated per password
- **Hashing Function**: bcrypt($password, $cost=12)
- **Never stored as plaintext**
- **Never transmitted in logs**

---

## Session Management

### JWT Token
- **Expiration**: 24 hours
- **Refresh Token**: 7 days (optional)
- **Storage**: Secure HTTP-only cookie + localStorage
- **Validation**: On every API request

### Logout
```json
POST /api/auth/logout
Authorization: Bearer <JWT>
```

**Actions**:
1. Clear authentication cookie
2. Invalidate JWT token (optional, add to blacklist)
3. Clear localStorage
4. Redirect to login page
5. Log audit record

### Idle Session Timeout
- **Timeout Duration**: 30 minutes of inactivity
- **Activity Tracking**: API requests reset the timer
- **Warning**: Show warning 5 minutes before timeout
- **Action**: Auto-logout when timeout reached

---

## Security Measures

### Password Storage Security
1. **Bcrypt Hashing**
   - Slow algorithm (resistant to brute force)
   - Cost factor 12 (configurable)
   - Automatic salt generation

2. **No Reversible Encryption**
   - Passwords never recoverable
   - Admin can only reset, not retrieve

3. **Audit Trail**
   - Every password change logged
   - Admin user recorded
   - Timestamp recorded
   - Not visible in logs (hashes only)

### Attack Prevention
1. **Brute Force Protection**
   - Lock account after 5 failed attempts
   - Lock duration: 15 minutes
   - Log all failed attempts

2. **Password Reset Abuse Prevention**
   - Rate limit: Maximum 3 resets per day per user
   - Admin resets logged with admin email
   - Email confirmation for user-initiated resets

3. **Token Security**
   - HTTPS only (no HTTP)
   - HTTP-only cookies (no JavaScript access)
   - SameSite=Strict
   - No sensitive data in token payload

### Audit Logging
Every authentication event logged:
- Successful login (email, timestamp, IP)
- Failed login attempts (email, timestamp, IP, reason)
- Password changes (user email, admin email if admin-initiated, timestamp)
- Session timeout (email, timestamp)
- Token refresh (email, timestamp)
- Logout (email, timestamp)

---

## Implementation Guidelines

### Backend (ASP.NET Core)

#### NuGet Packages
```
Microsoft.AspNetCore.Authentication.JwtBearer
System.IdentityModel.Tokens.Jwt
BCrypt.Net-Next
Microsoft.EntityFrameworkCore
Microsoft.EntityFrameworkCore.SqlServer
```

#### Key Classes
```csharp
public class AuthenticationService
{
    // Login logic
    public async Task<LoginResponse> LoginAsync(LoginRequest request)
    
    // Password validation
    public bool ValidatePassword(string password)
    
    // JWT generation
    public string GenerateToken(User user)
    
    // Password change
    public async Task<bool> ChangePasswordAsync(string email, string oldPassword, string newPassword)
}

public class AuthorizationService
{
    // Check user role
    public async Task<bool> IsAdminAsync(string email)
    
    // Check grower access
    public async Task<bool> CanAccessGrowerAsync(string email, string growerCode)
    
    // Get accessible growers for user
    public async Task<List<string>> GetAccessibleGrowersAsync(string email)
}
```

#### JWT Configuration
```csharp
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(key),
            ValidateIssuer = true,
            ValidIssuer = "TWGGrowerPortal",
            ValidateAudience = true,
            ValidAudience = "GrowerPortalAPI",
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero
        };
    });
```

### Frontend (React)

#### Authentication Context
```javascript
const AuthContext = createContext();

function AuthProvider({ children }) {
    const [user, setUser] = useState(null);
    const [isLoading, setIsLoading] = useState(false);
    
    const login = async (email, password) => {
        // Call /api/auth/login
        // Store JWT in cookie
        // Set user context
    };
    
    const logout = () => {
        // Call /api/auth/logout
        // Clear cookie
        // Clear context
    };
    
    const changePassword = async (currentPassword, newPassword) => {
        // Call /api/auth/change-password
    };
    
    return (
        <AuthContext.Provider value={{ user, login, logout, changePassword }}>
            {children}
        </AuthContext.Provider>
    );
}
```

#### Protected Routes
```javascript
function ProtectedRoute({ children, requiredRole }) {
    const { user } = useContext(AuthContext);
    
    if (!user) {
        return <Navigate to="/login" />;
    }
    
    if (requiredRole && user.role !== requiredRole) {
        return <Navigate to="/unauthorized" />;
    }
    
    return children;
}
```

---

## Testing Scenarios

### Valid Login
```
Email: valid@example.com
Password: ValidPassword123!
Expected: JWT token returned, redirected to dashboard
```

### Invalid Email
```
Email: nonexistent@example.com
Password: AnyPassword123!
Expected: 401 with message "Invalid email or password"
```

### Invalid Password
```
Email: valid@example.com
Password: WrongPassword123!
Expected: 401 with message "Invalid email or password"
```

### Inactive Account
```
Email: inactive@example.com
Password: CorrectPassword123!
Expected: 401 with message "Account is deactivated"
```

### Password Change - Success
```
Old: OldPassword123!
New: NewPassword456@
Expected: Success message, tokens invalidated
```

### Password Change - Weak Password
```
New: weak
Expected: 400 with validation error
```

### Grower Access - Own Data
```
User: grower@example.com (role: Grower)
Access: /api/growers/LDC001
Expected: 200 OK (has relationship)
```

### Grower Access - Other Grower Data
```
User: grower@example.com (role: Grower)
Access: /api/growers/UHF009 (no relationship)
Expected: 403 Forbidden
```
