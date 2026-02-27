# Dremio Plugins for Claude Code

Official [Claude Code](https://claude.ai/claude-code) plugins from [Dremio](https://dremio.com).

## Installation

```
/plugin marketplace add dremio/claude-plugins
/plugin install dremio@dremio-plugins
```

## Setup

1. Create a `.env` file in your project directory with your Dremio credentials:

    ```
    DREMIO_PAT=<your_personal_access_token>
    DREMIO_PROJECT_ID=<your_project_id>
    ```

2. Add the Dremio MCP server through the [Claude web interface](https://claude.ai) under **Customize > Connectors > Add custom connector**. Claude Code automatically inherits the connection.

3. Run `/dremio-setup` in Claude Code for step-by-step guidance.

For full MCP server setup instructions, see the [Dremio documentation](https://docs.dremio.com/dremio-cloud/developer/mcp-server/).

## Plugins

| Plugin | Description |
|:-------|:------------|
| **dremio** | Working with Dremio Cloud, including ingesting, transforming, and querying data |
