# n8n MCP Server and Skills Setup

This document describes the n8n tooling setup for Claude Code in this project.

## What Was Installed

### 1. n8n-mcp MCP Server
- **Source**: https://github.com/czlonkowski/n8n-mcp
- **Description**: Model Context Protocol server providing access to n8n's 1,084+ workflow automation nodes
- **Configuration**: `.mcp.json` in project root
- **Access Method**: Via npx (no local installation required)

### 2. n8n-skills Plugin
- **Source**: https://github.com/czlonkowski/n8n-skills
- **Description**: Seven specialized skills for building n8n workflows
- **Installation Location**: `~/.claude/skills/`
- **Skills Installed**:
  1. **n8n-code-javascript** - Write JavaScript code in n8n Code nodes
  2. **n8n-code-python** - Write Python code in n8n Code nodes
  3. **n8n-expression-syntax** - Validate n8n expression syntax
  4. **n8n-mcp-tools-expert** - Expert guide for using n8n-mcp tools
  5. **n8n-node-configuration** - Operation-aware node configuration guidance
  6. **n8n-validation-expert** - Interpret and fix validation errors
  7. **n8n-workflow-patterns** - Proven workflow architectural patterns

## Configuration Files

### .mcp.json
```json
{
  "mcpServers": {
    "n8n-mcp": {
      "command": "npx",
      "args": ["-y", "n8n-mcp"],
      "env": {
        "MCP_MODE": "stdio",
        "LOG_LEVEL": "error",
        "DISABLE_CONSOLE_OUTPUT": "true"
      }
    }
  }
}
```

## How to Use

### Accessing n8n MCP Tools
The n8n-mcp server provides tools that Claude Code can automatically use when working with n8n workflows. These tools include:
- Searching for nodes and templates
- Getting node documentation
- Validating workflow configurations
- Managing workflow operations

### Using n8n Skills
Skills are automatically activated based on context. When working with n8n workflows, simply:
- Ask questions about n8n nodes and configurations
- Request help writing Code node scripts
- Ask for workflow pattern recommendations
- Request validation of workflow configurations

The appropriate skill will be automatically invoked based on your request.

## Capabilities

With this setup, Claude Code can now:
- ✅ Search through 525+ n8n nodes
- ✅ Access 2,653+ workflow templates
- ✅ Generate JavaScript/Python code for Code nodes
- ✅ Validate n8n expressions and syntax
- ✅ Configure nodes with operation-aware guidance
- ✅ Apply production-tested workflow patterns
- ✅ Debug and fix validation errors

## Requirements

- Node.js (for npx to work)
- Internet connection (to download n8n-mcp via npx)
- Claude Code CLI

## n8n Instance Configuration

The `.mcp.json` file is configured to connect to your n8n instance at:
- **URL**: `https://psychogenic-strengtheningly-stewart.ngrok-free.dev`
- **Authentication**: Via API key (configured in environment variables)

This allows the n8n-mcp server to:
- List and manage workflows in your instance
- Create and update workflows directly
- Deploy templates to your instance
- Test and validate workflows
- Access workflow execution history

## Verification

To verify the installation:

### 1. Check Files
```bash
# Check MCP configuration
cat .mcp.json

# Check skills installation
ls ~/.claude/skills/
```

### 2. Restart Claude Code
**IMPORTANT**: You need to restart Claude Code for the MCP server to be loaded.

### 3. Test Connection
After restarting, you can test the connection by asking Claude Code:
- "List my n8n workflows"
- "Show me the workflows in my n8n instance"
- "Create a simple webhook workflow in n8n"

### 4. Verify Tools Are Available
The following n8n-mcp tools should be available:
- `n8n_list_workflows` - List workflows in your instance
- `n8n_get_workflow` - Get workflow details
- `n8n_create_workflow` - Create new workflows
- `n8n_update_partial_workflow` - Update existing workflows
- `n8n_validate_workflow` - Validate workflow configurations
- `n8n_deploy_template` - Deploy templates to your instance
- And many more...

### 5. Common Issues

**If tools are not available:**
- Ensure you've restarted Claude Code
- Check that `.mcp.json` is in the project root
- Verify Node.js is installed (`node --version`)
- Check the Claude Code logs for MCP connection errors

**If connection fails:**
- Verify the ngrok URL is still active
- Check that the API key is valid (expires: 2026-02-16)
- Ensure your n8n instance is running and accessible

## Resources

- n8n-mcp GitHub: https://github.com/czlonkowski/n8n-mcp
- n8n-skills GitHub: https://github.com/czlonkowski/n8n-skills
- n8n-mcp Dashboard (hosted service): https://dashboard.n8n-mcp.com
- n8n Documentation: https://docs.n8n.io
