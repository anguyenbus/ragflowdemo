# Document Upload Mid-Chat Session

**Document Version:** 1.0
**Date:** 2026-05-06
**Purpose:** Mermaid diagrams for document upload during active chat session

---

## Table of Contents

1. [Session State Machine](#1-session-state-machine)
2. [Document Upload During Session (Sequence)](#2-document-upload-during-session)
3. [Document Upload Lifecycle](#3-document-upload-lifecycle)
4. [API Endpoints](#4-api-endpoints)

---

## 1. Session State Machine

Shows session states including document upload during active session:

```mermaid
stateDiagram-v2
    [*] --> Creating: User initiates session
    Creating --> Active: Resources allocated
    Active --> Uploading: User uploads documents
    Uploading --> Processing: Upload complete
    Processing --> Ready: Ingestion complete
    Processing --> Error: Ingestion failed
    Ready --> Querying: User submits query
    Querying --> Ready: Query complete
    Ready --> Uploading: User uploads more docs
    Ready --> Expiring: TTL timeout
    Active --> Expiring: TTL timeout
    Error --> Expiring: TTL timeout
    Expiring --> Deleting: Cleanup triggered
    Deleting --> Deleted: All resources removed
    Deleting --> [*]
    Active --> Deleting: Manual delete
    Ready --> Deleting: Manual delete

    note right of Creating
        Duration: ~500ms
        Creates:
        - OpenSearch index
        - S3 prefix
        - DynamoDB metadata
    end note

    note right of Ready
        User can now:
        - Query documents
        - Upload more documents
        - Summarize
        - Extract facts
    end note

    note right of Expiring
        TTL reached
        Cleanup scheduled
        within 5 minutes
    end note

    note right of Deleting
        Irreversible
        All data deleted:
        - OpenSearch index
        - S3 documents
        - DynamoDB metadata
    end note
```

---

## 2. Document Upload During Session

Sequence diagram for uploading documents to an active chat session:

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant WebApp as Web Application
    participant APIGW as API Gateway
    participant S3 as Amazon S3
    participant SQS as SQS Ingestion Queue
    participant Lambda as Ingestion Lambda
    participant Parser as Document Parser
    participant Chunking as Chunking Service
    participant Bedrock as Bedrock Embeddings
    participant OpenSearch as OpenSearch
    participant DynamoDB as DynamoDB
    participant SNS as SNS Notifications
    participant WebSocket as WebSocket Service

    User->>WebApp: Select documents to upload (mid-chat)
    WebApp->>APIGW: POST /sessions/{id}/documents
    APIGW-->>WebApp: Presigned S3 URL

    User->>S3: Upload file via presigned URL
    S3-->>WebApp: Upload complete

    S3->>SQS: s3:ObjectCreated:* event
    SQS->>Lambda: Trigger ingestion

    Lambda->>Parser: Parse document (MD/PDF/DOCX)
    Note over Parser: Extract text<br/>Calculate page numbers<br/>Extract metadata

    Parser-->>Lambda: Parsed text + metadata

    Lambda->>DynamoDB: UPDATE document status: PROCESSING
    Lambda->>Chunking: Chunk document (2000/1500/1000 tokens)
    Note over Chunking: Three chunk types:<br/>- Summary (2000t)<br/>- Detail (1500t)<br/>- Fact (1000t)

    Chunking-->>Lambda: Array of chunks

    loop For each chunk
        Lambda->>Bedrock: Generate embedding (Amazon Titan)
        Bedrock-->>Lambda: 1536-dimensional vector
        Lambda->>OpenSearch: Index chunk with vector
        Note over OpenSearch: Index: vector_store_session_{id}<br/>k-NN vector search enabled
    end

    Lambda->>DynamoDB: UPDATE document status: READY
    Lambda->>SNS: Publish document_ready event

    SNS->>WebSocket: Broadcast status update
    WebSocket-->>WebApp: WebSocket message: document.status_update
    WebApp-->>User: Show "Document ready for querying"

    Note over User,WebSocket: Total time: 5-30 seconds<br/>Depends on document size<br/>User can continue chatting
```

---

## 3. Document Upload Lifecycle

State diagram for individual document upload process:

```mermaid
stateDiagram-v2
    [*] --> Uploading: User selects file

    Uploading --> Uploaded: File stored in S3
    Uploading --> UploadFailed: Validation error

    Uploaded --> Processing: SQS triggers Lambda
    Uploaded --> UploadFailed: Queue timeout

    Processing --> Parsing: Document type detected
    Processing --> ProcessingFailed: Lambda error

    Parsing --> Chunking: Content extracted
    Parsing --> ProcessingFailed: Parse error

    Chunking --> Embedding: Chunks created
    Chunking --> ProcessingFailed: Chunking error

    Embedding --> Indexing: Embeddings generated
    Embedding --> ProcessingFailed: Bedrock error

    Indexing --> Indexed: OpenSearch write successful
    Indexing --> ProcessingFailed: Index write error

    Indexed --> [*]: Complete - Ready for queries

    UploadFailed --> [*]: Notify user
    ProcessingFailed --> [*]: Log error + notify

    note right of Uploading
        User initiates upload
        File validation (size, type)
    end note

    note right of Uploaded
        File in S3
        Metadata in DynamoDB
        Job queued
    end note

    note right of Processing
        Lambda execution
        Parse (MD/PDF/DOCX)
        Chunking
        Embedding generation
    end note

    note right of Indexed
        Searchable
        User notified via WebSocket
        Ready for queries in session
    end note
```

---

## 4. API Endpoints

### 4.1 Upload Document to Session

**POST** `/api/v1/sessions/{session_id}/documents`

**Request:**
```json
{
    "files": [binary files],
    "metadata": {
        "tags": ["policy", "hr"],
        "description": "Employee handbook 2025"
    }
}
```

**Response:** `202 Accepted`
```json
{
    "upload_id": "upload_abc123",
    "documents": [
        {
            "document_id": "doc_xyz789",
            "filename": "handbook.md",
            "status": "uploaded",
            "estimated_indexing_time": 180
        }
    ]
}
```

### 4.2 Get Document Status

**GET** `/api/v1/sessions/{session_id}/documents/{document_id}`

**Response:** `200 OK`
```json
{
    "document_id": "doc_xyz789",
    "filename": "handbook.md",
    "status": "ready",
    "chunk_count": 42,
    "indexed_at": "2026-05-06T10:30:00Z"
}
```

**Status Values:**
| Status | Description |
|--------|-------------|
| `uploading` | File being uploaded to S3 |
| `uploaded` | File in S3, queued for processing |
| `processing` | Being parsed and chunked |
| `indexed` | Embeddings generated, in OpenSearch |
| `ready` | Ready for queries |
| `error` | Processing failed |

---

## Supported Document Types

| Type | Extensions | Parser |
|------|------------|--------|
| Markdown | `.md` | Direct read |
| PDF | `.pdf` | PyPDF / Textract |
| Word | `.docx` | python-docx |
| Text | `.txt` | Direct read |
| Excel | `.xlsx` | openpyxl |

---

## Key Features

1. **Non-blocking upload**: User can continue chatting while documents process
2. **WebSocket updates**: Real-time status updates to frontend
3. **Session-scoped**: Documents isolated to specific session
4. **Multi-format support**: MD, PDF, DOCX, TXT, XLSX
5. **Status tracking**: Full lifecycle state per document
6. **Error handling**: Graceful failure with user notification

---

**END OF DOCUMENT**
