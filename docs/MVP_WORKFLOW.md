# Payment Notice Automation - n8n Workflow Setup (Complete MVP)

**Builder:** AltraDimension (building automation for Altra-CPG)
**Company:** Altra-CPG
**Status:** MVP Development - Build workflows from scratch

> **Note:** This document specifies the MVP workflow requirements. Existing workflow JSON files in this project are outdated and should not be used. Build all workflows from scratch using n8n MCP tools and skills.

## Relationship to Demo Workflow

**This document (MVP_WORKFLOW.md) is a superset of [DEMO_WORKFLOW.md](DEMO_WORKFLOW.md).**

| Feature | Demo | MVP |
|---------|------|-----|
| Email trigger & processing | ✅ | ✅ |
| Multi-format attachment extraction | ✅ | ✅ |
| AI extraction of deduction claims | ✅ | ✅ |
| RAG validation against contracts | ✅ | ✅ |
| Telegram notifications | ✅ | ✅ |
| Telegram Q&A | ✅ | ✅ |
| Context storage for Q&A | ✅ | ✅ |
| **Google Drive storage for audit** | ❌ | ✅ |
| **Decision routing (auto/manual/dispute)** | ❌ | ✅ |
| **Low-confidence routing to human review** | ❌ | ✅ |
| **Auto-approval email responses** | ❌ | ✅ |
| **Configurable approval thresholds** | ❌ | ✅ |
| **Dispute package compilation for review** | ❌ | ✅ |

**Start with:** [DEMO_WORKFLOW.md](DEMO_WORKFLOW.md) for core functionality and detailed node specifications.
**Then add:** The additional MVP features documented below.

## Overview

This n8n workflow automates the processing of payment notices with deductions, validating claims against contracts using AI/RAG, and routing decisions for auto-approval, manual review, or dispute.

## Architecture

```
WORKFLOW A: Payment Notice Processing

Email Trigger → Process Attachments → Store Documents
    → Extract Data (AI) → Query RAG (Contracts/Rules)
    → Validate Claims (AI) → Route Decision
    → [Auto-Approve | Manual Review | Dispute] → Telegram Notification
    → [Email Response] → Store Context

WORKFLOW B: Telegram Q&A

Telegram Trigger → Retrieve Stored Context
    → Claude Answer Question → Reply to Telegram
```

## Key Features

- **Email Trigger**: Monitors `accounts-receiveable@altra-cpg.com` for payment notices
- **Document Processing**: Downloads and stores all attachments to Google Drive/SharePoint
- **AI Extraction**: Uses LLM to extract claims, deductions, and evidence from documents (100% recall requirement)
- **RAG Validation**: Queries vector store with business rules and contracts (swappable with AltraDB)
- **Decision Routing**: Auto-approve, manual review, or dispute based on confidence and thresholds
- **Telegram Integration**: Real-time notifications to `n8ntest` channel
- **Telegram Q&A**: Ask questions about payment notices, deductions, and decisions directly in the notification channel; Claude provides contextual answers based on stored payment context
- **Error Handling**: Automatic error notifications with troubleshooting suggestions

## Prerequisites

### 1. n8n Instance
- n8n self-hosted or cloud instance
- Version: 1.0.0 or higher

### 2. Required Credentials

Configure these credentials in n8n:

#### Email (Gmail/Outlook)
- **Credential Type**: IMAP / Gmail OAuth2
- **Email**: `accounts-receiveable@altra-cpg.com`
- **Purpose**: Receive payment notices

#### Google Drive (or SharePoint)
- **Credential Type**: Google Drive OAuth2
- **Purpose**: Store emails and attachments for audit
- **Required Scope**: `drive.file`

#### Anthropic Claude
- **Credential Type**: Anthropic API
- **Purpose**: AI extraction and validation
- **Models**: claude-3-5-sonnet-20241022 or claude-3-7-sonnet-20250219 recommended
- **Alternative**: Test with local models (Qwen3, DeepSeek on Ollama) for cost reduction

#### Telegram
- **Credential Type**: Telegram API
- **Purpose**: Notifications to `n8ntest` channel
- **Setup**: Create a Telegram bot via @BotFather and get the bot token
- **Get Chat ID**: Add bot to `n8ntest` channel and get the channel chat ID

#### Gmail (Sending)
- **Credential Type**: Gmail OAuth2
- **Email**: `accounts-receiveable@altra-cpg.com`
- **Purpose**: Send auto-approval confirmations

### 3. Environment Variables

Set these in n8n Settings → Variables:

```bash
# Storage
GDRIVE_FOLDER_ID=<your-google-drive-folder-id>

# RAG Vector Store
VECTOR_STORE_INDEX=<your-vector-store-index-name>

# Business Logic
AUTO_APPROVE_THRESHOLD=5000  # Auto-approve only if amount < $5000

# Telegram
TELEGRAM_CHAT_ID=<telegram-channel-chat-id>  # Chat ID for n8ntest channel
```

## Building Workflows from Scratch

### Using n8n MCP Tools

Build workflows programmatically using these tools:

1. **Create workflow**: `n8n_create_workflow({name: "Payment Notice Processing", nodes: [...], connections: {...}})`
2. **Add nodes**: `n8n_update_partial_workflow({id: "...", operations: [{type: "addNode", ...}]})`
3. **Validate**: `n8n_validate_workflow({id: "..."})`
4. **Auto-fix issues**: `n8n_autofix_workflow({id: "..."})`
5. **Test**: `n8n_test_workflow({workflowId: "..."})`

### Using n8n Skills

Invoke skills for guidance:
- `/n8n-workflow-patterns` - Architecture guidance
- `/n8n-node-configuration` - Node setup help
- `/n8n-code-javascript` - Code node syntax

### Set Up RAG Vector Store

Before the workflow can validate claims, you need to initialize the RAG vector store with business rules and contracts.

#### Option A: Use n8n RAG Template

1. Import the RAG setup workflow: https://n8n.io/workflows/5010-rag-starter-template-using-simple-vector-stores-form-trigger-and-openai/
2. Upload these documents to the vector store:
   - Dollar General Vendor Guide
   - Altra-CPG business rules for deductions
   - Altra-CPG/Dollar General contracts
   - Vendor agreements and terms

#### Option B: Manual Setup

1. Create a separate n8n workflow with:
   - **Trigger**: Manual/Webhook
   - **Read Binary Files**: Load contract PDFs/documents
   - **Extract Text**: Convert to text
   - **Vector Store**: Store embeddings
2. Run workflow to populate vector store

#### Documents to Load (MVP)

From the spec references:
- [DG Vendor Guide](https://drive.google.com/file/d/14bzHclECCRK5VsBdX8mW9uHOE9hVKgan/view?usp=drive_link)
- Business rules for deductions (Altra-CPG-specific)
- Altra-CPG/Dollar General contracts
- Vendor agreements and terms

### Configure Telegram Channel

1. Create a Telegram bot:
   - Message @BotFather on Telegram
   - Send `/newbot` command
   - Follow prompts to create bot and get bot token
2. Add bot to `n8ntest` channel
3. Get the chat ID:
   - Send a message to the channel
   - Visit `https://api.telegram.org/bot<BOT_TOKEN>/getUpdates`
   - Find the chat ID in the response
4. Set `TELEGRAM_CHAT_ID` environment variable with the chat ID

### Test Workflow

#### Test Email
1. Send a test payment notice to `accounts-receiveable@altra-cpg.com`
2. Include sample attachments (PDF remittance advice)
3. Monitor workflow execution in n8n

#### Expected Flow
1. ✅ Email received
2. ✅ Attachments stored to Google Drive
3. ✅ Claims extracted via AI
4. ✅ RAG query retrieves relevant contracts
5. ✅ Validation determines decision
6. ✅ Telegram notification sent
7. ✅ (If auto-approve) Email response sent

## Workflow Nodes Explained

### Core Processing

| Node | Purpose | Notes |
|------|---------|-------|
| **Email Trigger** | Monitors email inbox | Polls every 1 min; configurable |
| **Extract Email Metadata** | Parse email details | Subject, sender, attachments, timestamps |
| **Process Attachments** | Split attachments for processing | Handles: txt, Word, CSV, Excel, PDF, JPEG |
| **Store to Google Drive** | Archive for audit | Configurable: GDrive or SharePoint |
| **Extract Claims & Evidence** | AI-powered data extraction | 100% recall requirement |

### AI/RAG Components

| Node | Purpose | Configuration |
|------|---------|---------------|
| **RAG: Query Contracts** | Retrieve relevant business rules | Vector store index from env var |
| **Validate Claims (AI Agent)** | Decision engine | Returns: decision, confidence, reasoning |

### Decision Routing

| Node | Decision Type | Action |
|------|---------------|--------|
| **Route: Auto-Approve** | Valid (high conf) + amount < threshold | → Telegram notification → Email vendor |
| **Route: Manual Approve** | Valid (low conf) OR amount ≥ threshold | → Telegram notification for review |
| **Route: Dispute** | Invalid claims | → Telegram notification with dispute package |

### Error Handling

- **Error Handler** - Catches all workflow errors
- **Telegram: Error Notification** - Notifies team with troubleshooting suggestions

## Decision Logic

### Auto-Approve
```
Deduction is valid (high confidence)
AND
Amount < $AUTO_APPROVE_THRESHOLD
```

**Actions**:
1. Telegram notification to `n8ntest` channel
2. Email sent to vendor confirming approval
3. CC `accounts-receiveable@altra-cpg.com`

### Manual Approve
```
Deduction is valid (low confidence)
OR
(Deduction is valid (high confidence) AND Amount ≥ $AUTO_APPROVE_THRESHOLD)
```

**Actions**:
1. Telegram notification to `n8ntest` channel
2. Includes: decision summary, reasoning
3. Approver can review/approve/reject (MVP: manual action outside workflow)

### Dispute
```
Deduction is invalid
```

**Actions**:
1. Telegram notification to `n8ntest` channel
2. Includes: dispute package details, reasoning
3. Approver reviews and approves response (MVP: manual action outside workflow)

## MVP Scope vs. Future Enhancements

### MVP Includes ✅

- Email trigger and attachment processing
- Google Drive storage for audit
- AI extraction of claims and evidence (100% recall requirement)
- RAG-based validation against contracts
- Decision routing (auto/manual/dispute)
- Telegram notifications with decision summaries
- **Telegram Q&A** - Ask questions about payment notices/deductions in the notification channel
- Auto-approval email responses
- Error handling with notifications
- Context storage for Q&A (30-day retention)

### Context Storage for Q&A

The MVP extends the Demo workflow's context storage (see [DEMO_WORKFLOW.md](DEMO_WORKFLOW.md#node-7-store-context-for-qa)) with additional fields for decision routing and audit trail:

```json
{
  // All fields from Demo workflow context, plus:

  "decision": {
    "type": "auto_approve | manual_review | dispute",
    "confidence": "high | low",
    "threshold_amount": 5000.00,
    "auto_approved": true,
    "reasoning": "All deductions valid with high confidence, total $1,500 under threshold"
  },
  "google_drive": {
    "folder_id": "1abc123...",
    "folder_url": "https://drive.google.com/drive/folders/1abc123...",
    "files": [
      {
        "filename": "remittance.pdf",
        "file_id": "1xyz789...",
        "file_url": "https://drive.google.com/file/d/1xyz789..."
      }
    ]
  },
  "actions_taken": {
    "telegram_notification_sent": true,
    "telegram_msg_id": "12345",
    "email_response_sent": true,
    "email_response_to": "apvendor@dollargeneral.com",
    "email_response_subject": "RE: Payment Processed - PO DG-892374 - Approved",
    "email_response_timestamp": "2026-01-05T14:35:00Z"
  },
  "dispute_package": {
    "generated": false,
    "deductions_to_dispute": ["1", "3"],
    "total_dispute_amount": 625.00,
    "contract_references": ["DG Vendor Guide Section 4.2", "DG Vendor Guide Section 5.3"],
    "counter_evidence_summary": "Routing documentation shows correct DC specified in PO..."
  }
}
```

**MVP Q&A Examples:**
- "Was this payment auto-approved?" → Uses `decision.type` and `decision.auto_approved`
- "Why was it sent for manual review?" → Uses `decision.reasoning` and `decision.confidence`
- "Where are the attachments stored?" → Uses `google_drive.folder_url` and `google_drive.files`
- "Did we send a response email?" → Uses `actions_taken.email_response_sent`
- "What deductions should we dispute?" → Uses `dispute_package.deductions_to_dispute`
- "What's the total amount to dispute?" → Uses `dispute_package.total_dispute_amount`

### MVP Excludes (P2) ⏳

- ❌ Email relevance validation (assumes all emails are payment notices)
- ❌ Download links in email body (only attachments)
- ❌ Explicit data extraction step (done in prompt)
- ❌ ERP system updates (NetSuite)
- ❌ Interactive approval buttons in Telegram
- ❌ Learning from manual corrections
- ❌ Pattern analysis and contract negotiation suggestions
- ❌ Editing/refining static content (only create/delete)

## Customization

### Swap RAG for AltraDB

The workflow is designed to easily swap the vector store:

1. Replace the **RAG: Query Contracts & Rules** node
2. Update to AltraDB connection
3. Maintain same output schema
4. Benchmark: accuracy, cost, latency

### Use Local Models

To reduce costs, test open-source models as alternatives to Claude:

1. Install Ollama locally or use Ollama Cloud
2. Update **Claude AI Agent** nodes to use Ollama HTTP endpoint
3. Test models:
   - Qwen 3
   - DeepSeek
   - Kimi K2

Example configuration:
```json
{
  "baseURL": "http://localhost:11434/v1",
  "model": "qwen3:latest"
}
```

### Adjust Thresholds

Modify `AUTO_APPROVE_THRESHOLD` environment variable to change auto-approval limit.

### Branding for Other Customers

1. Update email addresses in:
   - Email Trigger node
   - Email Response nodes
2. Update Telegram channel chat ID
3. Customize email templates
4. Update company-specific business rules in RAG

## Troubleshooting

### Workflow Not Triggering

- ✅ Check email credentials are valid
- ✅ Verify IMAP settings (Gmail: enable IMAP, app password)
- ✅ Check workflow is **Active**

### Attachments Not Downloading

- ✅ Check attachment file formats are supported
- ✅ Verify Google Drive credentials and folder permissions

### AI Extraction Failing

- ✅ Verify Anthropic API key is valid
- ✅ Check quota/rate limits
- ✅ Review attachment content (text-readable?)
- ✅ Test with simpler documents

### RAG Returning No Results

- ✅ Verify vector store is initialized with documents
- ✅ Check vector store index name matches env var
- ✅ Test RAG query separately
- ✅ Ensure embeddings model is consistent

### Telegram Notifications Not Sent

- ✅ Check Telegram bot token is valid
- ✅ Verify `TELEGRAM_CHAT_ID` is correct
- ✅ Ensure bot is added to `n8ntest` channel
- ✅ Test bot by sending a message manually

### Processing Takes Too Long

- ✅ Check AI/LLM response times
- ✅ Optimize RAG query (reduce topK)
- ✅ Consider switching to faster model

## References

### n8n Workflow Templates

1. [Automated Invoice Workflow](https://n8n.io/workflows/6515-automated-invoice-workflow-with-smart-reminders-using-gpt-4-stripe-and-google-workspace/)
2. [AI-Powered Damage Reporting](https://n8n.io/workflows/11048-ai-powered-damage-reporting-tool-for-logistics-with-gmail-telegram-and-gpt/)
3. [RAG Starter Template](https://n8n.io/workflows/5010-rag-starter-template-using-simple-vector-stores-form-trigger-and-openai/)
4. [Vector Store with Q&A](https://n8n.io/workflows/5011-save-costs-in-rag-workflows-using-the-qanda-tool-with-multiple-models)

### Documentation

- [n8n RAG Guide](https://docs.n8n.io/advanced-ai/rag-in-n8n/)
- [n8n Workflow Documentation](https://docs.n8n.io/)

### Specification Documents

- Dollar General Vendor Guide
- Altra-CPG Business Rules
- Altra-CPG/Dollar General Contracts

## Support

For issues or questions:
1. Check n8n logs: Workflow → Executions
2. Check Telegram `n8ntest` channel for error notifications
3. Contact AltraDimension support (info@altradimension.com)

## Future Roadmap

1. **Telegram Interactive Approvals**: Inline keyboards with approve/reject buttons
2. **ERP Integration**: Auto-update NetSuite with approved deductions
3. **Learning Loop**: Use manual corrections to improve AI
4. **Pattern Analysis**: Identify recurring issues, suggest contract changes
5. **AltraDB Integration**: Replace RAG with AltraDB for better performance
6. **Multi-format Support**: Handle download links, scanned documents (OCR)
7. **Advanced Validation**: Email relevance filtering, sender verification
