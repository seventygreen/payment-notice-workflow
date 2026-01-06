# Payment Notice Workflow Plan (Demo)

**Last Updated:** 2026-01-06
**Status:** Planning Phase
**Company:** Altra-CPG
**Initial Distributor:** Dollar General
**Demo audience (preliminary):** Extraordinary AI, VCs

---

## Business Context

**Company:** Altra-CPG (coffee products manufacturer)
**Distributors:** Dollar General, Kroger, Walmart, Sam's Club
**Focus:** Start with Dollar General automation

### Current Manual Process

1. Dollar General sends payment notices to `accounts-receivable@altra-cpg.com`
2. Email contains remittance advice (email body OR attachments)
3. If deductions claimed, evidence is in email body OR attachments
4. AR team at Altra-CPG reviews deductions against vendor guides/contracts
5. When Altra-CPG disagrees → create dispute package with explanation + evidence

---

## Demo Scope

### Included ✅

- Email trigger and processing
- Attachment extraction (multi-format: PDF, DOCX, Excel, images)
- Remittance and deduction parsing
- RAG query for Dollar General contract terms
- Claude AI Agent validation of deductions
- Telegram notification with summary
- Interactive Q&A via Telegram (ask Claude about payment notices)
- Minimal context storage (for Q&A only)

### Deferred to Phase 2 ⏸️

- Dispute package generation
- ERP/NetSuite integration
- Auto-approve workflows
- Comprehensive audit trail
- System monitoring & alerts
- Detailed logging
- Additional vendors (Kroger, Walmart, Sam's Club)

---

## Workflow Architecture

```
WORKFLOW A: Payment Notice Processing

1. Gmail Trigger
   ↓
2. Extract Email & Attachments
   ↓
3. Parse Remittance & Deductions
   ↓
4. Query Vector Store (RAG - Dollar General contracts)
   ↓
5. Claude AI Agent Validation
   ↓
6. Format & Send Telegram Message
   ↓
7. Store Context (for Q&A)

---

WORKFLOW B: Telegram Q&A

1. Telegram Message Trigger (user asks question)
   ↓
2. Retrieve Stored Context
   ↓
3. Claude Answer Question
   ↓
4. Reply to Telegram
```

---

## Detailed Node Specifications

### **Node 1: Gmail Trigger**

**Purpose:** Detect incoming Dollar General payment notices

**Configuration:**
- Email: `michael+payments@seventyblue.com` (for demo: `accounts-receiveable@altra-cpg.com` or `accounts-receivable@altradimension.com`)
- Download attachments: Yes
- Poll frequency: Every 5 minutes

**Output:**
- Email metadata (sender, subject, date)
- Email body (HTML and plain text)
- All attachments (binary data)

---

### **Node 2: Extract Email & Attachments**

**Purpose:** Parse email and extract content from all attachment formats

**Single Code Node:**

**A. Extract Email Basics:**
- From, subject, date
- Body (plain text preferred, HTML if needed)
- Reference numbers (check #, PO #, invoice #)

**B. Process Each Attachment:**

| Format | Method | Output |
|--------|--------|--------|
| **PDF** | pdf-parse library | Extracted text |
| **Excel/CSV** | xlsx library | Parsed JSON data |
| **Images (JPG, PNG)** | Base64 encoding | Base64 string for Claude vision |
| **DOCX** | mammoth library | Extracted text |

**npm Packages Required:**
```bash
npm install pdf-parse xlsx mammoth
```

**Output Schema:**
```json
{
  "email": {
    "from": "apvendor@dollargeneral.com",
    "subject": "Payment Processed with Chargebacks - PO DG-892374",
    "date": "2026-01-05",
    "body": "..."
  },
  "attachments": [
    {
      "filename": "remittance.pdf",
      "type": "pdf",
      "content": "extracted text..."
    },
    {
      "filename": "photo.jpg",
      "type": "image",
      "base64": "..."
    }
  ]
}
```

---

### **Node 3: Parse Remittance & Deductions**

**Purpose:** Extract payment details and deduction claims

**Single Code Node:**

**A. Find Remittance Data:**
- Search email body first
- Then search attachments (PDF, Excel most common)
- Extract:
  - Invoice numbers being paid
  - Gross invoice amounts
  - Deduction amounts by type
  - Net payment amount
  - Check number / ACH reference
  - Payment date

**B. Identify Deductions:**
- Parse each deduction line
- Extract: type, code, amount, invoice, description
- Link to supporting evidence (attachments)
- Build structured deduction list

**Common Dollar General Deduction Types:**
- **D-001:** Incorrect DC violation
- **Q-002:** Short quantity violation
- **U-003:** Missing UPC code
- Rework charges (unit level)
- Re-handling charges (case level)
- Damaged product disposal
- Compliance chargebacks

**C. Format for Claude:**
- Combine email, attachments, structured data
- Create comprehensive prompt context

**Output Schema:**
```json
{
  "check_number": "DG-CHK-2025-784392",
  "check_date": "2025-12-19",
  "gross_amount": 28450.00,
  "total_deductions": 2700.00,
  "net_payment": 25750.00,
  "deductions": [
    {
      "id": "1",
      "type": "incorrect_dc",
      "code": "D-001",
      "amount": 250.00,
      "description": "Incorrect DC Violation",
      "backup_reference": "See Distribution_Center_Routing_Error_Documentation.docx"
    },
    {
      "id": "2",
      "type": "short_quantity",
      "code": "Q-002",
      "amount": 375.00,
      "description": "Short Quantity Violation - 25 cases missing",
      "backup_reference": "See Receiving_Inspection_Report.docx"
    }
  ]
}
```

---

### **Node 4: Query Vector Store (RAG)**

**Purpose:** Retrieve relevant Dollar General contract terms

**Implementation Options:**

**Option A: Direct API Call (Qdrant)**
```javascript
// HTTP Request Node
POST http://localhost:6333/collections/dollar_general_contracts/points/search
{
  "vector": [embeddings from deduction query],
  "filter": {
    "must": [
      {"key": "vendor_id", "match": {"value": "DOLLAR_GENERAL"}}
    ]
  },
  "limit": 10,
  "score_threshold": 0.7
}
```

**Option B (preferred): With Embedding Step** 
1. Generate embedding for deduction query (OpenAI API)
2. Vector search with embedding

**Query Construction:**
```javascript
const queryText = `
Dollar General deductions:
Types: ${deduction_types.join(', ')}
Codes: ${deduction_codes.join(', ')}
Total amount: $${total_deductions}
Claims: ${deductions.map(d => d.description).join('; ')}
`;
```

**Output Schema:**
```json
{
  "contract_clauses": [
    {
      "text": "Section 4.2 - Incorrect DC Violations: $250 flat fee per violation...",
      "score": 0.92,
      "metadata": {
        "document": "DG Domestic Vendor Guide",
        "section": "4.2",
        "page": 34,
        "topic": "compliance_chargebacks"
      }
    }
  ]
}
```

---

### **Node 5: Claude AI Agent - Validate Deductions**

**Purpose:** Analyze each deduction against Dollar General contract terms

**Configuration:**
- **Model:** `claude-3-5-sonnet-20241022` or `claude-3-7-sonnet-20250219`
- **Temperature:** 0.1 (low for consistency)
- **Max Tokens:** 4096
- **Tools Enabled:**
  - Vision (for damage photos, scanned docs)
  - Calculator (for amount verification)
  - Web Search (optional, for rate verification)

**System Prompt:**
```
You are a payment validation specialist for Altra-CPG analyzing Dollar General payment notices.

Your task: Validate each deduction claim against Dollar General's vendor agreements and compliance guides.

For each deduction:
1. Review the claim details (type, amount, description)
2. Compare against the relevant contract terms (provided from RAG)
3. Examine any evidence provided (photos, documents, reports)
4. Verify calculations and amounts
5. Check procedural compliance (deadlines, documentation requirements)

Categorize each deduction:
- VALID: Legitimate per contract terms
- INVALID: Violates contract terms or procedures
- INSUFFICIENT_EVIDENCE: May be valid but lacking required documentation
- NEEDS_REVIEW: Ambiguous or requires human judgment

Provide clear reasoning for each decision, citing specific contract sections.
```

**User Prompt Template:**
```
Payment Notice from Dollar General

Check #: {check_number}
Date: {check_date}
Gross Amount: ${gross_amount}
Total Deductions: ${total_deductions}
Net Payment: ${net_payment}

DEDUCTIONS CLAIMED:
{for each deduction:}
{id}. {type} ({code}) - ${amount}
Description: {description}
Backup: {backup_reference}

DOLLAR GENERAL CONTRACT TERMS (from RAG):
{for each clause:}
---
Document: {document_name}
Section: {section_number}
Text: {clause_text}
---

ATTACHMENTS PROVIDED:
{for each attachment:}
- {filename} ({type})
  {if image: [IMAGE - analyze with vision]}
  {if text: Content: {content}}

Please validate each deduction and return structured JSON with your analysis.
```

**Expected Response Format:**
```json
{
  "summary": {
    "total_deductions": 6,
    "total_amount": 2700.00,
    "valid_count": 3,
    "valid_amount": 1500.00,
    "invalid_count": 2,
    "invalid_amount": 625.00,
    "needs_review_count": 1,
    "needs_review_amount": 575.00
  },
  "deductions": [
    {
      "id": "1",
      "type": "incorrect_dc",
      "code": "D-001",
      "amount": 250.00,
      "status": "INVALID",
      "reasoning": "Per DG Vendor Guide Section 4.2, Incorrect DC violations require proof of routing instructions. The attached documentation shows Altra-CPG followed the routing guide correctly (Jonesville, SC DC specified in PO). Dollar General's delivery to Alachua, FL DC was their carrier's error, not vendor's fault.",
      "contract_reference": "DG Domestic Vendor Guide, Section 4.2, Page 34",
      "evidence_assessment": "Distribution_Center_Routing_Error_Documentation.docx shows correct routing per PO instructions",
      "recommendation": "Dispute this deduction - not vendor's fault"
    },
    {
      "id": "2",
      "type": "short_quantity",
      "code": "Q-002",
      "amount": 375.00,
      "status": "VALID",
      "reasoning": "Per DG Vendor Guide Section 5.1, short shipments are vendor's responsibility. Receiving_Inspection_Report.docx confirms 25 cases shortage (ordered 960, received 935). This falls within contract terms for quantity violations.",
      "contract_reference": "DG Domestic Vendor Guide, Section 5.1",
      "evidence_assessment": "Complete - receiving report with signatures confirms shortage",
      "recommendation": "Accept deduction - legitimate shortage claim"
    }
  ],
  "overall_assessment": "3 deductions are valid ($1,500). 2 should be disputed ($625) due to procedural violations. 1 requires human review ($575).",
  "flags": [
    "Incorrect DC chargeback appears to be DG carrier error, not vendor fault",
    "Missing UPC chargeback requires verification of UPC requirements in original PO"
  ]
}
```

---

### **Node 6: Format & Send Telegram Message**

**Purpose:** Notify AR team with actionable summary

**Code Node - Format Message:**
```javascript
const message = `
🔔 **Dollar General Payment Processed**

📅 Date: ${check_date}
💰 Check: ${check_number}

**Summary**
Gross: $${gross_amount.toFixed(2)}
Deductions: $${total_deductions.toFixed(2)}
Net: $${net_payment.toFixed(2)}

**Validation Results**
✅ Valid: ${valid_count} ($${valid_amount.toFixed(2)})
❌ Invalid: ${invalid_count} ($${invalid_amount.toFixed(2)})
⚠️ Review: ${review_count} ($${review_amount.toFixed(2)})

**Deduction Details**
${deductions.map(d => `
${statusEmoji(d.status)} ${d.type} - $${d.amount.toFixed(2)}
${d.reasoning.substring(0, 150)}...
`).join('\n')}

**Overall Assessment**
${overall_assessment}

💬 **Ask Questions:** Reply to this message to ask Claude about this payment notice.

Reference ID: \`${payment_id}\`
`;
```

**Telegram Node:**
- **Action:** Send Message
- **Chat ID:** AR team channel
- **Parse Mode:** Markdown
- **Reply Markup:** None (Demo - interactive buttons in Phase 2)

**Output:**
- Telegram message_id (for Q&A threading)

---

### **Node 7: Store Context (for Q&A)**

**Purpose:** Save payment notice data for interactive Q&A

**Storage Options:**

**Option A: Database (PostgreSQL)**
```sql
CREATE TABLE payment_contexts (
  payment_id VARCHAR(255) PRIMARY KEY,
  telegram_msg_id VARCHAR(255),
  created_at TIMESTAMP DEFAULT NOW(),
  expires_at TIMESTAMP,
  context JSONB
);
```

**Option B: Redis**
```javascript
SET payment:{payment_id} {full_json_context}
EXPIRE payment:{payment_id} 2592000  // 30 days
```

**Context Data Stored:**
```json
{
  "payment_id": "DG_20260105_784392",
  "telegram_msg_id": "12345",
  "timestamp": "2026-01-05T14:30:00Z",
  "email": {full email data},
  "remittance": {parsed remittance},
  "deductions": {all deduction details},
  "contract_clauses": {RAG results},
  "validation_results": {Claude's analysis},
  "attachments": {processed attachments}
}
```

---

## Workflow B: Interactive Q&A

### **Node 1: Telegram Message Trigger**

**Purpose:** Detect when user asks a question

**Configuration:**
- **Trigger:** New message in AR channel
- **Filter:**
  - Replies to bot's payment notice messages
  - OR messages containing payment_id reference
  - OR @mentions of bot

---

### **Node 2: Retrieve Stored Context**

**Purpose:** Get payment notice data for Claude

**Logic:**
```javascript
// Extract payment_id from message or reply
const payment_id = extractPaymentId(message);

// Query storage
const context = await getContext(payment_id);

if (!context) {
  return { error: "Payment notice not found or expired" };
}
```

---

### **Node 3: Claude Answer Question**

**Purpose:** Provide contextual answers

**System Prompt:**
```
You previously analyzed a Dollar General payment notice for Altra-CPG AR team.

The AR team member has a question about this payment notice. Answer using:
- The payment notice details
- Your previous validation analysis
- The Dollar General contract terms
- Any attachments or evidence

Be helpful, concise, and specific. Reference relevant details.
```

**User Prompt:**
```
PAYMENT NOTICE CONTEXT:
{stored context}

YOUR PREVIOUS ANALYSIS:
{validation results}

USER QUESTION:
{user's message}

Answer the question clearly and concisely.
```

**Example Q&A:**
- Q: "Why did you mark the DC violation as invalid?"
- A: "The Incorrect DC violation (D-001, $250) is invalid because the documentation shows Altra-CPG followed the routing instructions correctly. The PO specified delivery to Jonesville, SC DC, but Dollar General's carrier delivered to Alachua, FL instead. Per Section 4.2 of the DG Vendor Guide, this type of carrier error is not the vendor's responsibility."

---

### **Node 4: Reply to Telegram**

**Purpose:** Send answer to user

**Configuration:**
- **Reply To:** User's message (threaded)
- **Parse Mode:** Markdown

---

## Configuration & Settings

### **Easy-to-Update Config**

```javascript
const CONFIG = {
  // Email
  PAYMENT_EMAIL: 'michael+payments@gmail.com',

  // Vendor
  VENDOR: 'Dollar General',
  DG_EMAIL_ADDRESSES: [
    'apvendor@dollargeneral.com',
    'vnc@dollargeneral.com',
    'remittance@dg.com'
  ],

  // Telegram
  TELEGRAM_ENABLED: true,
  TELEGRAM_CHAT_ID: 'AR_TEAM_CHANNEL_ID',

  // Claude
  CLAUDE_MODEL: 'claude-3-5-sonnet-20241022',
  CLAUDE_TEMPERATURE: 0.1,

  // RAG
  RAG_COLLECTION: 'dollar_general_contracts',
  RAG_TOP_K: 10,
  RAG_MIN_SCORE: 0.7,

  // Storage
  CONTEXT_RETENTION_DAYS: 30,

  // Processing
  MAX_ATTACHMENT_SIZE_MB: 25
};
```

---

## Required Integrations & APIs

### **Essential:**

- ✅ Gmail API (email trigger)
- ✅ Anthropic API (Claude)
- ✅ OpenAI API (embeddings for RAG)
- ✅ Vector Database (Qdrant, Pinecone, or similar)
- ✅ Telegram Bot API
- ✅ Storage (PostgreSQL or Redis)

### **Credentials Setup:**

1. **Gmail:** OAuth2 or App Password
2. **Anthropic:** API key from console.anthropic.com
3. **OpenAI:** API key from platform.openai.com
4. **Telegram:** Bot token from @BotFather
5. **Vector DB:** Connection string/API key

---

## Testing Plan

### **Test Cases:**

**1. Basic Flow**
- Input: Simple payment notice (no deductions)
- Expected: Email processed, Telegram notification sent

**2. With Valid Deductions**
- Input: Payment notice with 2-3 valid deductions
- Expected: Correctly validated as VALID with reasoning

**3. With Invalid Deductions**
- Input: Payment notice with procedural violations
- Expected: Flagged as INVALID with contract references

**4. Mixed Scenario**
- Input: Valid + invalid + ambiguous deductions
- Expected: Proper categorization, clear recommendations

**5. Multi-format Attachments**
- Input: PDF remittance, Excel detail, JPG damage photo, DOCX reports
- Expected: All formats processed, vision analysis works

**6. Telegram Q&A**
- Input: User asks about specific deduction
- Expected: Context retrieved, accurate answer provided

**7. Edge Cases**
- Malformed email
- Corrupt attachment
- API failure
- Expected: Graceful error handling

---

## Success Criteria

✅ Process Dollar General payment notices end-to-end
✅ Extract remittance from email/attachments (PDF, Excel, DOCX, images)
✅ Identify and categorize all deductions
✅ Validate against Dollar General contract terms (RAG)
✅ Send Telegram notification with actionable summary
✅ Enable Q&A about payment notices via Telegram
✅ Store minimal context for Q&A (30 day retention)
✅ Process 80%+ of payments without errors
✅ AR team can interact conversationally with results

---

## Example Email References

Located in: `gmail_trigger_download/examples_payment_notice_emails/`

**Dollar General Example:**
- `Payment Processed with Vendor Compliance Chargebacks - PO DG-892374-2025.eml`
- Contains: 6 deduction types with supporting attachments
- Tests: Multi-format processing, vision analysis, contract validation

**Other Distributors (Phase 2):**
- Kroger: `Payment Notice with Deductions - PO KR-458219.eml`
- Walmart: `Payment Remittance - PO #4582917634.eml`
- Walmart SQEP: `Payment Remittance with SQEP Deductions.eml`
- Sam's Club: `Payment Confirmation - PO SC-7829364.eml`

---

## Next Steps

1. Complete RAG setup (see RAG_SETUP.md)
2. Build n8n Workflow A (payment processing)
3. Build n8n Workflow B (Q&A)
4. Test with Dollar General examples
5. Refine prompts and validation logic
6. Deploy to production

---

## Phase 2 Roadmap

**Dispute Package Generation:**
- Auto-generate dispute emails
- Compile counter-evidence
- Format professional dispute documentation

**ERP Integration:**
- NetSuite API connection
- Auto-post validated deductions
- Update invoice statuses

**Advanced Features:**
- Auto-approve small valid deductions (<$100)
- Comprehensive audit trail
- System monitoring & dashboards
- Multi-vendor support (Kroger, Walmart, etc.)
- Interactive Telegram buttons
- Learning from dispute outcomes

---

**Document Version:** 1.1
**Last Updated:** 2026-01-06
**Status:** Ready for Implementation



