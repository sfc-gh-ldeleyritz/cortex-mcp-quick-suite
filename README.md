# Connect Snowflake Cortex Agent to Amazon QuickSight via MCP

This guide walks you through exposing a Snowflake Cortex Agent to Amazon QuickSight using Snowflake's Managed MCP (Model Context Protocol) server.

## Overview

Amazon QuickSight supports MCP integration for both action execution and data access. By creating an MCP server in Snowflake that wraps your Cortex Agent, QuickSight users can interact with your agent directly from the QuickSight interface.

## Prerequisites

Before you begin, ensure you have:

- **Snowflake Requirements**
  - An existing Cortex Agent in Snowflake
  - ACCOUNTADMIN role access (or equivalent privileges to create MCP servers and security integrations)
  - A warehouse for query execution

- **Amazon QuickSight Requirements**
  - Amazon QuickSight Author subscription or higher
  - Access to the QuickSight console

## Step-by-Step Guide

### Step 1: Identify Your Agent

First, locate your Cortex Agent. If you're unsure where it is, run:

```sql
SHOW AGENTS IN ACCOUNT;
```

Note the following values:
- **Database**: The database containing your agent
- **Schema**: The schema containing your agent
- **Agent Name**: Your Cortex Agent name
- **Account**: Your Snowflake account identifier (e.g., `abc12345`)
- **Warehouse**: The warehouse to use for query execution

### Step 2: Create the MCP Server

Create an MCP server that exposes your Cortex Agent:

```sql
USE ROLE ACCOUNTADMIN;

CREATE OR REPLACE MCP SERVER <DATABASE>.<SCHEMA>.<AGENT>_MCP_SERVER
  FROM SPECIFICATION $$
    tools:
      - title: "<AGENT>"
        name: "<AGENT>"
        type: "CORTEX_AGENT_RUN"
        identifier: "<DATABASE>.<SCHEMA>.<AGENT>"
        description: "Cortex Agent exposed via MCP"
  $$;
```

Replace `<DATABASE>`, `<SCHEMA>`, and `<AGENT>` with your actual values.

Verify the server was created:

```sql
SHOW MCP SERVERS IN SCHEMA <DATABASE>.<SCHEMA>;
```

### Step 3: Grant Required Permissions

Grant access to the MCP server and related objects. Choose either PUBLIC role (simpler) or a custom role (more secure):

**Option A: Grant to PUBLIC role**

```sql
GRANT USAGE ON DATABASE <DATABASE> TO ROLE PUBLIC;
GRANT USAGE ON SCHEMA <DATABASE>.<SCHEMA> TO ROLE PUBLIC;
GRANT USAGE ON MCP SERVER <DATABASE>.<SCHEMA>.<AGENT>_MCP_SERVER TO ROLE PUBLIC;
GRANT USAGE ON AGENT <DATABASE>.<SCHEMA>.<AGENT> TO ROLE PUBLIC;
GRANT USAGE ON WAREHOUSE <WAREHOUSE> TO ROLE PUBLIC;
GRANT SELECT ON ALL TABLES IN SCHEMA <DATABASE>.<SCHEMA> TO ROLE PUBLIC;
```

**Option B: Grant to a custom role**

```sql
CREATE ROLE IF NOT EXISTS QUICKSIGHT_MCP_ROLE;

GRANT USAGE ON DATABASE <DATABASE> TO ROLE QUICKSIGHT_MCP_ROLE;
GRANT USAGE ON SCHEMA <DATABASE>.<SCHEMA> TO ROLE QUICKSIGHT_MCP_ROLE;
GRANT USAGE ON MCP SERVER <DATABASE>.<SCHEMA>.<AGENT>_MCP_SERVER TO ROLE QUICKSIGHT_MCP_ROLE;
GRANT USAGE ON AGENT <DATABASE>.<SCHEMA>.<AGENT> TO ROLE QUICKSIGHT_MCP_ROLE;
GRANT USAGE ON WAREHOUSE <WAREHOUSE> TO ROLE QUICKSIGHT_MCP_ROLE;
GRANT SELECT ON ALL TABLES IN SCHEMA <DATABASE>.<SCHEMA> TO ROLE QUICKSIGHT_MCP_ROLE;

-- Grant the role to users who need access
GRANT ROLE QUICKSIGHT_MCP_ROLE TO USER <username>;
```

**Additional grants (if applicable)**

```sql
-- If your agent uses a semantic model stored on a stage:
GRANT READ ON STAGE <DATABASE>.<SCHEMA>.<STAGE_NAME> TO ROLE PUBLIC;

-- If your agent uses Cortex Search:
GRANT USAGE ON CORTEX SEARCH SERVICE <DATABASE>.<SCHEMA>.<SERVICE_NAME> TO ROLE PUBLIC;
```

### Step 4: Create OAuth Security Integration

Create an OAuth integration for QuickSight authentication:

```sql
CREATE OR REPLACE SECURITY INTEGRATION <AGENT>_QUICKSIGHT_OAUTH
    TYPE = OAUTH
    ENABLED = TRUE
    OAUTH_CLIENT = CUSTOM
    OAUTH_CLIENT_TYPE = 'CONFIDENTIAL'
    OAUTH_REDIRECT_URI = 'https://placeholder.example.com/callback'
    OAUTH_ISSUE_REFRESH_TOKENS = TRUE
    OAUTH_REFRESH_TOKEN_VALIDITY = 86400;
```

> **Note**: The redirect URI is a placeholder. You'll update it after QuickSight generates the real URL.

### Step 5: Retrieve OAuth Credentials

Get the client ID and secret:

```sql
SELECT SYSTEM$SHOW_OAUTH_CLIENT_SECRETS('<AGENT>_QUICKSIGHT_OAUTH');
```

**Save these values securely** - you'll need them for QuickSight configuration:
- `OAUTH_CLIENT_ID`
- `OAUTH_CLIENT_SECRET`

### Step 6: Configure QuickSight MCP Integration

1. Open the **Amazon QuickSight console**

2. Navigate to **Integrations** and click **Add** (+)

3. On the **Create Integration** page, enter:
   - **Name**: A descriptive name (e.g., "Snowflake Cortex Agent")
   - **Description**: Optional description
   - **MCP server endpoint**: 
     ```
     https://<ACCOUNT>.snowflakecomputing.com/api/v2/databases/<DATABASE>/schemas/<SCHEMA>/mcp-servers/<AGENT>_MCP_SERVER
     ```

4. Click **Next**

5. Select **User authentication (OAuth)**

6. Select **Manual configuration**

7. Enter OAuth details:
   | Field | Value |
   |-------|-------|
   | Client ID | `<OAUTH_CLIENT_ID from Step 5>` |
   | Client Secret | `<OAUTH_CLIENT_SECRET from Step 5>` |
   | Token URL | `https://<ACCOUNT>.snowflakecomputing.com/oauth/token-request` |
   | Auth URL | `https://<ACCOUNT>.snowflakecomputing.com/oauth/authorize` |
   | Redirect URL | Leave blank (auto-generated) |

8. Click **Create and continue**

9. **Copy the generated Redirect URL** from the success screen

10. Continue through the remaining screens (capability review, sharing) as needed

### Step 7: Update Snowflake with the Redirect URL

Back in Snowflake, update the OAuth integration with the redirect URL from QuickSight:

```sql
ALTER SECURITY INTEGRATION <AGENT>_QUICKSIGHT_OAUTH 
    SET OAUTH_REDIRECT_URI = '<REDIRECT_URL_FROM_QUICKSIGHT>';
```

Verify the update:

```sql
DESCRIBE SECURITY INTEGRATION <AGENT>_QUICKSIGHT_OAUTH;
```

### Step 8: Complete the Connection

1. Return to **Amazon QuickSight**
2. Navigate to **Integrations**
3. Find your MCP integration
4. Click to authenticate
5. Sign in with your Snowflake credentials
6. Your agent is now available as an action in QuickSight

## Testing Your Integration

After setup is complete, test the integration by using your agent through QuickSight's action connector. The agent should respond using your Snowflake Cortex Agent's capabilities.

## Quick Reference

### Snowflake Objects Created

| Object Type | Naming Convention |
|-------------|-------------------|
| MCP Server | `<DATABASE>.<SCHEMA>.<AGENT>_MCP_SERVER` |
| OAuth Integration | `<AGENT>_QUICKSIGHT_OAUTH` |

### URLs Reference

| URL Type | Template |
|----------|----------|
| MCP Server Endpoint | `https://<ACCOUNT>.snowflakecomputing.com/api/v2/databases/<DATABASE>/schemas/<SCHEMA>/mcp-servers/<AGENT>_MCP_SERVER` |
| Authorization URL | `https://<ACCOUNT>.snowflakecomputing.com/oauth/authorize` |
| Token URL | `https://<ACCOUNT>.snowflakecomputing.com/oauth/token-request` |

### OAuth Scopes

| Role Type | Scope Value |
|-----------|-------------|
| PUBLIC role | `session:role:PUBLIC` |
| Custom role | `session:role:<CUSTOM_ROLE>` |

## Known Limitations

- **60-second timeout**: MCP operations in QuickSight have a fixed 60-second timeout. Operations exceeding this limit fail with HTTP 424.
- **No custom headers**: Custom HTTP headers are not supported in MCP operations.
- **Static tool lists**: Tool lists remain static after initial registration. Refresh manually to detect server-side changes.
- **No VPC connectivity**: VPC connectivity is not supported for MCP integrations.

## Troubleshooting

### "Object does not exist" Error

```sql
-- Verify MCP server exists
SHOW MCP SERVERS IN SCHEMA <DATABASE>.<SCHEMA>;

-- Check grants
SHOW GRANTS ON MCP SERVER <DATABASE>.<SCHEMA>.<AGENT>_MCP_SERVER;
```

### OAuth Authentication Failures

```sql
-- Verify redirect URL matches exactly
DESCRIBE SECURITY INTEGRATION <AGENT>_QUICKSIGHT_OAUTH;

-- Ensure integration is enabled
-- Look for ENABLED = TRUE in the output
```

### Permission Denied

```sql
-- Check role grants
SHOW GRANTS TO ROLE PUBLIC;

-- Verify warehouse access
SHOW GRANTS ON WAREHOUSE <WAREHOUSE>;
```

### Timeout Errors

If you're hitting the 60-second timeout:
- Optimize your agent's queries
- Use simpler prompts
- Break complex operations into smaller steps

### Agent Not Responding

1. Test the agent directly in Snowflake first
2. Verify the agent has proper tool access
3. Check semantic model permissions

## Cleanup

To remove the integration:

```sql
-- Drop the MCP server
DROP MCP SERVER IF EXISTS <DATABASE>.<SCHEMA>.<AGENT>_MCP_SERVER;

-- Drop the OAuth integration
DROP SECURITY INTEGRATION IF EXISTS <AGENT>_QUICKSIGHT_OAUTH;

-- If using a custom role, drop it (optional)
DROP ROLE IF EXISTS QUICKSIGHT_MCP_ROLE;
```

Also remove the integration from the QuickSight console under **Integrations**.

## Additional Resources

- [Amazon QuickSight MCP Integration Documentation](https://docs.aws.amazon.com/quick/latest/userguide/mcp-integration.html)
- [Snowflake Cortex Agents Documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents)
