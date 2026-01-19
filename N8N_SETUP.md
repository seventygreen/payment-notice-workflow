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

## Verification

To verify the installation:
1. Check that `.mcp.json` exists in project root
2. Check that skills are in `~/.claude/skills/`
3. Restart Claude Code if needed
4. Try asking about n8n nodes or workflows

## Resources

- n8n-mcp GitHub: https://github.com/czlonkowski/n8n-mcp
- n8n-skills GitHub: https://github.com/czlonkowski/n8n-skills
- n8n-mcp Dashboard (hosted service): https://dashboard.n8n-mcp.com
- n8n Documentation: https://docs.n8n.io
