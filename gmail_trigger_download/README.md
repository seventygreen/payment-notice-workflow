# Gmail Payment Email Processor - n8n Workflow

This n8n workflow automatically processes emails sent to `michael+payments@seventyblue.com`, downloads attachments, identifies their types, and prepares the data for an agent call.

## Features

1. **Gmail Trigger**: Monitors for new emails every minute
2. **Email Filtering**: Filters emails sent to `michael+payments@seventyblue.com`
3. **Attachment Download**: Downloads all attachments from filtered emails
4. **Attachment Type Identification**: Identifies file types (PDF, Image, Spreadsheet, Word, Archive, Text, JSON, XML, etc.)
5. **Agent Context Preparation**: Structures email and attachment data for agent processing

## Workflow Structure

The workflow consists of 6 nodes:

1. **Gmail Trigger** - Monitors Gmail for new emails
2. **Filter Email Address** - Filters emails to the specific address
3. **Get Email with Attachments** - Retrieves full email data including attachments
4. **Process & Identify Attachments** - Identifies attachment types and prepares metadata
5. **Prepare Agent Context** - Structures the data for the agent
6. **Call Agent** - Sends the prepared context to your agent API

## Setup Instructions

### 1. Import the Workflow

1. Open your n8n instance
2. Click on "Workflows" → "Import from File"
3. Select the `workflow.json` file
4. The workflow will be imported with all nodes configured

### 2. Configure Gmail Credentials

1. Click on the **Gmail Trigger** node
2. Click on "Credential to connect with" → "Create New"
3. Select "Gmail OAuth2 API"
4. Follow the OAuth2 setup process:
   - You'll need to create OAuth2 credentials in Google Cloud Console
   - Add the redirect URI provided by n8n
   - Authorize the application
5. Repeat for the **Get Email with Attachments** node

### 3. Configure the Agent Endpoint

1. Click on the **Call Agent** node
2. Set the **URL** field to your agent API endpoint (e.g., `https://your-agent-api.com/process-email`)
3. Configure authentication if required:
   - Add headers for API keys
   - Or configure OAuth2/Basic Auth as needed
4. The request will be sent as a POST with JSON body containing:
   ```json
   {
     "email": {
       "id": "...",
       "subject": "...",
       "from": "...",
       "to": "...",
       "date": "...",
       "body": "...",
       "bodyHtml": "...",
       "bodyText": "..."
     },
     "attachments": [
       {
         "filename": "invoice.pdf",
         "mimeType": "application/pdf",
         "size": 12345,
         "fileType": "PDF",
         "fileCategory": "document",
         "fileExtension": "pdf",
         "attachmentId": "...",
         "binaryPropertyName": "attachment_0",
         "hasBinaryData": true
       }
     ],
     "attachmentCount": 1,
     "hasAttachments": true
   }
   ```

### 4. Activate the Workflow

1. Toggle the workflow to "Active" in the top right
2. The workflow will now monitor Gmail every minute for new emails

## Attachment Type Detection

The workflow identifies the following file types:

- **PDF** - PDF documents
- **Image** - JPG, JPEG, PNG, GIF, BMP, WEBP, SVG
- **Spreadsheet** - XLS, XLSX, CSV
- **Word Document** - DOC, DOCX
- **Archive** - ZIP, RAR, 7Z, TAR, GZ
- **Text** - TXT, LOG
- **JSON** - JSON files
- **XML** - XML files
- **Other** - Any other file types

Each attachment includes:
- Filename
- MIME type
- File size
- File type (human-readable)
- File category
- File extension
- Binary data reference

## Data Structure

The agent receives a structured context object with:

- **email**: Complete email metadata (subject, from, to, date, body in multiple formats)
- **attachments**: Array of attachment metadata with type identification
- **attachmentCount**: Number of attachments
- **hasAttachments**: Boolean indicating if attachments exist

## Notes

- The workflow polls Gmail every minute. Adjust the polling interval in the Gmail Trigger node if needed.
- Binary attachment data is available in the workflow but sent as metadata to the agent. If your agent needs the actual file content, you may need to:
  - Send attachments as base64-encoded strings
  - Upload attachments to cloud storage and send URLs
  - Use a different approach based on your agent's requirements
- The filter node provides a double-check to ensure only emails to the specified address are processed.

## Troubleshooting

- **No emails triggering**: Check Gmail credentials are properly configured and authorized
- **Attachments not downloading**: Verify the Gmail account has permission to read emails and attachments
- **Agent not receiving data**: Check the HTTP Request node URL and authentication settings
- **File types not identified**: The workflow uses MIME types and file extensions. Some files may be categorized as "Unknown" if neither is available.


