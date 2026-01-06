# RAG Vector Database Setup Guide

**Last Updated:** 2026-01-06
**Status:** Planning Phase
**Vendor:** Dollar General (Initial Focus)

---

## Overview

This guide provides step-by-step instructions to set up the RAG (Retrieval-Augmented Generation) vector database with Dollar General vendor guides and contracts for automated deduction validation.

---

## Prerequisites

### Required Software

- **Python 3.8+**
- **Vector Database:** Qdrant (recommended) or Pinecone
- **Embedding Model API:** OpenAI, Voyage AI, or Cohere

### Required Documents

Located in: `gmail_trigger_download/examples_vendor_guides/`

**Dollar General:**
- ✅ `DG - Domestic Vendor Guide.pdf` (2.6MB)

**Other Vendors (Phase 2):**
- Kroger: `kro_standard_vendor_agreement_merchandising.pdf`
- Walmart: `walmart - supply-chain-packaging-guide.pdf`
- Sam's Club: `sams-club-structural-packaging-standards.pdf`
- Target: `Target_Business-Partner-Code-of-Conduct.pdf`

---

## Architecture

```
Documents (PDF/DOCX)
    ↓
Text Extraction
    ↓
Intelligent Chunking (by section)
    ↓
Metadata Enrichment
    ↓
Generate Embeddings (OpenAI/Voyage/Cohere)
    ↓
Store in Vector Database (Qdrant/Pinecone)
    ↓
Index for Fast Search
    ↓
Ready for n8n Workflow Queries
```

---

## Phase 1: Document Collection

### Dollar General Documents Checklist

**Primary Documents:**
- [x] Vendor Compliance Guide / Domestic Vendor Guide
- [ ] Master Distribution Agreement (if separate)
- [ ] Deduction/Chargeback Policy Documentation
- [ ] Routing Guide / Logistics Requirements
- [ ] Quality Standards Guide
- [ ] Promotion/Allowance Terms
- [ ] Price Agreement / Price Files

**Current Available:**
- ✅ DG Domestic Vendor Guide (in `examples_vendor_guides/`)

**Additional Sources:**
- Vendor portal downloads
- Email attachments from Dollar General
- Contract management system
- Legal/procurement department files

---

## Phase 2: Environment Setup

### Install Python Dependencies

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install required packages
pip install \
    pypdf2 \
    pdfplumber \
    python-docx \
    langchain \
    langchain-text-splitters \
    openai \
    qdrant-client \
    python-dotenv

# Optional: For OCR (scanned PDFs)
pip install pytesseract pdf2image
```

### Setup Environment Variables

Create `.env` file:

```bash
# Embedding API
OPENAI_API_KEY=sk-...

# Vector Database
QDRANT_HOST=localhost
QDRANT_PORT=6333
# Or for cloud:
# QDRANT_URL=https://...
# QDRANT_API_KEY=...

# For Pinecone (alternative)
# PINECONE_API_KEY=...
# PINECONE_ENVIRONMENT=us-west1-gcp
```

---

## Phase 3: Vector Database Setup

### Option A: Qdrant (Recommended - Open Source)

#### Install via Docker

```bash
# Pull and run Qdrant
docker run -d \
  --name qdrant \
  -p 6333:6333 \
  -p 6334:6334 \
  -v $(pwd)/qdrant_storage:/qdrant/storage \
  qdrant/qdrant
```

#### Or Docker Compose

Create `docker-compose.yml`:

```yaml
version: '3.8'
services:
  qdrant:
    image: qdrant/qdrant
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - ./qdrant_storage:/qdrant/storage
    restart: unless-stopped
```

Run: `docker-compose up -d`

#### Verify Installation

```bash
curl http://localhost:6333/collections
```

Expected: `{"result": {"collections": []}, "status": "ok"}`

---

### Option B: Pinecone (Managed Cloud)

```python
import pinecone

# Initialize
pinecone.init(
    api_key="YOUR_API_KEY",
    environment="us-west1-gcp"
)

# Create index
pinecone.create_index(
    name="dollar-general-contracts",
    dimension=1536,  # For OpenAI text-embedding-3-small
    metric="cosine"
)
```

---

## Phase 4: Document Processing

### Step 1: Extract Text from PDFs

Create `extract_text.py`:

```python
import pdfplumber
from pathlib import Path

def extract_pdf_text(pdf_path):
    """Extract text from PDF with page tracking"""
    with pdfplumber.open(pdf_path) as pdf:
        pages_text = []
        for i, page in enumerate(pdf.pages):
            text = page.extract_text()
            pages_text.append({
                'page_number': i + 1,
                'text': text,
                'char_count': len(text)
            })

        full_text = '\n\n'.join([p['text'] for p in pages_text])

        return {
            'full_text': full_text,
            'pages': pages_text,
            'total_pages': len(pages_text),
            'metadata': {
                'filename': Path(pdf_path).name,
                'total_chars': len(full_text)
            }
        }

# Example usage
dg_guide = extract_pdf_text('gmail_trigger_download/examples_vendor_guides/DG - Domestic Vendor Guide.pdf')
print(f"Extracted {dg_guide['total_pages']} pages")
print(f"Total characters: {dg_guide['metadata']['total_chars']}")
```

### Step 2: Clean Extracted Text

```python
import re

def clean_text(text):
    """Clean and normalize extracted text"""
    # Remove excessive whitespace
    text = re.sub(r'\s+', ' ', text)

    # Remove page numbers (common patterns)
    text = re.sub(r'Page \d+ of \d+', '', text, flags=re.IGNORECASE)
    text = re.sub(r'^\d+$', '', text, flags=re.MULTILINE)

    # Remove headers/footers (adjust pattern for DG guide)
    text = re.sub(r'Dollar General.*?Confidential', '', text, flags=re.IGNORECASE)
    text = re.sub(r'Domestic Vendor Guide.*?\d{4}', '', text, flags=re.IGNORECASE)

    # Normalize quotes
    text = text.replace('"', '"').replace('"', '"')
    text = text.replace(''', "'").replace(''', "'")

    # Remove multiple newlines
    text = re.sub(r'\n{3,}', '\n\n', text)

    return text.strip()

# Apply cleaning
cleaned_text = clean_text(dg_guide['full_text'])
```

---

### Step 3: Intelligent Chunking (Section-Based)

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
import re

def chunk_by_sections(text, document_name):
    """Split by section headers for better context preservation"""

    # Detect section patterns (adjust for DG guide structure)
    section_patterns = [
        r'(SECTION \d+[\.:]\s*[A-Z][^\n]+)',
        r'(\d+\.\d+\s+[A-Z][^\n]+)',
        r'([A-Z\s]{10,}(?:\n|$))',  # All caps headers
    ]

    chunks = []
    current_section = None
    current_text = []

    lines = text.split('\n')

    for line in lines:
        # Check if line is a section header
        is_section = False
        for pattern in section_patterns:
            if re.match(pattern, line.strip()):
                # Save previous section
                if current_section and current_text:
                    chunks.append({
                        'section_header': current_section,
                        'text': '\n'.join(current_text),
                        'word_count': len(' '.join(current_text).split())
                    })

                # Start new section
                current_section = line.strip()
                current_text = []
                is_section = True
                break

        if not is_section:
            current_text.append(line)

    # Save last section
    if current_section and current_text:
        chunks.append({
            'section_header': current_section,
            'text': '\n'.join(current_text),
            'word_count': len(' '.join(current_text).split())
        })

    # Further split large sections
    final_chunks = []
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=1000,
        chunk_overlap=200,
        separators=["\n\n", "\n", ". ", " "]
    )

    for chunk in chunks:
        if chunk['word_count'] > 400:  # ~2000 chars
            # Split large sections
            sub_chunks = splitter.split_text(chunk['text'])
            for i, sub in enumerate(sub_chunks):
                final_chunks.append({
                    'section_header': chunk['section_header'],
                    'text': chunk['section_header'] + '\n\n' + sub,
                    'is_continuation': i > 0
                })
        else:
            final_chunks.append({
                'section_header': chunk['section_header'],
                'text': chunk['section_header'] + '\n\n' + chunk['text'],
                'is_continuation': False
            })

    return final_chunks

# Chunk the document
chunks = chunk_by_sections(cleaned_text, 'DG Domestic Vendor Guide')
print(f"Created {len(chunks)} chunks")
```

---

### Step 4: Metadata Enrichment

```python
def detect_topics(text):
    """Detect topics based on keywords"""
    topics = []

    topic_keywords = {
        'chargebacks': ['chargeback', 'deduction', 'penalty', 'fee'],
        'shortage': ['shortage', 'short ship', 'short shipment', 'missing', 'quantity variance'],
        'damage': ['damage', 'damaged', 'defect', 'defective', 'broken', 'quality'],
        'routing': ['routing', 'distribution center', 'DC', 'incorrect delivery'],
        'compliance': ['compliance', 'violation', 'requirement', 'must comply'],
        'upc': ['UPC', 'barcode', 'labeling', 'label requirement'],
        'packaging': ['packaging', 'carton', 'case pack', 'unit pack'],
        'payment_terms': ['payment', 'net 30', 'discount', 'due date', 'invoice'],
        'shipping': ['freight', 'shipping', 'delivery', 'transportation', 'carrier'],
        'rework': ['rework', 'repack', 're-pack', 're-handle'],
    }

    text_lower = text.lower()
    for topic, keywords in topic_keywords.items():
        if any(kw.lower() in text_lower for kw in keywords):
            topics.append(topic)

    return topics

def extract_chargeback_codes(text):
    """Extract Dollar General chargeback codes"""
    # Pattern: Letter-Number format (D-001, Q-002, U-003, etc.)
    codes = re.findall(r'\b([A-Z]{1,2}-\d{3})\b', text)
    return list(set(codes))

def enrich_chunks_with_metadata(chunks, doc_info):
    """Add comprehensive metadata to each chunk"""

    enriched = []

    for i, chunk in enumerate(chunks):
        # Extract section number if present
        section_match = re.search(r'(?:SECTION\s+)?(\d+\.?\d*)', chunk['section_header'])
        section_num = section_match.group(1) if section_match else None

        # Detect topics
        topics = detect_topics(chunk['text'])

        # Extract chargeback codes
        codes = extract_chargeback_codes(chunk['text'])

        # Extract dollar amounts
        amounts = re.findall(r'\$[\d,]+\.?\d{0,2}', chunk['text'])

        enriched.append({
            'text': chunk['text'],
            'metadata': {
                # Document info
                'document_name': doc_info['name'],
                'document_type': doc_info['type'],  # 'vendor_guide', 'agreement', etc.
                'vendor': 'Dollar General',
                'vendor_id': 'DOLLAR_GENERAL',

                # Section info
                'section_number': section_num,
                'section_title': chunk['section_header'],
                'chunk_index': i,
                'is_continuation': chunk.get('is_continuation', False),

                # Content analysis
                'topics': topics,
                'chargeback_codes': codes,
                'contains_amounts': len(amounts) > 0,

                # Document metadata
                'effective_date': doc_info.get('effective_date', '2025-01-01'),
                'version': doc_info.get('version', '2025'),
                'page_estimate': (i // 3) + 1,  # Rough page estimate

                # For filtering
                'char_count': len(chunk['text']),
                'word_count': len(chunk['text'].split())
            }
        })

    return enriched

# Enrich chunks
doc_info = {
    'name': 'DG Domestic Vendor Guide',
    'type': 'vendor_guide',
    'effective_date': '2025-01-01',
    'version': '2025'
}

enriched_chunks = enrich_chunks_with_metadata(chunks, doc_info)
print(f"Enriched {len(enriched_chunks)} chunks with metadata")
```

---

## Phase 5: Generate Embeddings

### Choose Embedding Model

| Model | Dimensions | Cost (per 1M tokens) | Best For |
|-------|-----------|---------------------|----------|
| OpenAI text-embedding-3-small | 1536 | $0.02 | Cost-effective, good quality |
| OpenAI text-embedding-3-large | 3072 | $0.13 | Highest accuracy |
| Voyage voyage-2 | 1024 | $0.12 | Optimized for RAG |
| Cohere embed-english-v3.0 | 1024 | $0.10 | Good balance |

**Recommendation:** OpenAI `text-embedding-3-small` for MVP

### Generate Embeddings

```python
import openai
from dotenv import load_dotenv
import os

load_dotenv()
openai.api_key = os.getenv('OPENAI_API_KEY')

def generate_embeddings(texts, model="text-embedding-3-small"):
    """Generate embeddings for text chunks"""

    # OpenAI allows up to 2048 texts per request
    batch_size = 100
    all_embeddings = []

    for i in range(0, len(texts), batch_size):
        batch = texts[i:i+batch_size]

        print(f"Processing batch {i//batch_size + 1}/{(len(texts)-1)//batch_size + 1}")

        response = openai.embeddings.create(
            model=model,
            input=batch
        )

        embeddings = [item.embedding for item in response.data]
        all_embeddings.extend(embeddings)

    return all_embeddings

# Extract just the text
texts = [chunk['text'] for chunk in enriched_chunks]

# Generate embeddings
print("Generating embeddings...")
embeddings = generate_embeddings(texts)
print(f"Generated {len(embeddings)} embeddings")

# Combine with chunks
for i, chunk in enumerate(enriched_chunks):
    chunk['embedding'] = embeddings[i]
```

**Estimated Cost:**
- ~100 chunks × ~500 tokens/chunk = 50,000 tokens
- Cost: ~$0.001 (less than 1 cent)

---

## Phase 6: Load into Vector Database

### Qdrant Implementation

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct

# Connect to Qdrant
client = QdrantClient(host="localhost", port=6333)
# Or cloud: client = QdrantClient(url="https://...", api_key="...")

# Create collection
collection_name = "dollar_general_contracts"

client.create_collection(
    collection_name=collection_name,
    vectors_config=VectorParams(
        size=1536,  # Match embedding dimension
        distance=Distance.COSINE
    )
)

# Prepare points
points = []
for i, chunk in enumerate(enriched_chunks):
    points.append(
        PointStruct(
            id=i,
            vector=chunk['embedding'],
            payload={
                'text': chunk['text'],
                **chunk['metadata']
            }
        )
    )

# Upload in batches
batch_size = 100
for i in range(0, len(points), batch_size):
    batch = points[i:i+batch_size]
    client.upsert(
        collection_name=collection_name,
        points=batch
    )
    print(f"Uploaded batch {i//batch_size + 1}")

print(f"✅ Uploaded {len(points)} vectors to Qdrant")
```

---

### Pinecone Implementation (Alternative)

```python
import pinecone

# Initialize
pinecone.init(api_key="YOUR_API_KEY", environment="us-west1-gcp")

# Get index
index = pinecone.Index("dollar-general-contracts")

# Prepare vectors
vectors = []
for i, chunk in enumerate(enriched_chunks):
    vectors.append({
        "id": f"dg_{i}",
        "values": chunk['embedding'],
        "metadata": {
            "text": chunk['text'][:40000],  # Pinecone 40KB limit
            **{k: v for k, v in chunk['metadata'].items()
               if not isinstance(v, list) or len(str(v)) < 1000}
        }
    })

# Upload in batches
batch_size = 100
for i in range(0, len(vectors), batch_size):
    batch = vectors[i:i+batch_size]
    index.upsert(vectors=batch)
    print(f"Uploaded batch {i//batch_size + 1}")
```

---

## Phase 7: Create Indexes for Fast Filtering

### Qdrant Payload Indexes

```python
from qdrant_client.models import PayloadSchemaType

# Index frequently filtered fields
client.create_payload_index(
    collection_name=collection_name,
    field_name="vendor_id",
    field_schema=PayloadSchemaType.KEYWORD
)

client.create_payload_index(
    collection_name=collection_name,
    field_name="topics",
    field_schema=PayloadSchemaType.KEYWORD
)

client.create_payload_index(
    collection_name=collection_name,
    field_name="chargeback_codes",
    field_schema=PayloadSchemaType.KEYWORD
)

client.create_payload_index(
    collection_name=collection_name,
    field_name="document_type",
    field_schema=PayloadSchemaType.KEYWORD
)

print("✅ Created indexes for fast filtering")
```

---

## Phase 8: Test Search Quality

### Test Queries

```python
def test_search(query_text, top_k=5):
    """Test semantic search"""

    # Generate query embedding
    response = openai.embeddings.create(
        model="text-embedding-3-small",
        input=[query_text]
    )
    query_embedding = response.data[0].embedding

    # Search
    results = client.search(
        collection_name=collection_name,
        query_vector=query_embedding,
        query_filter={
            "must": [
                {"key": "vendor_id", "match": {"value": "DOLLAR_GENERAL"}}
            ]
        },
        limit=top_k,
        score_threshold=0.7
    )

    # Display results
    print(f"\n{'='*80}")
    print(f"Query: {query_text}")
    print(f"{'='*80}\n")

    for i, result in enumerate(results):
        print(f"Result {i+1} (Score: {result.score:.3f})")
        print(f"Section: {result.payload.get('section_title', 'N/A')}")
        print(f"Topics: {', '.join(result.payload.get('topics', []))}")
        print(f"Text: {result.payload['text'][:300]}...")
        print(f"{'-'*80}\n")

    return results

# Test with Dollar General deduction scenarios
test_queries = [
    "What are the requirements for shortage claims and deductions?",
    "Incorrect DC violation chargeback policy",
    "Missing UPC code compliance requirements",
    "Rework and rehandling charges rates",
    "Damaged product disposal deduction process",
    "Vendor performance chargeback dispute process"
]

for query in test_queries:
    test_search(query, top_k=3)
```

**Expected Good Results:**
- Score > 0.8: Highly relevant
- Score 0.7-0.8: Relevant
- Score < 0.7: May not be relevant

---

## Phase 9: Integration with n8n Workflow

### Query Vector Store from n8n

**Step 1: Generate Query Embedding**

n8n HTTP Request Node:
```json
{
  "method": "POST",
  "url": "https://api.openai.com/v1/embeddings",
  "headers": {
    "Authorization": "Bearer {{$credentials.openai.apiKey}}",
    "Content-Type": "application/json"
  },
  "body": {
    "model": "text-embedding-3-small",
    "input": "{{$json.query_text}}"
  }
}
```

**Step 2: Search Qdrant**

n8n HTTP Request Node:
```json
{
  "method": "POST",
  "url": "http://localhost:6333/collections/dollar_general_contracts/points/search",
  "headers": {
    "Content-Type": "application/json"
  },
  "body": {
    "vector": "{{$json.embedding}}",
    "filter": {
      "must": [
        {"key": "vendor_id", "match": {"value": "DOLLAR_GENERAL"}}
      ]
    },
    "limit": 10,
    "score_threshold": 0.7,
    "with_payload": true
  }
}
```

**Step 3: Parse Results**

n8n Code Node:
```javascript
const results = $input.all()[0].json.result;

const clauses = results.map(r => ({
  text: r.payload.text,
  section: r.payload.section_title,
  document: r.payload.document_name,
  topics: r.payload.topics,
  score: r.score
}));

return { contract_clauses: clauses };
```

---

## Phase 10: Maintenance & Updates

### When to Update Vector Store

- ✅ New vendor guide version released
- ✅ Contract amendments signed
- ✅ Policy changes announced
- ✅ Quarterly review (minimum)

### Update Process

```python
def update_document(new_pdf_path, document_info):
    """Add new document version to vector store"""

    # 1. Extract and process new document
    extracted = extract_pdf_text(new_pdf_path)
    cleaned = clean_text(extracted['full_text'])
    chunks = chunk_by_sections(cleaned, document_info['name'])
    enriched = enrich_chunks_with_metadata(chunks, document_info)

    # 2. Generate embeddings
    texts = [c['text'] for c in enriched]
    embeddings = generate_embeddings(texts)
    for i, chunk in enumerate(enriched):
        chunk['embedding'] = embeddings[i]

    # 3. Get current max ID
    collection_info = client.get_collection(collection_name)
    current_count = collection_info.points_count

    # 4. Upload with new IDs
    points = []
    for i, chunk in enumerate(enriched):
        points.append(
            PointStruct(
                id=current_count + i,
                vector=chunk['embedding'],
                payload={'text': chunk['text'], **chunk['metadata']}
            )
        )

    client.upsert(collection_name=collection_name, points=points)

    print(f"✅ Added {len(points)} new chunks")

    # 5. Optionally delete old version
    # client.delete(
    #     collection_name=collection_name,
    #     points_selector={
    #         "filter": {
    #             "must": [
    #                 {"key": "document_name", "match": {"value": "Old Doc Name"}},
    #                 {"key": "version", "match": {"value": "2024"}}
    #             ]
    #         }
    #     }
    # )
```

---

## Complete Setup Script

### `setup_rag.py` - All-in-One

```python
#!/usr/bin/env python3
"""
Complete RAG setup script for Dollar General vendor guide
"""

import os
from pathlib import Path
from dotenv import load_dotenv
import pdfplumber
import re
import openai
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct, PayloadSchemaType
from langchain.text_splitter import RecursiveCharacterTextSplitter

# Load environment
load_dotenv()
openai.api_key = os.getenv('OPENAI_API_KEY')

# Configuration
VENDOR_GUIDE_PATH = 'gmail_trigger_download/examples_vendor_guides/DG - Domestic Vendor Guide.pdf'
COLLECTION_NAME = 'dollar_general_contracts'
QDRANT_HOST = os.getenv('QDRANT_HOST', 'localhost')
QDRANT_PORT = int(os.getenv('QDRANT_PORT', 6333))

def main():
    print("🚀 Starting RAG Setup for Dollar General")
    print("="*80)

    # Step 1: Extract text
    print("\n📄 Step 1: Extracting text from PDF...")
    with pdfplumber.open(VENDOR_GUIDE_PATH) as pdf:
        full_text = '\n\n'.join([page.extract_text() for page in pdf.pages])
        print(f"✅ Extracted {len(pdf.pages)} pages")

    # Step 2: Clean text
    print("\n🧹 Step 2: Cleaning text...")
    cleaned_text = clean_text(full_text)
    print(f"✅ Cleaned {len(cleaned_text)} characters")

    # Step 3: Chunk by sections
    print("\n✂️  Step 3: Chunking by sections...")
    chunks = chunk_by_sections(cleaned_text, 'DG Domestic Vendor Guide')
    print(f"✅ Created {len(chunks)} chunks")

    # Step 4: Enrich with metadata
    print("\n📊 Step 4: Enriching with metadata...")
    doc_info = {
        'name': 'DG Domestic Vendor Guide',
        'type': 'vendor_guide',
        'effective_date': '2025-01-01',
        'version': '2025'
    }
    enriched = enrich_chunks_with_metadata(chunks, doc_info)
    print(f"✅ Enriched {len(enriched)} chunks")

    # Step 5: Generate embeddings
    print("\n🔢 Step 5: Generating embeddings...")
    texts = [c['text'] for c in enriched]
    embeddings = generate_embeddings(texts)
    for i, chunk in enumerate(enriched):
        chunk['embedding'] = embeddings[i]
    print(f"✅ Generated {len(embeddings)} embeddings")

    # Step 6: Setup Qdrant
    print("\n🗄️  Step 6: Setting up Qdrant...")
    client = QdrantClient(host=QDRANT_HOST, port=QDRANT_PORT)

    # Recreate collection (delete if exists)
    try:
        client.delete_collection(COLLECTION_NAME)
        print(f"   Deleted existing collection")
    except:
        pass

    client.create_collection(
        collection_name=COLLECTION_NAME,
        vectors_config=VectorParams(size=1536, distance=Distance.COSINE)
    )
    print(f"✅ Created collection '{COLLECTION_NAME}'")

    # Step 7: Upload vectors
    print("\n⬆️  Step 7: Uploading vectors...")
    points = []
    for i, chunk in enumerate(enriched):
        points.append(
            PointStruct(
                id=i,
                vector=chunk['embedding'],
                payload={'text': chunk['text'], **chunk['metadata']}
            )
        )

    batch_size = 100
    for i in range(0, len(points), batch_size):
        batch = points[i:i+batch_size]
        client.upsert(collection_name=COLLECTION_NAME, points=batch)
        print(f"   Uploaded batch {i//batch_size + 1}/{(len(points)-1)//batch_size + 1}")

    print(f"✅ Uploaded {len(points)} vectors")

    # Step 8: Create indexes
    print("\n🔍 Step 8: Creating indexes...")
    client.create_payload_index(COLLECTION_NAME, "vendor_id", PayloadSchemaType.KEYWORD)
    client.create_payload_index(COLLECTION_NAME, "topics", PayloadSchemaType.KEYWORD)
    client.create_payload_index(COLLECTION_NAME, "chargeback_codes", PayloadSchemaType.KEYWORD)
    print("✅ Created indexes")

    # Step 9: Test search
    print("\n🧪 Step 9: Testing search...")
    test_query = "What are the chargeback policies for shortage violations?"
    results = test_search(client, test_query, top_k=3)
    print(f"✅ Search test complete - found {len(results)} results")

    print("\n" + "="*80)
    print("🎉 RAG Setup Complete!")
    print(f"   Collection: {COLLECTION_NAME}")
    print(f"   Vectors: {len(points)}")
    print(f"   Ready for n8n integration")
    print("="*80)

# Include all helper functions here (clean_text, chunk_by_sections, etc.)

if __name__ == '__main__':
    main()
```

Run: `python setup_rag.py`

---

## Validation Checklist

- [ ] Dollar General Vendor Guide processed
- [ ] Text extraction quality verified (no gibberish)
- [ ] Chunks are coherent and meaningful
- [ ] Metadata correctly assigned (topics, codes, sections)
- [ ] Embeddings generated successfully
- [ ] Vector store contains expected number of records
- [ ] Search returns relevant results for test queries
- [ ] Filtering by vendor/topic works correctly
- [ ] n8n integration tested end-to-end
- [ ] Performance acceptable (<2 seconds for queries)

---

## Quality Metrics

### Coverage
- ✅ All major deduction types covered in vector store
- ✅ Chargeback policies indexed
- ✅ Compliance requirements searchable
- ✅ Dispute processes documented

### Precision & Recall
- **Precision:** Top 10 results relevant >80% of time
- **Recall:** Relevant sections found in top 10 results >90% of time
- **Response Time:** <2 seconds per query

---

## Troubleshooting

### Issue: Low search scores (<0.7)

**Solution:**
- Verify embeddings model is consistent (same model for docs and queries)
- Check if query text needs more context
- Adjust chunk size (try 800-1200 characters)

### Issue: Irrelevant results returned

**Solution:**
- Improve metadata tagging (topics, keywords)
- Use tighter filters (specific topics, chargeback codes)
- Increase score threshold (0.75-0.8)

### Issue: Missing relevant sections

**Solution:**
- Check if section was filtered out
- Verify section text was extracted correctly
- Try broader query phrasing
- Increase top_k (retrieve more results)

---

## Cost Estimates

### One-Time Setup
- **Document Processing:** Free (Python libraries)
- **Embeddings:** ~$0.001-0.01 per document
- **Vector DB Storage:** Free (Qdrant self-hosted) or $0/month for <1GB

### Ongoing Costs
- **Qdrant Self-Hosted:** Free (Docker)
- **Qdrant Cloud:** ~$25/month (optional)
- **Pinecone:** ~$70/month (if used instead)
- **Embedding Updates:** <$1/month (quarterly updates)
- **Query Embeddings:** ~$0.0001 per query

**Total Estimated Monthly Cost:** $0-25 (self-hosted) or $70-100 (cloud)

---

## Next Steps

1. ✅ Run complete setup script
2. Verify data quality with test queries
3. Integrate with n8n workflow (see WORKFLOW_PLAN.md)
4. Monitor search quality in production
5. Plan Phase 2: Add Kroger, Walmart, Sam's Club guides

---

## Phase 2 Expansion

### Additional Vendors

**Kroger:**
- `kro_standard_vendor_agreement_merchandising.pdf`
- Create collection: `kroger_contracts`

**Walmart:**
- `walmart - supply-chain-packaging-guide.pdf`
- `supplier-policies-guidelines-022425.pdf`
- Create collection: `walmart_contracts`

**Sam's Club:**
- `sams-club-structural-packaging-standards.pdf`
- Create collection: `sams_club_contracts`

**Multi-Vendor Search:**
- Query across collections
- Filter by vendor_id
- Unified response format

---

**Document Version:** 1.0
**Last Updated:** 2026-01-06
**Status:** Ready for Implementation
