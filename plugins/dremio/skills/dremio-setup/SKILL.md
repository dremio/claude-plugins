---
name: dremio-setup
description: Set up the Dremio MCP server for Claude Code
user-invocable: true
---

# Dremio MCP Server Setup

Help the user set up the Dremio hosted MCP server for Claude Code. Walk them through the steps below.

## Prerequisites

- A Dremio Cloud account
- A Personal Access Token (PAT) — generated from Dremio Cloud under Account Settings > Personal Access Tokens
- Your Project ID — found in Admin > Project > Project Overview
- An OAuth application — created in Admin > Organization > OAuth Applications (select "Native Application" type, add redirect URLs: `https://claude.ai/api/mcp/auth_callback,https://claude.com/api/mcp/auth_callback,http://localhost/callback,http://localhost`)

## Step 1: Create a .env file

Ask the user for their PAT and Project ID, then create a `.env` file in their project directory:

```
DREMIO_PAT=<their_pat>
DREMIO_PROJECT_ID=<their_project_id>
```

## Step 2: Add the MCP server via Claude's web interface

The Dremio hosted MCP server must be added through the Claude web interface. Claude Code automatically inherits the connection.

1. Go to [claude.ai](https://claude.ai)
2. Navigate to **Customize > Connectors > Add custom connector**
3. Enter:
   - **Server name**: `Dremio`
   - **Remote MCP server URL**: `https://mcp.dremio.cloud/mcp/<project_id>` (US) or `https://mcp.eu.dremio.cloud/mcp/<project_id>` (EU)
   - **OAuth client ID**: the Client ID from the OAuth application
   - **OAuth client secret**: leave blank
4. Complete the OAuth authentication when prompted
5. Enable the connector in the Tools menu

## Step 3: Verify

Tell the user to restart Claude Code and run `/mcp` to confirm the Dremio server is connected. Claude Code inherits MCP connections configured in the web interface.

## Notes

- Full documentation: https://docs.dremio.com/dremio-cloud/developer/mcp-server/
- If the user already has the MCP server configured, skip to verifying the connection.
