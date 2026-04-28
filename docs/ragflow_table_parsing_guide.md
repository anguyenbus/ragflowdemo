# RAGFlow Table Parsing Guide

## Overview

RAGFlow provides sophisticated table parsing capabilities that handle structured data from multiple sources including Excel files, CSV files, and tables embedded within PDF documents. The table parser extracts, processes, and preserves table structure while making content searchable and retrievable.

---

## Supported Table Sources

| Source | File Format | Parser Type |
|--------|-------------|-------------|
| **Excel** | .xlsx, .xls | `Table` parser |
| **CSV** | .csv | `Table` parser |
| **TXT (Tab-delimited)** | .txt | `Table` parser |
| **PDF (embedded tables)** | .pdf | `General` parser with layout analysis |
| **DOCX (embedded tables)** | .docx | `General` parser |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         TABLE PARSING PIPELINE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  INPUT                    PROCESSING                  OUTPUT                │
│  ┌─────────┐             ┌─────────────┐             ┌──────────┐           │
│  │ Excel   │             │   Header    │             │ Chunks   │           │
│  │ CSV     │────────────▶│ Detection   │────────────▶│ + Fields │           │
│  │ TXT     │             │   Type      │             │ + Image  │           │
│  │ PDF     │             │ Inference   │             │ + JSON   │           │
│  └─────────┘             └─────────────┘             └──────────┘           │
│                                   │                                          │
│                                   ▼                                          │
│                        ┌─────────────────┐                                  │
│                        │  Cell Images    │                                  │
│                        │  (Vision LLM)   │                                  │
│                        └─────────────────┘                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Excel Table Parsing

### File Location
`/ragflow/rag/app/table.py` - Main table parser implementation
`/ragflow/deepdoc/parser/excel_parser.py` - Base Excel parser

### Parsing Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          EXCEL PARSING FLOW                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. WORKBOOK LOADING                                                        │
│     │                                                                        │
│     ├──▶ Try openpyxl (native Excel)                                       │
│     ├──▶ Fallback: pandas (CSV conversion)                                 │
│     └──▶ Fallback: calamine engine (performance)                           │
│                                                                              │
│  2. SHEET ITERATION                                                         │
│     │                                                                        │
│     ├──▶ Extract images from worksheet                                     │
│     ├──▶ Get actual row count (binary search for large files)              │
│     └──▶ Process rows within page range                                    │
│                                                                              │
│  3. HEADER DETECTION                                                        │
│     │                                                                        │
│     ├──▶ Check for merged cells (complex header)                           │
│     ├──▶ Detect multi-level headers                                        │
│     └──▶ Build hierarchical column names                                   │
│                                                                              │
│  4. ROW EXTRACTION                                                          │
│     │                                                                        │
│     ├──▶ Handle merged cell values                                         │
│     ├──▶ Inherit values from merged regions                                │
│     └──▶ Skip empty rows                                                    │
│                                                                              │
│  5. IMAGE PROCESSING (Optional)                                             │
│     │                                                                        │
│     ├──▶ Extract embedded images                                           │
│     ├──▶ Classify as single-cell or multi-cell                             │
│     └──▶ Generate vision LLM descriptions                                   │
│                                                                              │
│  6. DATA TYPE INFERENCE                                                     │
│     │                                                                        │
│     ├──▶ Detect: int, float, datetime, bool, text                          │
│     └──▶ Transform values to detected types                                │
│                                                                              │
│  7. CHUNKING                                                                │
│     │                                                                        │
│     └──▶ One row = One chunk (with formatted text)                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Header Detection Algorithm

```python
# File: /ragflow/rag/app/table.py

def _parse_headers(self, ws, rows):
    """Detect simple vs complex headers"""
    has_complex = self._has_complex_header_structure(ws, rows)

    if has_complex:
        # Multi-level headers with merged cells
        return self._parse_multi_level_headers(ws, rows)
    else:
        # Simple single-row headers
        return self._parse_simple_headers(rows)

def _has_complex_header_structure(self, ws, rows):
    """Check if first 2 rows have merged cells"""
    merged_ranges = list(ws.merged_cells.ranges)
    for rng in merged_ranges:
        if rng.min_row <= 2:
            return True
    return False
```

### Header vs Data Classification

```python
def _row_looks_like_header(self, row):
    """Classify row as header or data"""
    header_like = 0
    data_like = 0

    for cell in row:
        val = str(cell.value).strip()

        if self._looks_like_header(val):
            header_like += 1
        elif self._looks_like_data(val):
            data_like += 1

    return header_like >= data_like

def _looks_like_header(self, value):
    """Header indicators: non-ASCII, multiple letters, special chars"""
    if any(ord(c) > 127 for c in value):
        return True  # Chinese, Japanese, etc.
    if len([c for c in value if c.isalpha()]) >= 2:
        return True
    if any(c in value for c in ["(", ")", ":", "_", "-"]):
        return True
    return False

def _looks_like_data(self, value):
    """Data indicators: single letters, numbers, dates"""
    if len(value) == 1 and value.upper() in ["Y", "N", "M", "X", "/", "-"]:
        return True
    if value.replace(".", "").replace("-", "").replace(",", "").isdigit():
        return True
    if value.startswith("0x") and len(value) <= 10:
        return True
    return False
```

### Multi-Level Header Handling

For complex Excel sheets with merged cells across multiple header rows:

```python
def _build_hierarchical_headers(self, ws, rows, header_rows):
    """Build column names from multi-row headers"""
    headers = []
    max_col = max(len(row) for row in rows[:header_rows])

    for col_idx in range(max_col):
        header_parts = []

        # Collect non-empty values from each header row
        for row_idx in range(header_rows):
            cell_value = self._get_merged_cell_value(ws, row_idx + 1, col_idx + 1)
            if cell_value and self._is_valid_header_part(cell_value):
                header_parts.append(str(cell_value).strip())

        # Join with hyphen: "Category-SubCategory-Metric"
        if header_parts:
            headers.append("-".join(header_parts))
        else:
            headers.append(f"Column_{col_idx + 1}")

    return headers
```

**Example:**
```
Original Excel:
┌──────────┬────────────┬────────┐
│ Category │  Metrics   │        │
├──────────┼────────────┼────────┤
│          │ Revenue    │ Profit │
├──────────┼────────────┼────────┤
│ Electronics │ 1000    │ 200    │
│ Clothing    │ 500     │ 50     │
└──────────┴────────────┴────────┘

Result: ["Category", "Metrics-Revenue", "Metrics-Profit"]
```

### Data Type Inference

```python
def column_data_type(arr):
    """Infer column data type from sample values"""
    counts = {"int": 0, "float": 0, "text": 0, "datetime": 0, "bool": 0}

    for value in arr:
        if value is None:
            continue

        # Pattern matching for type detection
        if re.match(r"[+-]?[0-9]+$", str(value).replace("%%", "")):
            counts["int"] += 1
        elif re.match(r"[+-]?[0-9.]{,19}$", str(value).replace("%%", "")):
            counts["float"] += 1
        elif re.match(r"(true|yes|是|\*|✓|✔|☑|✅|√)$", str(value), flags=re.IGNORECASE):
            counts["bool"] += 1
        elif trans_datatime(str(value)):
            counts["datetime"] += 1
        else:
            counts["text"] += 1

    # Return most common type
    return sorted(counts.items(), key=lambda x: x[1] * -1)[0][0]
```

### Boolean Value Normalization

```python
def trans_bool(s):
    """Normalize various boolean representations"""
    true_patterns = r"(true|yes|是|\*|✓|✔|☑|✅|√)$"
    false_patterns = r"(false|no|否|⍻|×)$"

    if re.match(true_patterns, str(s).strip(), flags=re.IGNORECASE):
        return "yes"
    if re.match(false_patterns, str(s).strip(), flags=re.IGNORECASE):
        return "no"
    return None
```

---

## 2. Embedded Image Handling

Excel files can contain images within cells or spanning multiple cells:

```python
# Image classification
for img in images:
    if img["span_type"] == "single_cell":
        # Insert image description into cell value
        df.iat[row_idx, col_idx] = img["image_description"]
    else:
        # Multi-cell image - store as separate table chunk
        flow_images.append(img)
```

### Vision LLM Integration

```python
# File: /ragflow/deepdoc/parser/figure_parser.py

def vision_figure_parser_figure_xlsx_wrapper(images, callback=None, **kwargs):
    """Generate descriptions for embedded images using vision LLM"""

    # Get vision model configuration
    vision_model_config = get_tenant_default_model_by_type(
        kwargs["tenant_id"], LLMType.IMAGE2TEXT
    )
    vision_model = LLMBundle(kwargs["tenant_id"], vision_model_config)

    # Process images with vision LLM
    parser = VisionFigureParser(vision_model=vision_model, figures_data=images)
    boosted_figures = parser(callback=callback)

    return boosted_figures
```

---

## 3. Chunk Generation

### Row-Based Chunking

Each row becomes a separate chunk with structured formatting:

```python
for ii, row in df.iterrows():
    d = {"docnm_kwd": filename}

    # Build field map for search
    row_fields = []
    data_json = {}

    for j in range(len(columns)):
        if row[columns[j]] is not None:
            # Store based on database engine
            if settings.DOC_ENGINE_INFINITY:
                data_json[str(columns[j])] = row[columns[j]]
            else:
                # Elasticsearch: add type suffix
                fld = f"{py_column}_{type_suffix}"
                d[fld] = row[columns[j]] if type != "text" else tokenize(row[columns[j]])

            row_fields.append((columns[j], row[columns[j]]))

    # Format as structured text for LLM comprehension
    formatted_text = "\n".join([f"- {field}: {value}" for field, value in row_fields])

    # Tokenize and store
    tokenize(d, formatted_text, eng)
    res.append(d)
```

### Chunk Format Example

**Input Excel:**
| Product | Category | Price | Stock |
|---------|----------|-------|-------|
| Widget A | Electronics | 29.99 | 150 |
| Widget B | Clothing | 19.99 | 200 |

**Output Chunk:**
```
- Product: Widget A
- Category: Electronics
- Price: 29.99
- Stock: 150
```

### Field Mapping by Database Engine

| Database Engine | Storage Format | Example |
|-----------------|----------------|---------|
| **Elasticsearch** | Individual fields with type suffixes | `product_kwd: "Widget A"`, `price_flt: 29.99` |
| **Infinity/OceanBase** | JSON object in `chunk_data` | `{"Product": "Widget A", "price": 29.99}` |

### Field Type Suffixes

```python
fields_map = {
    "text": "_tks",      # Tokenized text
    "int": "_long",      # Long integer
    "keyword": "_kwd",   # Exact match keyword
    "float": "_flt",     # Floating point
    "datetime": "_dt",   # Date/time
    "bool": "_kwd"       # Boolean as keyword
}
```

---

## 4. CSV and TXT Parsing

### CSV Format

```python
# Delimiter detection
delimiter = kwargs.get("delimiter", ",")

# Parse with error handling
reader = csv.reader(io.StringIO(txt), delimiter=delimiter)
all_rows = list(reader)

headers = all_rows[0]
rows = all_rows[1:]

# Validate row length
for i, row in enumerate(rows):
    if len(row) != len(headers):
        fails.append(str(i))  # Track malformed rows
        continue
```

### TXT (Tab-Delimited) Format

```python
# First line must be headers
headers = lines[0].split("\t")

# Subsequent lines are data
for line in lines[1:]:
    row = line.split("\t")
    if len(row) != len(headers):
        continue  # Skip malformed rows
    rows.append(row)
```

---

## 5. PDF Table Extraction

### Layout Analysis Integration

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PDF TABLE EXTRACTION                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. OCR (if scanned)                                                        │
│     │                                                                        │
│     ▼                                                                        │
│  2. Layout Recognition (DeepDOC/PaddleOCR/DocLing)                          │
│     │                                                                        │
│     ├──▶ Detect table regions                                               │
│     ├──▶ Classify layout type (table/text/image)                            │
│     └──▶ Extract cell boundaries                                            │
│                                                                              │
│  3. Table Structure Recognition (TSR)                                       │
│     │                                                                        │
│     ├──▶ Evaluate table orientation (auto-rotate)                           │
│     ├──▶ Detect cell boundaries                                             │
│     └──▶ Extract cell contents                                              │
│                                                                              │
│  4. Output Formatting                                                        │
│     │                                                                        │
│     ├──▶ HTML table markup                                                  │
│     └──▶ Plain text representation                                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Table Orientation Detection

```python
# File: /ragflow/deepdoc/parser/pdf_parser.py

def _evaluate_table_orientation(self, table_img, sample_ratio=0.3):
    """Find best rotation angle for table image"""

    angles = [0, 90, 180, 270]
    best_angle = 0
    best_score = 0

    for angle in angles:
        # Rotate image
        rotated = table_img.rotate(-angle, expand=True)

        # Run TSR and get confidence score
        ocr_result = self.table_model(rotated)
        score = calculate_confidence(ocr_result)

        if score > best_score:
            best_score = score
            best_angle = angle

    return best_angle, best_rotated_img
```

---

## 6. Column Name Processing

### Pinyin Conversion for Chinese Headers

```python
PY = Pinyin()

# Convert Chinese column names to Pinyin for database compatibility
clmns = ["姓名", "电话", "地址"]
py_clmns = [PY.get_pinyins(re.sub(r"(/.*|（[^（）]+?）)", "", str(n)), "_")[0]
            for n in clmns]

# Result: ["xing_ming", "dian_hua", "di_zhi"]
```

### Synonym Handling in Headers

Headers can include synonyms and value hints for better NLP understanding:

```
Recommended format:
- gender/sex(male, female)
- phone/mobile/联系电话
- size(M,L,XL,XXL)
- 最高学历（高中，本科，硕士，博士）
```

This helps the parser understand:
1. Alternative names for the same field
2. Possible enum values
3. Language variations

---

## 7. Configuration Options

### Parser Configuration

```yaml
parser_config:
  chunk_token_num: 512          # Not used for table (each row is a chunk)
  delimiter: "\t"               # For TXT files
  layout_recognize: "DeepDOC"   # For PDF tables
```

### Field Mapping Storage

```python
# Stored in knowledgebase parser_config
field_map = {
    "product_kwd": "Product",
    "category_kwd": "Category",
    "price_flt": "Price",
    "stock_long": "Stock"
}

KnowledgebaseService.update_parser_config(kb_id, {"field_map": field_map})
```

---

## 8. Output Structure

### Chunk Document Structure

```python
{
    # Document metadata
    "docnm_kwd": "products.xlsx",
    "title_tks": ["products"],

    # For Elasticsearch/OpenSearch: individual typed fields
    "product_kwd": "Widget A",
    "category_kwd": "Electronics",
    "price_flt": 29.99,
    "stock_long": 150,

    # For Infinity/OceanBase: JSON object
    "chunk_data": {
        "Product": "Widget A",
        "Category": "Electronics",
        "Price": 29.99,
        "Stock": 150
    },

    # Tokenized content for search
    "content_with_weight": "- Product: Widget A\n- Category: Electronics\n...",

    # Document type
    "doc_type_kwd": "table",

    # Optional: embedded image
    "image": <LazyImage or PIL.Image>
}
```

---

## 9. Best Practices

### Excel File Preparation

1. **Use clear column headers**
   - Avoid empty headers (will generate "Column_N")
   - Use meaningful names
   - Include synonyms with "/" separator

2. **Avoid duplicate column names**
   - Will raise `ValueError`
   - Use qualifiers like "Price (USD)" vs "Price (EUR)"

3. **Handle merged cells properly**
   - Parser automatically detects multi-level headers
   - Values inherited from merged regions

4. **Large file optimization**
   - Parser uses binary search for actual row count
   - Supports page range extraction (`from_page`, `to_page`)

### CSV/TXT File Preparation

1. **First line must be headers**
   - No headerless support
   - Use meaningful column names

2. **Consistent delimiter**
   - CSV: comma (default) or custom
   - TXT: tab (default)

3. **Row length validation**
   - Malformed rows skipped and logged
   - Error reporting includes line numbers

---

## 10. Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| "Duplicate column names detected" | Two columns have same name | Rename columns to be unique |
| "Skip sheet due to rows access error" | Corrupted sheet | Remove problematic sheet |
| Images not extracted | Missing vision model | Configure IMAGE2TEXT model |
| Wrong data type | Pattern matching error | Use explicit data formats |
| Chinese headers not searchable | Database incompatible | Configure ES/Infinity properly |

### Debug Logging

```python
# Enable debug logging for field mapping
logging.debug(f"Field map: {field_map}")

# Check column type inference
logging.info(f"Column types: {clmn_tys}")
```

---

## References

- Table parser: `/ragflow/rag/app/table.py`
- Excel base parser: `/deepdoc/parser/excel_parser.py`
- PDF table extraction: `/deepdoc/parser/pdf_parser.py`
- Vision figure parser: `/deepdoc/parser/figure_parser.py`
- Tokenization: `/ragflow/nlp/__init__.py`
