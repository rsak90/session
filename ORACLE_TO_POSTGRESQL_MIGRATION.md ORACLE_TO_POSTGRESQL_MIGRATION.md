
# Oracle to PostgreSQL Migration Guide

This guide provides step-by-step instructions to convert the Oracle Session Migration POC to use PostgreSQL instead of Oracle Database.

## Overview

The migration involves updating NuGet packages, database schema, connection strings, and database-specific code. The core session management logic and MVC/API architecture remain unchanged.

## Prerequisites

- PostgreSQL 12+ installed and running
- .NET 9 SDK
- Database administration tool (pgAdmin, DBeaver, or psql)

---

## Step 1: Update NuGet Packages

### 1.1 Update `SessionOracleMigration.csproj`

**Replace:**
```xml
<PackageReference Include="Oracle.ManagedDataAccess.Core" Version="23.5.1" />
```

**With:**
```xml
<PackageReference Include="Npgsql" Version="8.0.1" />
```

**Final file should look like:**
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Npgsql" Version="8.0.1" />
    <PackageReference Include="Swashbuckle.AspNetCore" Version="6.8.1" />
  </ItemGroup>

</Project>
```

---

## Step 2: Create PostgreSQL Database Schema

### 2.1 Create `Database/CreateSessionsTable_PostgreSQL.sql`

**Create new file with PostgreSQL-specific SQL:**
```sql
-- PostgreSQL Sessions Table Creation Script
-- This table stores session data for the ASP.NET MVC application

CREATE TABLE Sessions (
    SessionId VARCHAR(449) PRIMARY KEY,
    SessionKey VARCHAR(200) NOT NULL,
    SessionValue TEXT,
    ExpiresAtTime TIMESTAMP NOT NULL,
    SlidingExpirationInSeconds INTEGER,
    AbsoluteExpiration TIMESTAMP
);

-- Create index for efficient cleanup of expired sessions
CREATE INDEX IX_Sessions_ExpiresAtTime ON Sessions (ExpiresAtTime);

-- Create composite index for session retrieval
CREATE INDEX IX_Sessions_SessionId_Key ON Sessions (SessionId, SessionKey);

-- Grant necessary permissions (adjust user as needed)
-- GRANT SELECT, INSERT, UPDATE, DELETE ON Sessions TO your_app_user;
```

### 2.2 Execute the SQL Script

Connect to your PostgreSQL database and execute the script:
```bash
psql -U postgres -d your_database_name -f Database/CreateSessionsTable_PostgreSQL.sql
```

---

## Step 3: Update Configuration Files

### 3.1 Update `appsettings.json`

**Replace:**
```json
"ConnectionStrings": {
  "OracleConnection": "Data Source=localhost:1521/XE;User Id=your_username;Password=your_password;Connection Timeout=30;"
}
```

**With:**
```json
"ConnectionStrings": {
  "PostgreSqlConnection": "Host=localhost;Database=sessiondb;Username=postgres;Password=your_password;Port=5432;Timeout=30;"
}
```

### 3.2 Update `appsettings.Development.json`

**Replace:**
```json
"ConnectionStrings": {
  "OracleConnection": "Data Source=localhost:1521/XE;User Id=dev_user;Password=dev_password;Connection Timeout=30;"
}
```

**With:**
```json
"ConnectionStrings": {
  "PostgreSqlConnection": "Host=localhost;Database=sessiondb_dev;Username=postgres;Password=dev_password;Port=5432;Timeout=30;"
}
```

---

## Step 4: Update Service Files

### 4.1 Rename and Update `Services/OracleSessionStore.cs` → `Services/PostgreSqlSessionStore.cs`

**Complete file replacement:**
```csharp
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Npgsql;
using SessionOracleMigration.Models;
using System.Data;

namespace SessionOracleMigration.Services
{
    /// <summary>
    /// PostgreSQL-based implementation of ISessionStore for ASP.NET Core session management
    /// </summary>
    public class PostgreSqlSessionStore : ISessionStore
    {
        private readonly string _connectionString;
        private readonly ILogger<PostgreSqlSessionStore> _logger;
        private readonly TimeSpan _defaultTimeout;

        public PostgreSqlSessionStore(string connectionString, ILogger<PostgreSqlSessionStore> logger, TimeSpan defaultTimeout)
        {
            _connectionString = connectionString ?? throw new ArgumentNullException(nameof(connectionString));
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));
            _defaultTimeout = defaultTimeout;
        }

        /// <summary>
        /// Creates a new session with the specified session ID
        /// </summary>
        public ISession Create(string sessionKey, TimeSpan idleTimeout, TimeSpan ioTimeout, Func<bool> tryEstablishSession, bool isNewSessionKey)
        {
            if (string.IsNullOrEmpty(sessionKey))
                throw new ArgumentException("Session key cannot be null or empty", nameof(sessionKey));

            var session = new PostgreSqlSession(sessionKey, this, _logger, idleTimeout);
            
            if (isNewSessionKey)
            {
                _logger.LogDebug("Creating new session with ID: {SessionId}", sessionKey);
            }

            return session;
        }

        /// <summary>
        /// Retrieves session data from PostgreSQL database
        /// </summary>
        public async Task<Dictionary<string, string>> RetrieveAsync(string sessionId)
        {
            var sessionData = new Dictionary<string, string>();

            try
            {
                using var connection = new NpgsqlConnection(_connectionString);
                await connection.OpenAsync();

                const string query = @"
                    SELECT SessionKey, SessionValue 
                    FROM Sessions 
                    WHERE SessionId = @sessionId AND ExpiresAtTime > @currentTime";

                using var command = new NpgsqlCommand(query, connection);
                command.Parameters.AddWithValue("@sessionId", sessionId);
                command.Parameters.AddWithValue("@currentTime", DateTime.UtcNow);

                using var reader = await command.ExecuteReaderAsync();
                while (await reader.ReadAsync())
                {
                    var key = reader.GetString("SessionKey");
                    var value = reader.IsDBNull("SessionValue") ? string.Empty : reader.GetString("SessionValue");
                    sessionData[key] = value;
                }

                _logger.LogDebug("Retrieved {Count} session items for session {SessionId}", sessionData.Count, sessionId);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error retrieving session data for session {SessionId}", sessionId);
                throw;
            }

            return sessionData;
        }

        /// <summary>
        /// Commits session changes to PostgreSQL database
        /// </summary>
        public async Task CommitAsync(string sessionId, Dictionary<string, string> sessionData, TimeSpan timeout)
        {
            try
            {
                using var connection = new NpgsqlConnection(_connectionString);
                await connection.OpenAsync();

                using var transaction = connection.BeginTransaction();

                try
                {
                    // Delete existing session data
                    const string deleteQuery = "DELETE FROM Sessions WHERE SessionId = @sessionId";
                    using var deleteCommand = new NpgsqlCommand(deleteQuery, connection, transaction);
                    deleteCommand.Parameters.AddWithValue("@sessionId", sessionId);
                    await deleteCommand.ExecuteNonQueryAsync();

                    // Insert updated session data
                    const string insertQuery = @"
                        INSERT INTO Sessions (SessionId, SessionKey, SessionValue, ExpiresAtTime, SlidingExpirationInSeconds)
                        VALUES (@sessionId, @sessionKey, @sessionValue, @expiresAt, @slidingExpiration)";

                    var expiresAt = DateTime.UtcNow.Add(timeout);
                    var slidingExpirationSeconds = (int)timeout.TotalSeconds;

                    foreach (var kvp in sessionData)
                    {
                        using var insertCommand = new NpgsqlCommand(insertQuery, connection, transaction);
                        insertCommand.Parameters.AddWithValue("@sessionId", sessionId);
                        insertCommand.Parameters.AddWithValue("@sessionKey", kvp.Key);
                        insertCommand.Parameters.AddWithValue("@sessionValue", kvp.Value ?? (object)DBNull.Value);
                        insertCommand.Parameters.AddWithValue("@expiresAt", expiresAt);
                        insertCommand.Parameters.AddWithValue("@slidingExpiration", slidingExpirationSeconds);
                        
                        await insertCommand.ExecuteNonQueryAsync();
                    }

                    await transaction.CommitAsync();
                    _logger.LogDebug("Committed {Count} session items for session {SessionId}", sessionData.Count, sessionId);
                }
                catch
                {
                    await transaction.RollbackAsync();
                    throw;
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error committing session data for session {SessionId}", sessionId);
                throw;
            }
        }

        /// <summary>
        /// Removes expired sessions from PostgreSQL database
        /// </summary>
        public async Task RemoveAsync(string sessionId)
        {
            try
            {
                using var connection = new NpgsqlConnection(_connectionString);
                await connection.OpenAsync();

                const string deleteQuery = "DELETE FROM Sessions WHERE SessionId = @sessionId";
                using var command = new NpgsqlCommand(deleteQuery, connection);
                command.Parameters.AddWithValue("@sessionId", sessionId);
                
                var rowsAffected = await command.ExecuteNonQueryAsync();
                _logger.LogDebug("Removed session {SessionId}, rows affected: {RowsAffected}", sessionId, rowsAffected);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error removing session {SessionId}", sessionId);
                throw;
            }
        }

        /// <summary>
        /// Cleans up expired sessions from the database
        /// </summary>
        public async Task CleanupExpiredSessionsAsync()
        {
            try
            {
                using var connection = new NpgsqlConnection(_connectionString);
                await connection.OpenAsync();

                const string deleteQuery = "DELETE FROM Sessions WHERE ExpiresAtTime <= @currentTime";
                using var command = new NpgsqlCommand(deleteQuery, connection);
                command.Parameters.AddWithValue("@currentTime", DateTime.UtcNow);
                
                var rowsAffected = await command.ExecuteNonQueryAsync();
                _logger.LogInformation("Cleaned up {RowsAffected} expired sessions", rowsAffected);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error cleaning up expired sessions");
                throw;
            }
        }
    }
}
```

### 4.2 Rename and Update `Services/OracleSession.cs` → `Services/PostgreSqlSession.cs`

**Key changes:**
- Rename class from `OracleSession` to `PostgreSqlSession`
- Update constructor parameter from `OracleSessionStore` to `PostgreSqlSessionStore`
- Update field type from `OracleSessionStore` to `PostgreSqlSessionStore`

**Replace class declaration:**
```csharp
public class PostgreSqlSession : ISession
{
    private readonly string _sessionId;
    private readonly PostgreSqlSessionStore _store;
    // ... rest remains the same
    
    public PostgreSqlSession(string sessionId, PostgreSqlSessionStore store, ILogger logger, TimeSpan idleTimeout)
    {
        // ... constructor implementation remains the same
    }
    
    // ... all other methods remain exactly the same
}
```

### 4.3 Update `Services/SessionExpirationManager.cs`

**Replace Oracle-specific code:**

**Find and replace:**
```csharp
// Replace imports
using Oracle.ManagedDataAccess.Client;
// With:
using Npgsql;

// Replace connection creation
using var connection = new OracleConnection(_connectionString);
// With:
using var connection = new NpgsqlConnection(_connectionString);

// Replace command creation
using var command = new OracleCommand(updateQuery, connection);
// With:
using var command = new NpgsqlCommand(updateQuery, connection);

// Replace parameter syntax (throughout the file)
command.Parameters.Add(":newExpiration", OracleDbType.TimeStamp).Value = newExpirationTime;
// With:
command.Parameters.AddWithValue("@newExpiration", newExpirationTime);

// Replace all parameter names from :paramName to @paramName
:newExpiration → @newExpiration
:slidingExpiration → @slidingExpiration  
:sessionId → @sessionId
:currentTime → @currentTime
:absoluteExpiration → @absoluteExpiration
```

### 4.4 Update `Extensions/ServiceCollectionExtensions.cs`

**Replace method names and types:**
```csharp
// Replace method name
public static IServiceCollection AddOracleSession(
// With:
public static IServiceCollection AddPostgreSqlSession(

// Replace service registration
services.AddSingleton<ISessionStore>(serviceProvider =>
{
    var logger = serviceProvider.GetRequiredService<ILogger<PostgreSqlSessionStore>>();
    return new PostgreSqlSessionStore(connectionString, logger, timeout);
});
```

---

## Step 5: Update Program.cs

### 5.1 Update `Program.cs`

**Replace:**
```csharp
// Configure Oracle session management
var connectionString = builder.Configuration.GetConnectionString("OracleConnection")
    ?? throw new InvalidOperationException("Oracle connection string not found");

// Add Oracle session store
builder.Services.AddOracleSession(connectionString, sessionTimeout);
```

**With:**
```csharp
// Configure PostgreSQL session management
var connectionString = builder.Configuration.GetConnectionString("PostgreSqlConnection")
    ?? throw new InvalidOperationException("PostgreSQL connection string not found");

// Add PostgreSQL session store
builder.Services.AddPostgreSqlSession(connectionString, sessionTimeout);
```

**Also update Swagger title:**
```csharp
c.SwaggerDoc("v1", new() { Title = "PostgreSQL Session Migration API", Version = "v1" });
c.SwaggerEndpoint("/swagger/v1/swagger.json", "PostgreSQL Session Migration API v1");
```

---

## Step 6: Update Controllers

### 6.1 Update `Controllers/SessionTestController.cs`

**Replace type checks:**
```csharp
// Replace:
if (_sessionStore is OracleSessionStore oracleStore)
{
    await oracleStore.CleanupExpiredSessionsAsync();
// With:
if (_sessionStore is PostgreSqlSessionStore postgreSqlStore)
{
    await postgreSqlStore.CleanupExpiredSessionsAsync();
```

**Update response messages:**
```csharp
// Replace:
SessionStoreType = "Oracle",
// With:
SessionStoreType = "PostgreSQL",

// Replace:
Error = "Session store is not Oracle-based"
// With:
Error = "Session store is not PostgreSQL-based"
```

---

## Step 7: Update Views and Documentation

### 7.1 Update View Titles and Labels

**In all view files, replace:**
- "Oracle Session" → "PostgreSQL Session"
- "Oracle Database" → "PostgreSQL Database"
- "Oracle session storage" → "PostgreSQL session storage"

**Files to update:**
- `Views/Shared/_Layout.cshtml`
- `Views/Home/Index.cshtml`
- `Views/Home/Login.cshtml`
- `Views/Home/Dashboard.cshtml`
- `Views/Home/SessionTest.cshtml`
- `Views/SessionTest/TimeoutTest.cshtml`

### 7.2 Update `README.md`

**Replace throughout the file:**
- "Oracle Session Migration POC" → "PostgreSQL Session Migration POC"
- "Oracle database" → "PostgreSQL database"
- "Oracle-based session storage" → "PostgreSQL-based session storage"
- Connection string examples
- Prerequisites section

---

## Step 8: Clean Up Old Files

### 8.1 Delete Oracle-specific files:
- `Services/OracleSessionStore.cs`
- `Services/OracleSession.cs`
- `Database/CreateSessionsTable.sql` (keep as reference or rename)

### 8.2 Optional: Rename project
If desired, rename the project from `SessionOracleMigration` to `SessionPostgreSqlMigration`:
- Rename the `.csproj` file
- Update namespace declarations
- Update folder names

---

## Step 9: Testing

### 9.1 Build and Run
```bash
dotnet build
dotnet run
```

### 9.2 Test Functionality
1. Navigate to https://localhost:5001
2. Test login functionality
3. Test session persistence
4. Test API endpoints via Swagger
5. Test session timeout and cleanup

### 9.3 Verify Database
Check PostgreSQL database to ensure:
- Sessions table is created
- Session data is being stored
- Expired sessions are cleaned up

---

## PostgreSQL Connection String Examples

```json
{
  "ConnectionStrings": {
    "PostgreSqlConnection": "Host=localhost;Database=sessiondb;Username=postgres;Password=password;Port=5432;Timeout=30;",
    
    // With SSL
    "PostgreSqlConnectionSSL": "Host=localhost;Database=sessiondb;Username=postgres;Password=password;Port=5432;SSL Mode=Require;",
    
    // Remote server
    "PostgreSqlConnectionRemote": "Host=192.168.1.100;Database=sessiondb;Username=app_user;Password=secure_password;Port=5432;",
    
    // Cloud (example for AWS RDS)
    "PostgreSqlConnectionCloud": "Host=mydb.cluster-xyz.us-east-1.rds.amazonaws.com;Database=sessiondb;Username=postgres;Password=password;Port=5432;SSL Mode=Require;"
  }
}
```

---

## Troubleshooting

### Common Issues:

1. **Connection String Format**: Ensure PostgreSQL connection string format is correct
2. **Parameter Syntax**: Make sure all `:param` are changed to `@param`
3. **Data Types**: Verify PostgreSQL data types match the application expectations
4. **Permissions**: Ensure database user has proper permissions
5. **Port**: Default PostgreSQL port is 5432 (not 1521 like Oracle)

### Verification Steps:

1. Check application logs for database connection errors
2. Verify PostgreSQL service is running
3. Test database connection with psql or pgAdmin
4. Ensure Sessions table exists and has correct structure
5. Check that session data is being inserted and retrieved correctly

---

## Benefits of PostgreSQL Migration

1. **Cost**: Free and open-source (no licensing fees)
2. **Performance**: Often better performance for session storage scenarios
3. **Ease of Setup**: Simpler installation and configuration
4. **Cross-Platform**: Runs on Windows, Linux, macOS
5. **JSON Support**: Native JSON data types for future enhancements
6. **Community**: Large, active community and excellent documentation
7. **Tooling**: Great .NET support with Npgsql driver

---

This migration maintains all existing functionality while switching to PostgreSQL as the backend database. The session management logic, MVC/API hybrid architecture, and all testing capabilities remain unchanged.
