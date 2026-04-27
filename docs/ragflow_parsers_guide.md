# RAGFlow Parser Types Guide

RAGFlow provides multiple built-in parsers for different document types. Each parser is optimized for specific document structures and content types.

## Overview

| Parser | Document Type | Key Features |
|--------|--------------|--------------|
| **General** | General documents | OCR, layout analysis, multi-format support |
| **Table** | Excel/CSV | Table preservation, column extraction |
| **Laws** | Legal documents | Hierarchical structure, article numbers |
| **Paper** | Research papers | Academic sections, citations, tables |
| **Book** | Books | Chapter detection, hierarchical merging |
| **Presentation** | PPT/PDF slides | Slide-by-slide extraction |
| **Manual** | Technical manuals | Procedure extraction, step-by-step |
| **QA** | Q&A pairs | Question-answer format |
| **Resume** | Resumes/CVs | Structured field extraction |
| **Picture** | Images/Videos | Vision-based description |
| **One** | Single chunk | Entire document as one chunk |
| **Audio** | Audio files | Speech-to-text transcription |
| **Email** | Email files | Header, body, attachment parsing |
| **Tag** | Tagged content | Content-tag association |

---

## 1. General (Naive)

**Best for:** General-purpose documents, PDFs, Word, HTML, Markdown, TXT

**Features:**
- OCR for scanned documents
- Layout analysis (DeepDOC, Naive, Docling, MinerU options)
- Table extraction and preservation
- Image processing with vision models
- Multi-language support
- Preserves document structure with sections

**Processing Steps:**
1. OCR (if needed)
2. Layout recognition
3. Table analysis
4. Text merging
5. Chunking based on token limits

**File formats:** PDF, DOCX, HTML, MD, TXT, EPUB, JSON, CSV

---

## 2. Table

**Best for:** Excel files (.xlsx, .xls), CSV files with tabular data

**Features:**
- Excel sheet-by-sheet processing
- Header detection and preservation
- Table structure maintained
- Image extraction from cells (with vision descriptions)
- Row-based chunking for large tables

**Processing:**
- Extracts tables from each sheet
- Preserves column headers
- Handles cell images with descriptions
- Supports up to 3000 rows per chunk

---

## 3. Laws (Legal Documents Parser)

**Best for:** Legal documents, contracts, regulations, laws, statutes

### Supported File Formats
- `.docx` (Word documents)
- `.pdf` (PDF documents)
- `.txt` (Plain text)
- `.md` / `.markdown` (Markdown)
- `.html` / `.htm` (HTML documents)
- `.doc` (Legacy Word, via Apache Tika)

---

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         LAWS PARSER PIPELINE                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐    ┌──────────────┐    ┌─────────────────────┐     │
│  │  DOCUMENT   │───▶│  FORMAT      │───▶│  PARSER SELECTION   │     │
│  │  INPUT      │    │  DETECTION   │    │  (Docx/Pdf/Text)    │     │
│  └─────────────┘    └──────────────┘    └─────────────────────┘     │
│                                                           │         │
│                                    ┌──────────────────────┘
│                                    ▼
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    CONTENT EXTRACTION                         │  │
│  │  • DOCX: Paragraph extraction with style analysis             │  │
│  │  • PDF:  OCR → Layout → Vertical merge                        │  │
│  │  • Text: Line-by-line reading                                 │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                    │                                │
│                                    ▼                                │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    PREPROCESSING                              │  │
│  │  1. Remove table of contents (Contents)                       │
│  │  2. Convert colons to titles                                  │  │
│  │  3. Clean whitespace and formatting                           │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                    │                                │
│                                    ▼                                │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                 BULLET CATEGORIZATION                         │  │
│  │  Detect heading patterns and assign levels                    │  │
│  │  • Pattern 1: Numeric (1. 1.1 1.1.1)                          │  │
│  │  • Pattern 2: English legal (Chapter/Article)                 │  │
│  │  • Pattern 3: Markdown (# ## ###)                             │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                    │                                │
│                                    ▼                                │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                      TREE MERGE                               │  │
│  │  Build hierarchical tree and extract chunks                   │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                    │                                │
│                                    ▼                                │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    OUTPUT CHUNKS                              │  │
│  │  Hierarchical chunks with preserved legal structure           │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Key Algorithm: Tree Merge

The core innovation of the Laws parser is the `tree_merge()` function which builds a hierarchical structure from flat document content.

```python
# File: /ragflow/rag/nlp/__init__.py

def tree_merge(bull, sections, depth):
    """
    Build hierarchical tree from sections using bullet pattern detection.
    
    Args:
        bull: Bullet pattern index (0-4) from bullets_category()
        sections: List of (text, layout_info) tuples
        depth: Target hierarchy depth for chunking
    
    Returns:
        List of hierarchical chunks preserving legal structure
    """
    if not sections or bull < 0:
        return sections
    
    # Assign levels to each section based on bullet patterns
    def get_level(bull, section):
        text, layout = section
        for i, title in enumerate(BULLET_PATTERN[bull]):
            if re.match(title, text.strip()):
                return i + 1, text
        return len(BULLET_PATTERN[bull]) + 2, text
    
    # Build level-indexed lines
    lines = []
    level_set = set()
    for section in sections:
        level, text = get_level(bull, section)
        lines.append((level, text))
        level_set.add(level)
    
    # Determine target level for hierarchy
    sorted_levels = sorted(level_set)
    target_level = sorted_levels[depth - 1] if depth <= len(sorted_levels) else sorted_levels[-1]
    
    # Build tree and extract chunks
    root = Node(level=0, depth=target_level, texts=[])
    root.build_tree(lines)
    
    return [element for element in root.get_tree() if element]
```

---

### Bullet Pattern Detection

The parser uses regex patterns to detect legal document headings:

```python
# File: /ragflow/rag/nlp/__init__.py

BULLET_PATTERN = [
    # Pattern 0: Numeric Hierarchy
    [
        r"[0-9]{,2}[\. ]",                     # 1. 2. 3.
        r"[0-9]{,2}\.[0-9]{,2}[^a-zA-Z/%~-]",  # 1.1 2.3
        r"[0-9]{,2}\.[0-9]{,2}\.[0-9]{,2}",   # 1.1.1
        r"[0-9]{,2}\.[0-9]{,2}\.[0-9]{,2}\.[0-9]{,2}", # 1.1.1.1
    ],
    # Pattern 1: English Legal
    [
        r"PART (ONE|TWO|THREE|FOUR|FIVE|SIX|SEVEN|EIGHT|NINE|TEN)",
        r"Chapter (I+V?|VI*|XI|IX|X)",
        r"Section [0-9]+",
        r"Article [0-9]+"
    ],
    # Pattern 2: Markdown
    [
        r"^#[^#]",     # #
        r"^##[^#]",    # ##
        r"^###.*",     # ###
        r"^####.*",    # ####
        r"^#####.*",   # #####
        r"^######.*",  # ######
    ],
    # Pattern 3-5: Additional patterns for various formats
]
```

---

### Node Tree Structure

The `Node` class builds a tree representation of the document hierarchy:

```python
# File: /ragflow/rag/nlp/__init__.py

class Node:
    def __init__(self, level, depth=-1, texts=None):
        self.level = level      # Hierarchy level (0=root, 1=chapter, etc.)
        self.depth = depth      # Target depth for chunking
        self.texts = texts or []  # Content at this node
        self.children = []      # Child nodes
    
    def build_tree(self, lines):
        """Build tree from (level, text) tuples."""
        stack = [self]
        for level, text in lines:
            # Beyond target depth: merge into current leaf
            if self.depth != -1 and level > self.depth:
                stack[-1].add_text(text)
                continue
            
            # Find proper parent (level must be strictly smaller)
            while len(stack) > 1 and level <= stack[-1].get_level():
                stack.pop()
            
            # Create new node and descend
            node = Node(level=level, texts=[text])
            stack[-1].add_child(node)
            stack.append(node)
    
    def get_tree(self):
        """Extract chunks via DFS traversal."""
        tree_list = []
        self._dfs(self, tree_list, [])
        return tree_list
```

---

### Tree Structure Example

```
Input Document:
─────────────────────────────────────────────────────────────────────
CHAPTER I: GENERAL PROVISIONS
Article 1 This Law is enacted for...
  1.1 Scope of Application
      This Law applies to...

CHAPTER II: CONCLUSION OF CONTRACTS
Article 2 The parties may conclude a contract...
  Article 3 An offer is...
      An offer is a proposal...

Tree Representation:
─────────────────────────────────────────────────────────────────────
                    Node(level=0, root)
                           │
            ┌──────────────┴──────────────┐
            │                             │
    Node(level=1)                   Node(level=1)
    "CHAPTER I: GENERAL"            "CHAPTER II: CONCLUSION"
            │                             │
    ┌───────┴────────┐           ┌───────┴────────┐
    │                │           │                │
Node(2)        Node(2)       Node(2)        Node(2)
"Art 1"      "1.1 Scope"    "Art 2"        "Art 3"

Output Chunks (depth=2):
─────────────────────────────────────────────────────────────────────
Chunk 1: "CHAPTER I: GENERAL PROVISIONS\nArticle 1 This Law is enacted..."
Chunk 2: "CHAPTER I: GENERAL PROVISIONS\n1.1 Scope of Application\nThis Law applies..."
Chunk 3: "CHAPTER II: CONCLUSION OF CONTRACTS\nArticle 2 The parties may..."
Chunk 4: "CHAPTER II: CONCLUSION OF CONTRACTS\nArticle 3 An offer is...\nAn offer is a..."
```

---

### DOCX Processing Flow

For Word documents, the parser uses style-based hierarchy detection:

```python
# File: /ragflow/rag/app/laws.py

class Docx(DocxParser):
    def __call__(self, filename, binary=None, from_page=0, to_page=MAXIMUM_PAGE_NUMBER):
        self.doc = Document(filename) if not binary else Document(BytesIO(binary))
        lines = []
        level_set = set()
        
        # Categorize bullet/numbering styles
        bull = bullets_category([p.text for p in self.doc.paragraphs])
        
        for p in self.doc.paragraphs:
            # Extract question level from paragraph style
            question_level, p_text = docx_question_level(p, bull)
            if not p_text.strip("\n"):
                continue
            lines.append((question_level, p_text))
            level_set.add(question_level)
        
        # Determine H2 level (secondary heading level)
        sorted_levels = sorted(level_set)
        h2_level = sorted_levels[1] if len(sorted_levels) > 1 else 1
        
        # Build tree with detected hierarchy
        root = Node(level=0, depth=h2_level, texts=[])
        root.build_tree(lines)
        
        return [element for element in root.get_tree() if element]
```

---

### PDF Processing Flow

For PDF documents, the parser uses OCR and layout analysis:

```python
# File: /ragflow/rag/app/laws.py

class Pdf(PdfParser):
    def __init__(self):
        self.model_speciess = ParserType.LAWS.value
        super().__init__()
    
    def __call__(self, filename, binary=None, from_page=0, 
                 to_page=MAXIMUM_PAGE_NUMBER, zoomin=3, callback=None):
        # Step 1: OCR
        callback(msg="OCR started")
        self.__images__(filename if not binary else binary, zoomin, from_page, to_page, callback)
        callback(msg="OCR finished")
        
        # Step 2: Layout recognition
        self._layouts_rec(zoomin)
        callback(0.67, "Layout analysis")
        
        # Step 3: Vertical merge (lines → blocks)
        self._naive_vertical_merge()
        callback(0.8, "Text extraction")
        
        # Return text with position tags
        return [(b["text"], self._line_tag(b, zoomin)) for b in self.boxes], None
```

---

### Main Chunk Function

The main `chunk()` function orchestrates the entire pipeline:

```python
# File: /ragflow/rag/app/laws.py

def chunk(filename, binary=None, from_page=0, to_page=MAXIMUM_PAGE_NUMBER, 
          lang="Chinese", callback=None, **kwargs):
    """
    Main entry point for legal document parsing.
    
    Args:
        filename: Document filename
        binary: Document bytes (optional)
        from_page: Start page (for PDF)
        to_page: End page (for PDF)
        lang: Language ("Chinese" or "English")
        callback: Progress callback function
    
    Returns:
        List of tokenized chunks ready for embedding
    """
    parser_config = kwargs.get("parser_config", {
        "chunk_token_num": 512,
        "delimiter": "\n!?",
        "layout_recognize": "DeepDOC"
    })
    
    # Initialize document metadata
    doc = {
        "docnm_kwd": filename,
        "title_tks": rag_tokenizer.tokenize(re.sub(r"\.[a-zA-Z]+$", "", filename))
    }
    
    # Route to appropriate parser based on file extension
    if re.search(r"\.docx$", filename, re.IGNORECASE):
        chunks = Docx()(filename, binary)
        return tokenize_chunks(chunks, doc, eng, None)
    
    elif re.search(r"\.pdf$", filename, re.IGNORECASE):
        raw_sections, tables, pdf_parser = parser(...)
        # Process sections...
    
    elif re.search(r"\.(txt|md|markdown)$", filename, re.IGNORECASE):
        txt = get_text(filename, binary)
        sections = txt.split("\n")
    
    # Common post-processing for all formats
    remove_contents_table(sections, eng)  # Remove TOC section
    make_colon_as_title(sections)         # Convert "Title: Content" format
    bull = bullets_category(sections)     # Detect heading pattern
    res = tree_merge(bull, sections, 2)    # Build hierarchy (depth=2)
    
    return tokenize_chunks(res, doc, eng, pdf_parser)
```

---

### Preprocessing Functions

#### 1. Remove Table of Contents

```python
def remove_contents_table(sections, eng=False):
    """
    Remove table of contents section from legal documents.
    Detects patterns like 'Contents', 'Table of Contents', 'Acknowledgments'.
    """
    i = 0
    while i < len(sections):
        if not re.match(r"(contents|table of contents|acknowledge)$",
                        re.sub(r"( | )+", "", get(i).split("@@")[0], flags=re.IGNORECASE):
            i += 1
            continue
        sections.pop(i)
        # Remove following lines that match the TOC pattern
        # ... (continues to remove entire TOC section)
```

#### 2. Colon-to-Title Conversion

```python
def make_colon_as_title(sections):
    """
    Convert 'Title: Content' format to separate title element.
    Example: 'Chapter I: General Provisions. This Law is enacted...'
    → 'Chapter I: General Provisions' + 'This Law is enacted...'
    """
    for i, (txt, layout) in enumerate(sections):
        if txt[-1] not in ":":
            continue
        # Split on sentence boundaries
        arr = re.split(r"([.?!;]| \.)", txt[::-1])
        if len(arr) < 2 or len(arr[1]) < 32:
            continue
        # Insert title as separate element
        sections.insert(i, (arr[0][::-1], "title"))
```

---

### Configuration Options

| Option | Default | Description |
|--------|---------|-------------|
| `chunk_token_num` | 512 | Maximum tokens per chunk |
| `delimiter` | `\n!?` | Sentence delimiters for splitting |
| `layout_recognize` | `DeepDOC` | Layout analysis method |
| `task_page_size` | 12 | Pages per processing task |

---

### Example: Parsing a Contract Law Document

**Input:** `contract_law.pdf`

```
CONTRACT ACT
CHAPTER I: GENERAL PROVISIONS
Article 1 This Act is enacted for the purpose of protecting...
  1.1 Principle of Equality
    The parties shall follow...
  1.2 Principle of Voluntariness
    The parties shall enjoy...

CHAPTER II: FORMATION OF CONTRACTS
Article 2 The parties may conclude a contract...
```

**Output Chunks:**

```
Chunk 1 (tokens: 156):
─────────────────────────────────────────────────────────────────────
CHAPTER I: GENERAL PROVISIONS
Article 1 This Act is enacted for the purpose of protecting the legitimate
rights and interests of the parties to contracts, maintaining the
socio-economic order...

Chunk 2 (tokens: 203):
─────────────────────────────────────────────────────────────────────
CHAPTER I: GENERAL PROVISIONS
1.1 Principle of Equality
The parties shall follow the principles of equality, mutual benefit,
and consultation...

Chunk 3 (tokens: 178):
─────────────────────────────────────────────────────────────────────
CHAPTER I: GENERAL PROVISIONS
1.2 Principle of Voluntariness
The parties shall enjoy the right to voluntarily conclude contracts
in accordance with the law...

Chunk 4 (tokens: 134):
─────────────────────────────────────────────────────────────────────
CHAPTER II: FORMATION OF CONTRACTS
Article 2 The parties may conclude a contract, shall have the appropriate
civil rights capacity and civil conduct capacity...
```

---

### When to Use Laws Parser

| Document Type | Use Laws Parser? |
|---------------|------------------|
| **Statutes & Laws** | ✅ Ideal |
| **Contracts** | ✅ Ideal |
| **Regulations** | ✅ Ideal |
| **Court Decisions** | ✅ Good (if structured) |
| **Legal Briefs** | ✅ Good |
| **Case Law** | ⚠️ May need QA parser |
| **General Text** | ❌ Use General instead |

---

### Key Takeaways

1. **Pattern-Based Detection**: Uses 5 different bullet patterns to detect legal hierarchy
2. **Tree Structure**: Builds proper tree for hierarchical chunking
3. **Content Preservation**: Maintains article numbers and legal citations
4. **Multi-Format**: Supports PDF, DOCX, TXT, HTML, Markdown
5. **Preprocessing**: Automatically removes TOC and converts colon titles

---

## 4. Paper

**Best for:** Research papers, academic articles, technical reports

**Features:**
- Academic section detection (Abstract, Introduction, etc.)
- Title frequency analysis
- Citation handling
- Table and figure extraction
- Two-column layout support

**Processing:**
1. OCR with high zoom
2. Layout analysis for academic format
3. Table transformation
4. Section identification
5. Hierarchical chunking by sections

---

## 5. Book

**Best for:** Books, long-form documents with chapters

**Features:**
- Chapter detection
- Hierarchical merging
- Table of contents handling
- Section-based chunking
- Preserves reading order

**Processing:**
- Detects chapter breaks
- Builds hierarchical structure
- Merges content by chapter
- Removes table of contents
- Handles multi-page tables

---

## 6. Presentation

**Best for:** PowerPoint files, presentation PDFs

**Features:**
- Slide-by-slide extraction
- Layout analysis per slide
- Table extraction from slides
- PPT and PDF support
- Preserves slide order

**Processing:**
- OCR for scanned slides
- Layout analysis per slide
- Table/figure extraction
- Page-level chunking (one chunk per slide)

---

## 7. Manual

**Best for:** Technical manuals, user guides, tutorials

**Features:**
- Procedure extraction
- Step-by-step detection
- Numbered list handling
- Code block preservation
- Troubleshooting section detection

**Processing:**
- Detects procedural content
- Preserves numbered steps
- Handles code examples
- Extracts warnings and notes

---

## 8. QA (Question-Answer)

**Best for:** FAQ documents, Q&A pairs, interview transcripts

**Features:**
- Question-answer pair extraction
- Supports Excel with Q&A columns
- Bullet-based Q&A detection
- Preserves Q&A relationship

**Supported formats:**
- Excel with 2 columns (Question, Answer)
- Word documents with Q&A formatting
- Markdown with Q&A structure

---

## 9. Resume

**Best for:** Resumes, CVs, job applications

**Features:**
- Structured field extraction (name, contact, experience, education)
- Layout-aware reconstruction
- Parallel LLM extraction (3-way: basic info, work, education)
- Index pointer mechanism (reduces hallucination)
- Domain normalization (dates, locations)

**Processing Pipeline:**
1. PDF text fusion (metadata + OCR)
2. Layout segmentation (YOLOv10)
3. Hierarchical sorting
4. LLM-based extraction
5. Four-stage post-processing

---

## 10. Picture

**Best for:** Images, videos, screenshots

**Features:**
- Vision-based description generation
- Video frame extraction
- OCR for text in images
- Multi-modal LLM support

**Supported formats:**
- Images: PNG, JPG, etc.
- Videos: MP4, MOV, AVI, WebM, etc.

**Processing:**
- Uses vision LLM (image-to-text)
- Extracts video frames for processing
- OCR for embedded text
- Generates descriptive chunks

---

## 11. One

**Best for:** Short documents that should stay as one chunk

**Features:**
- No splitting
- Entire document as single chunk
- Preserves complete context

**Use cases:**
- Short memos
- Single-page documents
- Documents requiring holistic understanding

---

## 12. Audio

**Best for:** Audio files, podcasts, voice notes

**Features:**
- Speech-to-text transcription
- Multi-format support
- Uses LLM for transcription

**Supported formats:**
- MP3, WAV, AAC, FLAC, OGG, AIFF, AU, MIDI, WMA

**Processing:**
- Transcribes audio to text
- Creates single chunk with transcription
- Requires speech-to-text LLM model

---

## 13. Email

**Best for:** Email files (.eml)

**Features:**
- Header extraction (From, To, Subject, Date)
- Body content parsing
- Attachment handling
- HTML and text support

**Processing:**
- Extracts email metadata
- Parses body content
- Processes attachments separately
- Maintains email threading context

---

## 14. Tag

**Best for:** Documents with content-tag associations

**Features:**
- Content-tag pair extraction
- Excel/CSV support (2 columns: content, tags)
- Tag-based retrieval
- Multi-tag support per chunk

**Supported formats:**
- Excel with [Content, Tags] columns
- CSV with TAB or COMMA delimiter
- TXT with delimiter-separated values

**Processing:**
- Each row becomes a chunk
- Tags stored for filtering
- Comma-separated tags supported

---

## Parser Configuration Options

All parsers support these configuration options:

| Option | Description | Default |
|--------|-------------|---------|
| `chunk_token_num` | Maximum tokens per chunk | 512 |
| `layout_recognize` | Layout analysis method | DeepDOC |
| `delimiter` | Chunk delimiter | `\n` |
| `task_page_size` | Pages per task | 12 |
| `raptor.use_raptor` | Enable RAPTOR summarization | false |
| `graphrag.use_graphrag` | Enable GraphRAG | false |

---

## Choosing the Right Parser

| Document Type | Recommended Parser |
|---------------|-------------------|
| General PDF/Word | **General** |
| Excel spreadsheets | **Table** |
| Legal contracts | **Laws** |
| Research papers | **Paper** |
| Books | **Book** |
| PowerPoint | **Presentation** |
| Technical guides | **Manual** |
| FAQ documents | **QA** |
| Resumes | **Resume** |
| Images | **Picture** |
| Short documents | **One** |
| Audio recordings | **Audio** |
| Email archives | **Email** |
| Tagged content | **Tag** |

---

## Parser Selection Tips

1. **Start with General** - Works for most documents
2. **Use Table for Excel** - Preserves table structure
3. **Use Laws for legal** - Maintains legal hierarchy
4. **Use Resume for CVs** - Extracts structured fields
5. **Use One for short docs** - Keeps context intact

---

## References

- Parser source code: `/ragflow/rag/app/`
- Parser types: `/ragflow/common/constants.py` (ParserType enum)
- Task executor: `/ragflow/rag/svr/task_executor.py`
