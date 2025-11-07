# Troubleshooting Guide

## Common Issues and Solutions

### 1. Connection Issues

#### Issue: ORA-12170: TNS:Connect timeout occurred

**Symptoms:**
- Azure Functions timeout when connecting to Oracle
- Connection hangs for extended period

**Root Causes:**
- Network connectivity issues
- Firewall blocking traffic
- Incorrect connection string
- Oracle listener not running

**Solutions:**

```bash
# Check network connectivity from Azure
az network vnet subnet show \
  --resource-group rg-oracle-integration \
  --vnet-name vnet-oracle \
  --name subnet-database

# Verify NSG rules
az network nsg rule list \
  --resource-group rg-oracle-integration \
  --nsg-name nsg-database-subnet \
  --output table

# Test Oracle listener
tnsping ORCLPDB
lsnrctl status
```

```sql
-- Check Oracle listener status
SELECT * FROM V$LISTENER_NETWORK;

-- Verify service registration
SELECT name, value FROM V$PARAMETER WHERE name LIKE 'service%';
```

---

#### Issue: ORA-01017: Invalid username/password

**Symptoms:**
- Authentication failures
- Intermittent login issues

**Solutions:**

```sql
-- Check user account status
SELECT username, account_status, expiry_date, lock_date
FROM dba_users
WHERE username = 'AZURE_INTEGRATION';

-- Unlock account
ALTER USER azure_integration ACCOUNT UNLOCK;

-- Reset password
ALTER USER azure_integration IDENTIFIED BY "NewPassword123!";

-- Disable password expiration
ALTER PROFILE DEFAULT LIMIT PASSWORD_LIFE_TIME UNLIMITED;
```

```bash
# Update Key Vault secret
az keyvault secret set \
  --vault-name kv-oracle-integration \
  --name oracle-password \
  --value "NewPassword123!"
```

---

### 2. Performance Issues

#### Issue: Slow Query Execution

**Solutions:**

```sql
-- Check for blocking sessions
SELECT 
  s1.username blocking_user,
  s1.sid blocking_sid,
  s2.username blocked_user,
  s2.sid blocked_sid,
  s2.wait_time,
  s2.seconds_in_wait
FROM 
  v$lock l1, v$session s1,
  v$lock l2, v$session s2
WHERE 
  s1.sid = l1.sid
  AND s2.sid = l2.sid
  AND l1.block = 1
  AND l2.request > 0;

-- Find slow queries
SELECT 
  sql_id,
  elapsed_time/1000000 elapsed_seconds,
  executions,
  buffer_gets,
  disk_reads,
  SUBSTR(sql_text, 1, 100) sql_text
FROM 
  v$sql
WHERE 
  elapsed_time > 1000000
ORDER BY 
  elapsed_time DESC
FETCH FIRST 10 ROWS ONLY;

-- Gather statistics
EXEC DBMS_STATS.GATHER_SCHEMA_STATS('AZURE_INTEGRATION');
```

---

#### Issue: Azure Function Cold Start Delays

**Solutions:**

```bash
# Enable Always On (requires Premium plan)
az functionapp config set \
  --name func-oracle-api \
  --resource-group rg-oracle-integration \
  --always-on true

# Upgrade to Premium plan
az functionapp plan create \
  --resource-group rg-oracle-integration \
  --name plan-oracle-premium \
  --location eastus \
  --sku EP1
```

```javascript
// Implement connection pooling
const oracledb = require('oracledb');

let poolInitialized = false;

async function initializePool() {
    if (!poolInitialized) {
        await oracledb.createPool({
            user: process.env["ORACLE_USER"],
            password: process.env["ORACLE_PASSWORD"],
            connectionString: process.env["ORACLE_CONNECTION_STRING"],
            poolAlias: 'default',
            poolMin: 2,
            poolMax: 10,
            poolIncrement: 1
        });
        poolInitialized = true;
    }
}

// Call during cold start
initializePool();
```

---

### 3. Data Synchronization Issues

#### Issue: Duplicate Records in Target System

**Solutions:**

```sql
-- Create sync tracking table
CREATE TABLE sync_tracking (
  record_id VARCHAR2(100) PRIMARY KEY,
  table_name VARCHAR2(128),
  last_sync_time TIMESTAMP,
  sync_hash VARCHAR2(64),
  sync_status VARCHAR2(20)
);

-- Implement idempotent sync
CREATE OR REPLACE PROCEDURE sync_record(
  p_table_name IN VARCHAR2,
  p_record_id IN VARCHAR2,
  p_data IN CLOB
) AS
  l_existing_hash VARCHAR2(64);
  l_new_hash VARCHAR2(64);
BEGIN
  -- Calculate hash of new data
  l_new_hash := DBMS_CRYPTO.HASH(p_data, DBMS_CRYPTO.HASH_SH256);
  
  -- Check if record already synced with same data
  BEGIN
    SELECT sync_hash INTO l_existing_hash
    FROM sync_tracking
    WHERE record_id = p_record_id;
    
    IF l_existing_hash = l_new_hash THEN
      RETURN; -- Already synced, skip
    END IF;
  EXCEPTION
    WHEN NO_DATA_FOUND THEN
      NULL; -- New record, continue
  END;
  
  -- Perform sync (your sync logic here)
  
  -- Update tracking
  MERGE INTO sync_tracking t
  USING (SELECT p_record_id as record_id FROM dual) s
  ON (t.record_id = s.record_id)
  WHEN MATCHED THEN
    UPDATE SET 
      last_sync_time = SYSTIMESTAMP,
      sync_hash = l_new_hash,
      sync_status = 'SUCCESS'
  WHEN NOT MATCHED THEN
    INSERT (record_id, table_name, last_sync_time, sync_hash, sync_status)
    VALUES (p_record_id, p_table_name, SYSTIMESTAMP, l_new_hash, 'SUCCESS');
    
  COMMIT;
END sync_record;
/
```

---

### 4. Azure Function Issues

#### Issue: Function Exceeding Memory Limit

**Solutions:**

```javascript
// Implement streaming for large result sets
async function processLargeQuery(connection, query) {
    const stream = connection.queryStream(query);
    const batchSize = 100;
    let batch = [];
    
    return new Promise((resolve, reject) => {
        stream.on('data', (row) => {
            batch.push(row);
            
            if (batch.length >= batchSize) {
                processBatch(batch);
                batch = [];
            }
        });
        
        stream.on('end', () => {
            if (batch.length > 0) {
                processBatch(batch);
            }
            resolve();
        });
        
        stream.on('error', reject);
    });
}

// Limit result set size
const result = await connection.execute(
    query,
    [],
    {
        maxRows: 1000,
        resultSet: true
    }
);
```

---

#### Issue: Function Timeout (230 seconds exceeded)

**Solutions:**

```json
// Increase timeout in host.json
{
  "version": "2.0",
  "functionTimeout": "00:10:00",
  "extensions": {
    "http": {
      "routePrefix": "api",
      "maxOutstandingRequests": 200,
      "maxConcurrentRequests": 100
    }
  }
}
```

```javascript
// Implement long-running operations with Durable Functions
const df = require("durable-functions");

module.exports = df.orchestrator(function* (context) {
    const outputs = [];
    
    outputs.push(yield context.df.callActivity("Step1", input));
    outputs.push(yield context.df.callActivity("Step2", input));
    outputs.push(yield context.df.callActivity("Step3", input));
    
    return outputs;
});
```

---

### 5. Copilot Integration Issues

#### Issue: Natural Language Query Generation Failures

**Solutions:**

```javascript
// Improve schema context for OpenAI
async function getEnhancedSchemaInfo() {
    const schemaInfo = await getBasicSchemaInfo();
    
    const enhancedSchema = schemaInfo.map(table => ({
        ...table,
        sampleData: getSampleData(table.name),
        relationships: getRelationships(table.name),
        description: getTableDescription(table.name)
    }));
    
    return enhancedSchema;
}

// Add validation layer
function validateGeneratedSQL(sql) {
    const validationRules = [
        { pattern: /^SELECT/i, error: "Only SELECT queries allowed" },
        { pattern: /FROM\s+[\w_]+/i, error: "Must specify valid table" },
        { pattern: /;\s*DROP/i, error: "DROP statements not allowed", invert: true }
    ];
    
    for (const rule of validationRules) {
        const matches = rule.pattern.test(sql);
        if (rule.invert ? matches : !matches) {
            throw new Error(rule.error);
        }
    }
    
    return true;
}
```

---

## Diagnostic Tools

### Health Check Script

```bash
#!/bin/bash
# comprehensive-health-check.sh

echo "Oracle-Azure Integration Health Check"
echo "======================================"

# Check Azure Function
echo "1. Checking Azure Function..."
az functionapp show \
  --name func-oracle-api \
  --resource-group rg-oracle-integration \
  --query "{Name:name, State:state, HostNames:defaultHostName}" \
  --output table

# Check Key Vault
echo "2. Checking Key Vault..."
az keyvault show \
  --name kv-oracle-integration \
  --query "{Name:name, Location:location, Status:properties.provisioningState}" \
  --output table

# Test Oracle connectivity
echo "3. Testing Oracle connectivity..."
curl -X GET https://func-oracle-api.azurewebsites.net/api/health \
  -H "Content-Type: application/json" | jq '.'

# Check Application Insights
echo "4. Checking for errors (last 1 hour)..."
az monitor app-insights query \
  --app ai-oracle-integration \
  --resource-group rg-oracle-integration \
  --analytics-query "exceptions | where timestamp > ago(1h) | summarize count() by problemId" \
  --offset 1h
```

### Performance Monitoring Query

```sql
-- Oracle Performance Diagnostics
SET PAGESIZE 1000
SET LINESIZE 200

-- Active Sessions
SELECT 
  s.sid,
  s.serial#,
  s.username,
  s.status,
  s.wait_class,
  s.seconds_in_wait,
  s.sql_id
FROM 
  v$session s
WHERE 
  s.username = 'AZURE_INTEGRATION'
  AND s.status = 'ACTIVE';

-- Top SQL by Elapsed Time
SELECT 
  sql_id,
  executions,
  ROUND(elapsed_time/1000000, 2) elapsed_sec,
  buffer_gets,
  disk_reads,
  SUBSTR(sql_text, 1, 80) sql_text
FROM 
  v$sql
WHERE 
  parsing_schema_name = 'AZURE_INTEGRATION'
ORDER BY 
  elapsed_time DESC
FETCH FIRST 10 ROWS ONLY;

-- Wait Events
SELECT 
  event,
  total_waits,
  ROUND(time_waited/100, 2) time_sec,
  ROUND(average_wait/100, 4) avg_wait_ms
FROM 
  v$system_event
WHERE 
  wait_class != 'Idle'
ORDER BY 
  time_waited DESC
FETCH FIRST 10 ROWS ONLY;
```

---

## Getting Help

### Support Channels
- Email: support@yourcompany.com
- Slack: #oracle-azure-integration
- Documentation: https://docs.yourcompany.com

### Escalation Path
1. Level 1: Team Lead
2. Level 2: Principal Engineer
3. Level 3: Architecture Team
4. Level 4: Microsoft/Oracle Support

### Creating Support Tickets
Include the following information:
- Error messages and stack traces
- Azure Function logs
- Oracle AWR/ASH reports
- Steps to reproduce
- Impact and urgency level
