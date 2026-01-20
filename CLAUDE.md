# CLAUDE.md - Payment Notice Workflow Project

## Project Overview

**Company:** Altra-CPG (built by AltraDimension)
**Purpose:** Automate payment notice processing with AI-powered deduction validation
**Initial Vendor:** Dollar General
**Status:** MVP Development - Build workflows from scratch

This project automates the processing of payment notices with deductions, validating claims against contracts using AI/RAG, and routing decisions for auto-approval, manual review, or dispute.

> **Note:** Existing workflow JSON files in this project are outdated and should not be used. Build all workflows from scratch using n8n MCP tools and skills.

## Key Features

1. **Email Monitoring** - Trigger on new emails to `accounts-receiveable@altra-cpg.com`
2. **Multi-format Attachment Processing** - Extract content from PDF, Excel, DOCX, CSV, and images
3. **AI Deduction Extraction** - Extract all deduction claims from email body and attachments with 100% recall requirement
4. **RAG Validation** - Validate deductions against Dollar General Vendor Guide and contracts
5. **Decision Routing** - Auto-approve, manual review, or dispute based on validation results
6. **Telegram Notifications** - Real-time notifications to `n8ntest` channel with decision summaries
7. **Telegram Q&A** - Ask questions about payment notices, deductions, and decisions directly in the notification channel; Claude provides contextual answers based on stored payment context

## Workflow Architecture

```
WORKFLOW A: Payment Notice Processing

Email Trigger → Extract Email & Attachments → Parse Deductions
    → Query RAG (Dollar General contracts) → Validate Claims (AI)
    → Route Decision → Telegram Notification → [Email Response if auto-approve]
    → Store Context (for Q&A)

WORKFLOW B: Telegram Q&A

Telegram Trigger → Retrieve Stored Context
    → Claude Answer Question → Reply to Telegram
```

---

## n8n Tooling Available

### n8n MCP Server (20 Tools)

The n8n-mcp server provides programmatic access to n8n's workflow automation capabilities.

#### Documentation Tools (7) - Always Available

| Tool | Purpose | Example Usage |
|------|---------|---------------|
| `tools_documentation` | Get docs for any MCP tool | `tools_documentation({topic: "get_node"})` |
| `search_nodes` | Search 800+ nodes by keyword | `search_nodes({query: "telegram"})` |
| `get_node` | Get node schema, docs, properties | `get_node({nodeType: "nodes-base.telegram", mode: "docs"})` |
| `validate_node` | Validate node configuration | `validate_node({nodeType: "nodes-base.set", config: {...}})` |
| `validate_workflow` | Validate workflow offline | `validate_workflow({workflow: {...}})` |
| `get_template` | Get template by ID | `get_template({templateId: 6515})` |
| `search_templates` | Search 2,700+ templates | `search_templates({query: "invoice automation"})` |

#### Management Tools (13) - Requires API Connection

| Tool | Purpose |
|------|---------|
| `n8n_health_check` | Check n8n instance health and API connectivity |
| `n8n_create_workflow` | Create new workflow (returns inactive) |
| `n8n_get_workflow` | Get workflow by ID (full/details/structure/minimal) |
| `n8n_update_full_workflow` | Replace entire workflow |
| `n8n_update_partial_workflow` | Incremental diff-based updates (addNode, removeNode, etc.) |
| `n8n_delete_workflow` | Delete workflow permanently |
| `n8n_list_workflows` | List workflows with filters |
| `n8n_validate_workflow` | Validate workflow by ID (online) |
| `n8n_autofix_workflow` | Auto-fix common issues (expressions, typeVersions) |
| `n8n_test_workflow` | Test/trigger workflow execution (webhook/form/chat) |
| `n8n_executions` | Manage executions (get/list/delete) |
| `n8n_workflow_versions` | Version history and rollback |
| `n8n_deploy_template` | Deploy template directly to n8n instance |

#### Additional Resources
- `ai_agents_guide` - Comprehensive guide for building AI Agent workflows in n8n

### n8n Skills (7 Skills)

Invoke skills with `/skill-name` for specialized guidance.

| Skill | When to Use |
|-------|-------------|
| `/n8n-node-configuration` | Configuring nodes, understanding property dependencies, required fields |
| `/n8n-code-javascript` | Writing JavaScript in Code nodes (`$input`, `$json`, `$node` syntax) |
| `/n8n-code-python` | Writing Python in Code nodes (`_input`, `_json` syntax) |
| `/n8n-expression-syntax` | Validating n8n expressions (`{{ }}` syntax), fixing expression errors |
| `/n8n-workflow-patterns` | Designing workflow architecture, choosing patterns |
| `/n8n-validation-expert` | Interpreting validation errors, fixing issues |
| `/n8n-mcp-tools-expert` | Using n8n-mcp tools effectively, parameter formats |

---

## Key Project Files

| File | Description |
|------|-------------|
| `docs/DEMO_WORKFLOW.md` | **Base workflow** - Detailed node specs, prompts, schemas for core functionality |
| `docs/MVP_WORKFLOW.md` | **Superset of Demo** - Adds Google Drive storage, decision routing, auto-approval |
| `docs/RAG_SETUP.md` | Vector database setup guide (Qdrant/Pinecone) |
| `gmail_trigger_download/examples_payment_notice_emails/` | Sample payment notice emails for testing |
| `gmail_trigger_download/examples_vendor_guides/` | Vendor guides including DG Domestic Vendor Guide |
| `.mcp.json` | MCP server configuration |
| `N8N_SETUP.md` | n8n tooling setup documentation |

### Document Hierarchy

```
DEMO_WORKFLOW.md (Base)
    ├── Email trigger & processing
    ├── Multi-format attachment extraction
    ├── AI extraction of deduction claims
    ├── RAG validation
    ├── Telegram notifications
    ├── Telegram Q&A
    └── Context storage

MVP_WORKFLOW.md (Superset = Demo + Additional Features)
    ├── Everything in Demo
    ├── + Google Drive storage for audit
    ├── + Decision routing (auto/manual/dispute)
    ├── + Low-confidence routing to human review
    ├── + Auto-approval email responses
    ├── + Configurable approval thresholds
    └── + Dispute package compilation for review
```

---

## Development Guidelines

### Building Workflows from Scratch

**Important:** Do not use existing workflow JSON files in this project. Build all workflows from scratch using:

1. **n8n MCP tools** - Use `n8n_create_workflow`, `n8n_update_partial_workflow` to build programmatically
2. **n8n skills** - Invoke `/n8n-workflow-patterns` for architecture guidance
3. **Templates** - Use `search_templates` and `n8n_deploy_template` as starting points

### n8n Expression Syntax
```javascript
// Access JSON fields
{{ $json.field }}
{{ $json.nested.field }}

// Access previous node data
{{ $('NodeName').item.json.field }}

// Current timestamp
{{ $now }}

// Environment variables
{{ $env.VARIABLE_NAME }}
```

### Workflow Development Process
1. **Search** for nodes: `search_nodes({query: "keyword"})`
2. **Get** node details: `get_node({nodeType: "nodes-base.nodeName", detail: "standard"})`
3. **Configure** node with required parameters
4. **Validate** before deployment: `validate_workflow({workflow: {...}})`
5. **Test** with sample data from `gmail_trigger_download/examples_*`

### Best Practices
- Always validate workflows before deployment using `n8n_validate_workflow`
- Use `n8n_autofix_workflow` to fix common issues automatically
- Test with example emails in `gmail_trigger_download/examples_payment_notice_emails/`
- Store payment context for Telegram Q&A feature (30-day retention)
- Ensure 100% recall for deduction extraction (never miss a deduction)

---

## Credentials Required

| Integration | Credential Type | Purpose |
|-------------|-----------------|---------|
| Gmail/IMAP | OAuth2 or IMAP | Receive payment notices |
| Google Drive | OAuth2 | Store attachments for audit |
| Anthropic Claude | API Key | AI extraction and validation |
| OpenAI | API Key | Embeddings for RAG |
| Telegram | Bot Token | Notifications and Q&A |
| Vector DB | API Key/Connection | RAG queries (Qdrant or Pinecone) |

---

## Environment Variables

```bash
# Storage
GDRIVE_FOLDER_ID=<google-drive-folder-id>

# Telegram
TELEGRAM_CHAT_ID=<telegram-channel-chat-id>

# RAG
VECTOR_STORE_INDEX=<vector-store-index-name>

# Business Logic
AUTO_APPROVE_THRESHOLD=5000
```

---

## Decision Logic

| Decision | Condition | Actions |
|----------|-----------|---------|
| **Auto-Approve** | Valid (high confidence) AND amount < threshold | Telegram notify → Email vendor |
| **Manual Approve** | Valid (low confidence) OR amount >= threshold | Telegram notify for review |
| **Dispute** | Invalid claims | Telegram notify with dispute details |

---

## Testing

### Test Emails Available
- `Payment Processed with Vendor Compliance Chargebacks - PO DG-892374-2025.eml` (Dollar General, 6 deduction types)
- `Payment Notice with Deductions - PO KR-458219.eml` (Kroger)
- `Payment Remittance - PO #4582917634.eml` (Walmart)
- `Payment Remittance with SQEP Deductions.eml` (Walmart SQEP)
- `Payment Confirmation - PO SC-7829364.eml` (Sam's Club)

### Vendor Guides Available
- `DG - Domestic Vendor Guide.pdf` (Dollar General - primary)
- `kro_standard_vendor_agreement_merchandising.pdf` (Kroger)
- `walmart - supply-chain-packaging-guide.pdf` (Walmart)
- `sams-club-structural-packaging-standards.pdf` (Sam's Club)

---

## Common Deduction Types (Dollar General)

| Code | Type | Description |
|------|------|-------------|
| D-001 | Incorrect DC | Incorrect distribution center violation |
| Q-002 | Short Quantity | Short shipment violation |
| U-003 | Missing UPC | Missing UPC code |
| - | Rework | Unit-level rework charges |
| - | Re-handling | Case-level re-handling charges |
| - | Damage | Damaged product disposal |

---

## Support & References

- **n8n Documentation:** https://docs.n8n.io
- **n8n RAG Guide:** https://docs.n8n.io/advanced-ai/rag-in-n8n/
- **n8n-mcp GitHub:** https://github.com/czlonkowski/n8n-mcp
- **n8n-skills GitHub:** https://github.com/czlonkowski/n8n-skills
- **AltraDimension Support:** info@altradimension.com
