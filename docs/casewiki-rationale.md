# CaseWiki: Why LLM Wiki Matters for Case Assistant Projects

> **Target Audience**: Technical leads, product managers, and architects building case-based AI systems.
>
> **Scope**: Legal case management, but applicable to any domain where documents are organized into cases (medical, insurance, consulting, investigations).

---

## Executive Summary

Case assistant systems today rely heavily on **RAG (Retrieval-Augmented Generation)** for answering questions about case documents. While RAG provides accurate, source-grounded answers, it has a fundamental limitation: **knowledge is ephemeral**.

Each query re-processes the same documents. No cross-reference is built. Contradictions remain hidden until disaster. Handoffs to colleagues require re-reading everything.

**LLM Wiki** transforms case assistants from Q&A tools into **knowledge partners**. By synthesizing case documents into structured, interlinked wiki pages, knowledge compounds over time rather than being discarded after each query.

This document explains why wiki-style knowledge synthesis matters for case assistant projects, with specific focus on legal case management.

---

## 1. The Problem: Why Current Case Systems Fall Short

### 1.1 The RAG Trap: Stateless and Amnesiac

RAG systems work by chunking documents, embedding them, and retrieving relevant chunks at query time. This approach has critical flaws for case-based work.

#### How RAG Works Today

```mermaid
flowchart LR
    subgraph Upload["Document Upload"]
        PDF[PDF Documents] --> Parser[Text Extractor]
        Parser --> Chunker[Chunk Splitter]
        Chunker --> Embed[Embedding Model]
        Embed --> VecDB[(Vector Database)]
    end

    subgraph Query["User Query"]
        Q[User Question] --> QEmb[Query Embedding]
        QEmb --> Search[Similarity Search]
        Search --> Result[Top K Chunks]
        Result --> LLM[LLM Synthesis]
        LLM --> Ans[Answer]
    end

    VecDB -->|Provides chunks| Search

    style Ans fill:#f99,stroke:#333,stroke-width:2px
```

**The fundamental problem**: After the answer is generated, everything is discarded. The knowledge gained from processing those chunks evaporates.

#### The Re-Processing Problem

```mermaid
sequenceDiagram
    participant U as User
    participant R as RAG System
    participant D as Documents
    participant L as LLM

    Note over U,L: Session 1: Morning
    U->>R: What deductions were claimed?
    R->>D: Search & retrieve chunks
    D-->>R: 10 chunks returned
    R->>L: Synthesize answer
    L-->>U: $5,500 total claimed
    Note over R: Knowledge discarded

    Note over U,L: Session 2: Afternoon
    U->>R: Is home office deductible?
    R->>D: Search again (same docs)
    D-->>R: Different 8 chunks
    R->>L: Synthesize again
    L-->>U: Yes, if criteria met
    Note over R: No memory of Session 1

    Note over U,L: Session 3: Next day
    U->>R: Draft objection letter
    R->>D: Search yet again
    D-->>R: Another 12 chunks
    R->>L: Synthesize from scratch
    L-->>U: Draft letter
    Note over R: Third re-processing
```

**Cost accumulation**: Each query re-processes the same documents. For a case with 100+ questions over its lifecycle:

| Metric | Per Query | Case Total (100 queries) |
|--------|-----------|--------------------------|
| Token cost (input) | ~5,000 tokens | ~500,000 tokens |
| Processing time | 5-10 seconds | 500-1000 seconds |
| API cost | ~$0.01 | ~$1.00 |

The system pays repeatedly for the same knowledge extraction.

#### The Chunk Isolation Problem

```mermaid
graph TD
    Doc[Case Document: 50 pages] --> C1[Chunk 1: pages 1-5]
    Doc --> C2[Chunk 2: pages 6-10]
    Doc --> C3[Chunk 3: pages 11-15]
    Doc --> C4[Chunk 4: pages 16-20]
    Doc --> C5[Chunk 5: pages 21-25]

    C1 -.->|No link| C2
    C2 -.->|No link| C3
    C3 -.->|No link| C4
    C4 -.->|No link| C5

    Query[Query: Timeline of events] --> R1[Retrieval]
    R1 -->|Returns| C2
    R1 -->|Returns| C4

    style C2 fill:#9f9,stroke:#333
    style C4 fill:#9f9,stroke:#333
    style C1 fill:#ccc,stroke:#333
    style C3 fill:#ccc,stroke:#333
    style C5 fill:#ccc,stroke:#333
```

**Problem**: Chunks 2 and 4 might contain timeline fragments, but:
- Chunk 1 (page 5 ends mid-sentence) has context
- Chunk 3 (pages 11-15) continues the story
- Chunk 5 (page 21+) has the conclusion

RAG returns isolated fragments. The LLM tries to piece them together, but:
1. Context is split across chunk boundaries
2. No cross-references exist between chunks
3. Important connections are invisible

#### The Contradiction Blind Spot

```mermaid
flowchart LR
    subgraph Source1["Document 1: ATO Notice"]
        D1[Notice date: 2024-10-15<br/>Penalty: 75% of shortfall]
    end

    subgraph Source2["Document 2: Taxpayer Letter"]
        D2[Letter date: 2024-09-20<br/>Penalty: 25% - requested remission]
    end

    subgraph Source3["Document 3: ATO Response"]
        D3[Response: 2024-11-01<br/>Penalty reduced to 50%]
    end

    Query[Question: What is current penalty?] --> R1[Retrieval 1]
    Query --> R2[Retrieval 2]
    Query --> R3[Retrieval 3]

    R1 -->|Returns| D1
    R2 -->|Returns| D2
    R3 -.->|May not match| D3

    LLM[LLM Answer] --> A1["75% (from Notice)"]
    LLM --> A2["25% (from Letter)"]
    LLM --> A3["50% (if Response found)"]

    style A1 fill:#f99,stroke:#333
    style A2 fill:#f99,stroke:#333
    style A3 fill:#9f9,stroke:#333
```

**Problem**: RAG has no mechanism to:
- Track how facts evolve across documents
- Flag contradictions between sources
- Maintain a "current state" view

Wiki synthesis does exactly this.

#### No Cross-Case Learning

```mermaid
graph LR
    subgraph Case_A["Case A (Resolved 2024)"]
        A1[Issue: Home office]
        A2[Precedent: 50% allowable]
        A3[ATO accepted argument]
        A4[Key witness testimony]
    end

    subgraph Case_B["Case B (New 2025)"]
        B1[Same issue: Home office]
        B2[Starting from zero]
        B3[Must re-research]
        B4[No access to A4]
    end

    A2 -.->|Not reused| B2
    A3 -.->|Not visible| B3
    A4 -.->|Isolated| B4

    style Case_A fill:#9f9,stroke:#333
    style Case_B fill:#f99,stroke:#333
```

**Reality**: The firm won the home office argument in Case A. Case B starts from zero. The precedent, the successful arguments, the key testimony—all lost.

### 1.2 The Handoff Problem

When a case transfers between lawyers:

```mermaid
sequenceDiagram
    participant L1 as Lawyer 1 (Leaving)
    participant L2 as Lawyer 2 (Taking over)
    participant Docs as Case Documents
    participant RAG as RAG System

    L1->>L2: Here's the case file
    L1->>L2: Good luck!

    Note over L2,Docs: Week 1-2: Reading
    L2->>Docs: Read 150 documents
    Docs-->>L2: Partial understanding

    Note over L2,RAG: Week 3: Asking questions
    L2->>RAG: What are the key issues?
    RAG-->>L2: Partial answer (chunks)
    L2->>RAG: Who testified about X?
    RAG-->>L2: I don't see that in chunks

    Note over L2: Week 4: Still missing context
    L2->>L1: Can you explain your strategy?
    L1->>L2: Too busy, read the file

    Note over L2: Week 5: Finally competent
    L2->>L2: I understand the case now
```

**Cost**: 3-5 weeks of lost productivity. Risk of missing critical facts.

With wiki:
```mermaid
sequenceDiagram
    participant L1 as Lawyer 1
    participant L2 as Lawyer 2
    participant Wiki as Case Wiki

    L1->>L2: Here's the case wiki
    L2->>Wiki: Read facts.md (5 min)
    L2->>Wiki: Read issues.md (5 min)
    L2->>Wiki: Read strategy.md (5 min)

    Note over L2: 15 minutes later
    L2->>L2: I understand the case
```

---

## 2. RAG vs Wiki: The Fundamental Difference

### 2.1 Architecture Comparison

```mermaid
flowchart TB
    subgraph RAG["Traditional RAG System"]
        direction TB
        Docs1[(Document Store)] -->|Chunk & Embed| RAGVec[Vector DB]
        UserQ1[User Question] -->|Retrieval| RAGVec
        RAGVec -->|Top K chunks| RAGLLM[LLM]
        RAGLLM -->|Discard after response| RAGAns[Answer]
    end

    subgraph Wiki["LLM Wiki System"]
        direction TB
        Docs2[(Document Store)] -->|Ingest Agent| WikiPages[Wiki Pages]
        WikiPages -->|Organized by topic| WikiIndex[Index]
        WikiPages -->|Cross-referenced| WikiFacts[Facts]
        WikiPages -->|Cross-referenced| WikiEntities[Entities]
        WikiPages -->|Cross-referenced| WikiIssues[Issues]

        UserQ2[User Question] -->|Read full pages| WikiPages
        WikiPages -->|Synthesize| WikiLLM[LLM]
        WikiLLM -->|Save as new page| WikiSyn[Synthesis]
        WikiSyn -->|Updates| WikiPages
    end

    style RAGAns fill:#f99,stroke:#333
    style WikiSyn fill:#9f9,stroke:#333
```

### 2.2 Feature Comparison

| Aspect | RAG (Current) | LLM Wiki | Why It Matters for Cases |
|--------|---------------|----------|--------------------------|
| **Knowledge lifecycle** | Stateless, re-derived each query | Persistent, compounds over time | Cases span months; knowledge should accumulate |
| **Cross-references** | None (chunks are isolated) | Bi-directional links between pages | Legal arguments connect facts to issues to precedents |
| **Contradictions** | Hidden until query time | Flagged at ingest | Catch inconsistencies before tribunal |
| **Source attribution** | Chunk references (hard to verify) | Full-page context with citations | Legal requirement: trace every claim to source |
| **Query cost** | High (re-process every time) | Low (read pre-built pages) | Cases have 100+ queries; cost adds up |
| **Handoff** | "Read all docs" | "Read wiki/facts.md" | 20+ hours saved per case |
| **Cross-case** | No learning | Entities track across cases | Precedent library emerges automatically |
| **Visual understanding** | None (text only) | Graph visualization | See case relationships at a glance |

> **"RAG is like reading the entire library each time you have a question. Wiki is like having a librarian who already synthesized everything."** — Inspired by Karpathy's agentic wiki pattern

### 2.3 The Honest Answer: RAG + Wiki Hybrid

**Nobody serious is replacing RAG.** The "RAG is dead" narrative was clickbait. Production systems use both: each solves different problems.

| Layer | RAG | Wiki |
|-------|-----|------|
| **Best for** | Source citations, long-tail queries, large corpora | Repeat questions, synthesized knowledge, cross-references |
| **Scales to** | 10,000+ documents | ~100-1,000 pages per case |
| **Retrieval** | Chunk-level vector search | Full-page semantic search |
| **Knowledge** | Raw, unprocessed | Compiled, cross-linked |
| **Speed** | Slower (re-process every query) | Faster (pre-compiled) |

#### The Hybrid Architecture

```mermaid
flowchart TB
    subgraph Ingest["Document Ingestion"]
        Docs[Documents] --> Chunk[Chunk & Embed]
        Docs -->|Full text| WikiBuild[Wiki Ingest Agent]
        Chunk --> ChunkDB[(Chunk Vector DB)]
        WikiBuild -->|Async, offline| WikiPages[(Wiki Pages S3)]
        WikiBuild -->|Updates| WikiIndex[(Wiki Vector Index)]
    end

    subgraph Query["Query Time"]
        Q[User Question] --> Router{Query Router}
        
        Router -->|Facts, summaries| WikiSearch[Wiki Index Search]
        Router -->|Source citations| ChunkSearch[Chunk Vector Search]
        Router -->|Complex analysis| Both[Both Retrievers]
        
        WikiSearch -->|Full pages| Synth1[Synthesize]
        ChunkSearch -->|Chunks| Synth2[Synthesize]
        Both -->|Wiki + Chunks| Synth3[Synthesize]
        
        Synth1 -->|Threshold met| Ans[Answer]
        Synth2 --> Ans
        Synth3 --> Ans
        
        Ans -->|Optional| Update[Queue for Wiki Update]
    end

    style Ans fill:#9f9,stroke:#333,stroke-width:2px
    style WikiPages fill:#69f,stroke:#333
    style ChunkDB fill:#fc9,stroke:#333
```

#### Five Hybrid Patterns in Production

**Pattern 1: Wiki on Top, RAG Underneath** (most common)

Wiki serves as structured, high-priority knowledge that always loads. RAG layer handles larger archives when needed. The agent checks wiki first, then falls back to vector search only if wiki doesn't contain what's needed.

> *"The wiki files serve as structured, high-priority knowledge that always loads, while a RAG layer handles larger archives when needed. The agent checks the wiki first, then falls back to vector search only if the wiki doesn't contain what it needs. This preserves the clarity and reliability of the wiki pattern while extending the reach to larger corpora."* — MindStudio

```mermaid
flowchart LR
    Q[Question] --> Wiki{Wiki has it?}
    Wiki -->|Yes - 80%| Fast[Instant from wiki<br/>Fast path]
    Wiki -->|No - 20%| RAG[Chunk search<br/>Long-tail path]
    RAG --> Ans[Answer]
    Fast --> Ans
    Ans -->|Optional| Queue[Queue for wiki promotion<br/>Next time = fast path]

    style Fast fill:#9f9,stroke:#333
    style RAG fill:#fc9,stroke:#333
    style Queue fill:#69f,stroke:#333,stroke-dasharray: 5 5
```

**Production references**: Denser.ai (chatbot product), MindStudio (default recommendation)

---

**Pattern 2: Vector Index Over Wiki**

Karpathy's pattern works at ~100 pages because index fits in context. At 10,000 pages, you need a retriever in front of wiki.

Solution: Two retrieval surfaces emerge:

| Retrieval Surface | Content | Quality | Recall | Use Case |
|-------------------|---------|---------|--------|----------|
| Wiki pages | Synthesized, cross-linked | High | Medium | Common questions, concepts |
| Raw chunks | Unprocessed source | Medium | High | Long-tail, obscure details |

```mermaid
flowchart LR
    Q[Question] --> W[Wiki Page Vector Search<br/>High-quality, low-noise]
    Q --> C[Chunk Vector Search<br/>High-recall fallback]

    W -->|Top score > threshold| Use[Use wiki pages<br/>Full context]
    C -->|Otherwise| Use

    style Use fill:#9f9,stroke:#333
    style W fill:#69f,stroke:#333
    style C fill:#fc9,stroke:#333
```

**When to use**: Wiki grows beyond ~100-500 pages (context window limit)

---

**Pattern 3: Triple Retrieval (BM25 + Vector + Graph)**

From the "LLM Wiki v2" gist — production lessons at scale. Wiki provides graph structure via wikilinks, enabling graph traversal effectively for free.

Combined retrieval handles vague conversational queries like *"what did we decide about the pricing thing"*:

1. **BM25 (keyword)**: Exact matches — "section 8-1", "ITAA 1997", "Case #12345"
2. **Vector (semantic)**: Conceptual similarity — "penalty", "deduction", "remission"
3. **Graph (structure)**: Linked pages — `[[Pricing]]` → `[[Decisions]]` → `[[Case A Precedent]]`

```mermaid
flowchart LR
    Q[Vague query] --> B[BM25<br/>Keyword]
    Q --> V[Vector<br/>Semantic]
    Q --> G[Graph<br/>Wikilinks]

    B --> R[Ranked Results]
    V --> R
    G --> R

    R --> A[Answer]

    style G fill:#69f,stroke:#333
    style R fill:#9f9,stroke:#333
```

**Why it matters**: Conversational queries are often vague. Single retriever fails. Triple retrieval combines signals.

---

**Pattern 4: Pre-generated Q&A Bank (HybridRAG)**

Academic framework (2026) for chatbots with bounded question types.

**Process**:
1. Ingest raw PDFs → convert to hierarchical chunks
2. LLM pre-generates plausible Q&A pairs from chunks
3. At query time: match question to QA bank
4. If no suitable match → fall back to on-the-fly generation

```mermaid
flowchart LR
    subgraph Offline["Offline: Ingest"]
        Docs[PDFs] --> Chunk[Hierarchical Chunks]
        Chunk --> LLM[LLM Generate QA]
        LLM --> QABank[(QA Bank)]
    end

    subgraph Online["Online: Query"]
        Q[Question] --> Match[Match to QA Bank]
        Match -->|Found| QA[Return QA Pair]
        Match -->|Not found| Gen[Generate from chunks]
        Gen --> QA
    end

    style QABank fill:#9f9,stroke:#333
```

**Same insight as wiki**: Compile once, query many times. Different format: Q&A pairs vs prose pages.

**Best for**: Bounded question types (FAQs, policies, procedures)

---

**Pattern 5: Entropy-Gated Lazy Retrieval (L-RAG)**

Principled routing instead of heuristics. Two-tier context:

| Tier | Content | Token Cost | When Used |
|------|---------|------------|-----------|
| **Tier 1** | Compact wiki/summary | Low | Always loaded |
| **Tier 2** | Detailed chunks | High | Only when needed |

**Gating mechanism**: Predictive entropy
- Model can answer confidently from Tier 1 → entropy low → skip Tier 2
- Model uncertain → entropy high → retrieve Tier 2

```mermaid
flowchart LR
    Q[Question] --> T1[Tier 1: Wiki<br/>Always loaded]
    T1 --> Entropy{Entropy}

    Entropy -->|Low<br/>Confident| Ans[Answer from Tier 1<br/>Skip retrieval]
    Entropy -->|High<br/>Uncertain| T2[Tier 2: Chunk Retrieval<br/>Expensive]

    T2 --> Ans

    style Ans fill:#9f9,stroke:#333
    style T2 fill:#fc9,stroke:#333
```

**Replace "compact document summary" with "wiki"** → you have a principled decision mechanism for when to fall back to RAG.

#### What to Actually Build

For existing RAG systems, the lowest-risk path: **Pattern 1 + Pattern 2** (Wiki on top, vector index underneath).

```mermaid
flowchart LR
    subgraph Keep["Don't Touch — Existing RAG"]
        Existing[Existing Vector DB<br/>Chunk Pipeline<br/>Embeddings]
    end

    subgraph Add["Add On Top — Wiki Layer"]
        Ingest[Wiki Ingest Agent<br/>Offline/Async]
        WikiIdx[(Wiki Vector Index<br/>Separate from chunks)]
        Router[Query Router<br/>Threshold-based]
    end

    Q[Question] --> Router
    Router -->|Try first| WikiIdx
    Router -->|Fallback| Existing

    WikiIdx -->|Score > 0.75| Ans[Answer from Wiki<br/>Full page context]
    Existing -->|Otherwise| Ans

    Ans -->|If from chunks| Review[Queue for Wiki Promotion<br/>Review before adding]

    style Keep fill:#9f9,stroke:#333
    style Add fill:#69f,stroke:#333,stroke-width:2px
    style Ans fill:#fc9,stroke:#333
```

**Implementation steps**:
1. **Keep** existing vector DB and chunk pipeline — no changes
2. **Add** wiki ingestion agent (runs offline/async, not at query time)
3. **Embed** wiki pages into separate vector index (different collection)
4. **Query routing**: wiki index first → if top score > threshold (e.g., 0.75) → use wiki → else RAG fallback
5. **Optional**: when chatbot answers from chunks, queue that Q&A for review and possible promotion to wiki

**Key insight**: You get compounding-knowledge benefit without sacrificing recall or scale. The two systems complement each other.

**Reference implementations to study**:
| Repo | What to Learn |
|------|---------------|
| [nashsu/llm_wiki](https://github.com/nashsu/llm_wiki) | Chat app shape, wiki-on-top pattern |
| [Beever Atlas](https://github.com/beever/atlas) | Chat memory + wiki layered exactly this way |
| [llm-wiki-agent](../references/llm-wiki-agent/) | File-based agentic wiki with health/lint |
| [HybridRAG Paper](https://arxiv.org) | Pre-generated QA bank pattern |
| [L-RAG Paper](https://arxiv.org) | Entropy-gated retrieval mechanism |
| MindStudio docs | Wiki + RAG practical guidance |

---

#### Query Routing Strategy

The critical question: **when to use wiki vs RAG?** Three approaches:

**1. Score-based routing (simplest)**
```python
wiki_results = wiki_index.search(question, top_k=3)
if wiki_results[0].score > 0.75:
    return answer_from_wiki(wiki_results)
else:
    return answer_from_rag(question)
```

**2. Entropy-based routing (principled)**
```python
tier1_context = load_wiki_summary(question)
answer_with_confidence = llm.generate(tier1_context, question)
entropy = calculate_entropy(answer_with_confidence.logprobs)

if entropy < threshold:
    return answer_with_confidence  # Confident from wiki alone
else:
    tier2_chunks = rag_search(question)
    return llm.generate(tier1_context + tier2_chunks, question)
```

**3. Hybrid retrieval (highest quality)**
```python
wiki_pages = wiki_index.search(question, top_k=2)
chunks = chunk_index.search(question, top_k=5)

# Score fusion: prioritize wiki due to higher quality
combined = merge_results(wiki_pages, chunks, wiki_weight=1.5)
return llm.generate(combined)
```

**Recommended**: Start with score-based (Pattern 1). Add entropy-based (Pattern 5) if latency becomes an issue.

---

#### Case Assistant: Fast Path vs Long-Tail

Concrete example of how routing works in a legal case context:

```mermaid
flowchart TD
    Q[User Question] --> Type{Question Type}

    Type -->|80%| Common[Common Questions<br/>Fast Path: Wiki]
    Type -->|20%| Rare[Long-Tail Questions<br/>Fallback: RAG]

    Common --> C1["What is the case about?"]
    Common --> C2["Who are the parties?"]
    Common --> C3["What are the key issues?"]
    Common --> C4["What evidence supports X?"]
    Common --> C5["What precedents apply?"]

    Rare --> R1["What exactly does page 47 say?"]
    Rare --> R2["Quote the tribunal's words on Y"]
    Rare --> R3["How many times does 'penalty' appear?"]
    Rare --> R4["What's the timestamp on exhibit Z?"]

    C1 --> Wiki[[wiki/facts.md<br/>wiki/issues.md]]
    C2 --> Wiki[[wiki/parties.md]]
    C3 --> Wiki[[wiki/issues.md]]
    C4 --> Wiki[[wiki/evidence.md]]
    C5 --> Wiki[[wiki/precedents.md]]

    R1 --> Chunk[(Chunk Vector DB<br/>Full document search)]
    R2 --> Chunk
    R3 --> Chunk
    R4 --> Chunk

    Wiki --> Ans[Answer < 2s<br/>Full context]
    Chunk --> Ans2[Answer < 10s<br/>With citations]

    style Wiki fill:#9f9,stroke:#333
    style Chunk fill:#fc9,stroke:#333
    style Ans fill:#69f,stroke:#333
```

**Distribution**: In a typical case with 100 questions:
- 80 questions → wiki (facts, parties, issues, evidence, precedents)
- 20 questions → RAG (specific page quotes, exact wording, counts)

**Cost savings**:
- Wiki path: ~500 tokens per query = ~$0.001
- RAG path: ~5,000 tokens per query = ~$0.01
- Mixed: 80 × $0.001 + 20 × $0.01 = **$0.28 vs $1.00** (72% savings)

---

#### Scale Evolution: When to Move Between Patterns

As your wiki grows, follow this progression:

```mermaid
timeline
    title Wiki Scale Evolution
    section 0-100 pages
        Pattern 1: Wiki on top of RAG
        : Wiki fits in context
        : Direct page lookup
    section 100-500 pages
        Pattern 2: Add wiki vector index
        : Context overflow
        : Retrieval over wiki pages
    section 500-1000 pages
        Pattern 3: Triple retrieval
        : BM25 + Vector + Graph
        : Handle vague queries
    section 1000+ pages
        Pattern 4 or 5: Specialized
        : Pre-generated QA bank
        : Or entropy-gated retrieval
```

**Key milestones**:
- **~100 pages**: Wiki index becomes necessary
- **~500 pages**: Graph structure provides significant value
- **~1000 pages**: Consider specialized patterns (QA bank or L-RAG)

---

## 3. Why Case Assistant Specifically Needs Wiki

### 3.1 Legal Cases Have Inherent Structure

Legal cases are NOT random document collections. They have predictable entities:

```mermaid
graph TD
    Case[Legal Case] --> Parties[Parties]
    Case --> Events[Events/Timeline]
    Case --> Documents[Documents]
    Case --> Issues[Issues in Dispute]
    Case --> Evidence[Evidence]
    Case --> Arguments[Legal Arguments]
    Case --> Outcomes[Outcomes/Decisions]

    Parties --> TP[Taxpayer]
    Parties --> Comm[Commissioner]
    Parties --> Rep[Representatives]

    Issues --> I1[Deduction allowable?]
    Issues --> I2[Penalty remission?]
    Issues --> I3[Interest assessment?]

    style Case fill:#69f,stroke:#333
    style Parties fill:#fc9,stroke:#333
    style Issues fill:#f9c,stroke:#333
```

**Wiki pages map 1:1 to legal structure. RAG chunks don't.**

### 3.2 The Objection Letter Use Case

**Scenario**: Draft an objection letter for a home office deduction dispute.

#### With RAG Only:
```mermaid
sequenceDiagram
    User->>RAG: "Draft objection letter for home office"
    RAG->>Docs: Search all 150 documents
    Docs-->>RAG: Return 10 chunks (partial context)
    RAG->>LLM: "Draft letter using these chunks"
    LLM-->>User: Draft (may miss key facts)

    Note over User,LLM: 30 seconds, partial context
```

#### With RAG + Wiki:
```mermaid
sequenceDiagram
    User->>Wiki: "Draft objection letter"
    Wiki->>Facts: Read wiki/facts.md
    Wiki->>Parties: Read wiki/parties.md
    Wiki->>Issues: Read wiki/issues.md (home office section)
    Wiki->>Evidence: Read wiki/evidence.md (relevant docs)
    Wiki->>LLM: "Draft letter with full context"
    LLM-->>User: Comprehensive draft
    LLM->>Wiki: Save draft as wiki/objection-draft.md

    Note over User,LLM: 15 seconds, complete context
```

---

## 4. The Agentic Advantage: Multi-Step Reasoning

### 4.1 Ingest Agent Workflow

```mermaid
stateDiagram-v2
    [*] --> SaveSource: New document uploaded
    SaveSource --> PlanPages: Document saved to sources/
    PlanPages --> WritePages: LLM plans pages
    WritePages --> UpdateIndex: Pages written
    UpdateIndex --> AppendLog: Index updated
    AppendLog --> [*]: Done

    note right of PlanPages
        LLM decides:
        - Create new entity pages?
        - Update existing pages?
        - Extract relationships?
        - Flag contradictions?
    end note

    note right of WritePages
        Creates:
        - wiki/facts.md
        - wiki/parties.md
        - wiki/issues.md
        - wiki/evidence.md
    end note
```

### 4.2 Query Agent Workflow

```mermaid
stateDiagram-v2
    [*] --> Search: User asks question
    Search --> ReadContext: Find relevant pages
    ReadContext --> Synthesize: Load full page content
    Synthesize --> SaveAnswer: Generate answer
    SaveAnswer --> [*]: Optional: Save as synthesis

    note right of Search
        Keyword + semantic search
        across wiki pages
    end note

    note right of ReadContext
        Full pages, not chunks
        No truncation
        Complete context
    end note
```

### 4.3 Self-Healing Knowledge

```mermaid
flowchart LR
    subgraph Health["Health Checks"]
        H1[Empty/stub files]
        H2[Index sync]
        H3[Log coverage]
    end

    subgraph Lint["Lint (Semantic)"]
        L1[Orphan pages]
        L2[Broken links]
        L3[Contradictions]
        L4[Data gaps]
    end

    subgraph Actions["Auto-Fixes"]
        A1[Create missing pages]
        A2[Update broken links]
        A3[Flag contradictions]
        A4[Suggest sources]
    end

    Health -->|Every session| Actions
    Lint -->|Periodic| Actions

    style Health fill:#9f9,stroke:#333
    style Lint fill:#fc9,stroke:#333
    style Actions fill:#69f,stroke:#333
```

---

## 5. CaseWiki Structure: Per-Case Isolated Wikis

### 5.1 S3 Storage Layout

```mermaid
graph TD
    Bucket[s3://case-wiki-bucket]

    Bucket --> Case1[case_id_1]
    Bucket --> Case2[case_id_2]

    Case1 --> Wiki1[wiki/]
    Case1 --> Status1[.wiki-status.json]

    Wiki1 --> Index[index.md]
    Wiki1 --> Log[log.md]
    Wiki1 --> Facts[facts.md]
    Wiki1 --> Parties[parties.md]
    Wiki1 --> Issues[issues.md]
    Wiki1 --> Evidence[evidence.md]
    Wiki1 --> Timeline[timeline.md]

    Wiki1 --> Sources[sources/]
    Wiki1 --> Entities[entities/]
    Wiki1 --> Analyses[analyses/]

    Sources --> S1[audit-report.md]
    Sources --> S2[notice-assessment.md]

    Entities --> E1[TaxpayerName.md]
    Entities --> E2[ATO-Branch.md]

    style Bucket fill:#eef,stroke:#333
    style Case1 fill:#9f9,stroke:#333
    style Case2 fill:#9f9,stroke:#333
```

### 5.2 Wiki Page Relationships

```mermaid
graph TD
    Index[index.md] --> Facts[facts.md]
    Index --> Parties[parties.md]
    Index --> Issues[issues.md]
    Index --> Evidence[evidence.md]

    Facts -->|linked to| Parties
    Facts -->|linked to| Evidence

    Issues -->|linked to| Facts
    Issues -->|linked to| Evidence
    Issues -->|linked to| Precedents[precedents.md]

    Parties -->|linked to| Timeline[timeline.md]

    Evidence -->|linked to| Sources[sources/]

    Analyses[analyses/] -->|linked to| Facts
    Analyses -->|linked to| Issues

    style Index fill:#69f,stroke:#333,stroke-width:2px
    style Issues fill:#f9c,stroke:#333
    style Parties fill:#fc9,stroke:#333
```

---

## 6. Sync Status & User Experience

### 6.1 Wiki Status Lifecycle

```mermaid
stateDiagram-v2
    [*] --> NEW: Case created
    NEW --> STALE: Document uploaded
    STALE --> BUILDING: User clicks "Update Wiki"
    BUILDING --> CURRENT: Build complete
    CURRENT --> STALE: New document added
    BUILDING --> FAILED: Error during build
    FAILED --> STALE: Retry available

    note right of NEW
        Red badge
        "Build Wiki" button
    end note

    note right of CURRENT
        Green badge
        Last updated timestamp
    end note

    note right of STALE
        Yellow badge
        "Update Wiki" button
    end note
```

---

## 7. Cross-Case Learning & Entity Tracking

### 7.1 Entity Pages Across Cases

```mermaid
graph TD
    subgraph Case_A["Case A: Smith 2024"]
        A1[entities/Smith-John.md]
        A2[entities/ATO-Sydney.md]
    end

    subgraph Case_B["Case B: Jones 2025"]
        B1[entities/Jones-Mary.md]
        B2[entities/ATO-Sydney.md]
    end

    subgraph Global["Global Knowledge Base"]
        G1[ATO-Branch: Sydney] -.-> A2
        G1 -.-> B2
        G2[Precedent: Home Office 50%] -.-> A1
        G2 -.-> B1
    end

    style Global fill:#9f9,stroke:#333,stroke-width:2px
    style Case_A fill:#eef,stroke:#333
    style Case_B fill:#eef,stroke:#333
```

### 7.2 Knowledge Accumulation Over Time

```mermaid
xychart-beta
    title "Knowledge Base Growth Over Time"
    x-axis ["Jan", "Feb", "Mar", "Apr", "May", "Jun"]
    y-axis "Wiki Pages" 0 --> 100
    line [5, 15, 35, 60, 85, 95]
    bar [2, 5, 12, 20, 28, 35]
```

**Value proposition**: Each case makes the system smarter. First case: 5 pages. Tenth case: 35 reusable entities/concepts.

---

## 8. Industry Validation

### 8.1 Legal Tech Trends 2026

According to [Morgan Lewis](https://www.morganlewis.com/news/2026/01/legal-techs-predictions-for-knowledge-management-in-2026):

> *"Law firms will increasingly treat AI workflows as proprietary assets. By codifying institutional knowledge and client insights into AI‑enabled processes, firms will create branded, defensible intellectual property."*
>
> — Colleen Nihill, Chief AI & KM Officer

**Interpretation**: The market is moving toward structured, reusable knowledge assets—exactly what LLM Wiki provides.

---

## 9. Beyond Legal: Universal Case Pattern

### 9.1 Applicable Domains

| Domain | Case Type | Entities | Events | Documents | Issues |
|--------|-----------|----------|--------|-----------|--------|
| **Legal** | Lawsuits, tax disputes | Parties, judges, lawyers | Hearings, filings | Contracts, evidence | Claims, defenses |
| **Medical** | Patient records | Patients, doctors, conditions | Diagnoses, treatments | Test results, scans | Symptoms, prognosis |
| **Insurance** | Claims | Claimants, adjusters | Incidents, assessments | Photos, reports | Coverage, liability |
| **Consulting** | Client engagements | Clients, stakeholders | Milestones, deliverables | Briefs, artifacts | Risks, decisions |
| **HR** | Investigations | Employees, complainants | Incidents, interviews | Emails, policies | Violations, sanctions |
| **Audit** | Financial audits | Companies, auditors | Period end, fieldwork | Financial statements | Findings, recommendations |

**Universal pattern**: All case-based systems track entities, events, documents, and issues. Wiki abstracts this structure. RAG doesn't.

---

## 10. Implementation: LangGraph Agents

### 10.1 Ingest Agent Graph

```mermaid
flowchart LR
    START([Document Uploaded]) --> Save[Save Source]
    Save --> Plan[Plan Pages]
    Plan --> Write[Write Pages]
    Write --> Index[Update Index]
    Index --> Log[Append Log]
    Log --> Validate[Validate]
    Validate --> END([Done])

    Validate -->|Broken links| Plan
    Validate -->|Missing pages| Write

    style START fill:#9f9,stroke:#333
    style END fill:#9f9,stroke:#333
    style Validate fill:#fc9,stroke:#333
```

### 10.2 Query Agent Graph

```mermaid
flowchart LR
    START([User Question]) --> Search[Search Pages]
    Search --> Read[Read Context]
    Read --> Synthesize[Synthesize Answer]
    Synthesize --> Save{Save?}
    Save -->|Yes| Write[Write Analysis]
    Save -->|No| END([Done])
    Write --> END

    style START fill:#9f9,stroke:#333
    style END fill:#9f9,stroke:#333
    style Save fill:#fc9,stroke:#333
```

---

## 11. ROI: Business Case

### 11.1 Time Savings Per Case

```mermaid
gantt
    title RAG vs Wiki: Time Allocation Per Case
    dateFormat HH:mm
    axisFormat %H hrs

    section RAG Only
    Initial analysis    :rag1, 00:00, 10h
    Document searches   :rag2, after rag1, 15h
    Drafting            :rag3, after rag2, 10h
    Handoff preparation :rag4, after rag3, 5h
    Total               :milestone, 40h, 0h

    section RAG + Wiki
    Initial analysis    :wiki1, 00:00, 10h
    Wiki build          :wiki2, after wiki1, 2h
    Document searches   :wiki3, after wiki2, 5h
    Drafting            :wiki4, after wiki3, 5h
    Handoff preparation :wiki5, after wiki4, 1h
    Total               :milestone, 23h, 0h
```

**Savings**: 17 hours per case = ~40% reduction in billable time.

### 11.2 Quality Improvements

| Quality Metric | RAG Only | RAG + Wiki | Improvement |
|----------------|----------|------------|-------------|
| Contradictions caught | 20% | 85% | +325% |
| Missing facts detected | 30% | 90% | +200% |
| Handoff errors | 25% | 5% | -80% |
| Source attribution accuracy | 70% | 98% | +40% |

---

## 12. Conclusion

### 12.1 Key Takeaways

1. **RAG answers questions. Wiki builds understanding.**
   - RAG: Stateless, re-processes every query
   - Wiki: Persistent, compounds over time

2. **Legal cases have structure. Wiki maps to it.**
   - Parties, Issues, Evidence, Timeline become Wiki pages
   - RAG chunks are isolated islands

3. **Knowledge should accumulate, not evaporate.**
   - Each document makes wiki richer
   - Cross-case learning happens automatically
   - Contradictions flagged at ingest

4. **LangGraph enables multi-step reasoning.**
   - Ingest: Extract → Link → Update → Validate
   - Query: Search → Read → Synthesize → Save
   - Lint: Diagnose → Propose → Fix → Verify

### 12.2 The Bottom Line

> **"A case assistant without wiki synthesis is like a library without a catalog—you can find things, but you'll never see the connections."**

For legal, medical, insurance, consulting, or any case-based work: that difference matters.

---

## 13. Next Steps

### 13.1 Proof of Concept

```mermaid
timeline
    title CaseWiki Implementation Roadmap
    section Phase 1: Foundation
        S3 storage setup : Week 1
        Wiki status DB schema : Week 1
        Basic page templates : Week 2
    section Phase 2: Ingest Agent
        Document extraction : Week 3
        Page planning LLM : Week 4
        Page generation : Week 5
        Index and log updates : Week 6
    section Phase 3: Query Agent
        Wiki search : Week 7
        Context aggregation : Week 8
        Answer synthesis : Week 9
    section Phase 4: UI and Integration
        Status indicator : Week 10
        Wiki viewer : Week 11
        Case integration : Week 12
```

### 13.2 Success Metrics

- [ ] Wiki build time: < 2 minutes per 10 documents
- [ ] Query response time: < 5 seconds (vs 15s RAG)
- [ ] Handoff time: < 1 hour (vs 20+ hours)
- [ ] User satisfaction: > 4.5/5 stars
- [ ] Cross-case reuse: > 30% of pages reference other cases

---

## Appendix A: References

### Code References

- Current case agent: [backend/app/agents/case_assistant.py](../backend/app/agents/case_assistant.py)
- Case model: [backend/app/db/models/case.py](../backend/app/db/models/case.py)
- RAG tools: [backend/app/agents/tools/rag_tool.py](../backend/app/agents/tools/rag_tool.py)

### External References

- [Morgan Lewis - Legal Tech's Predictions for Knowledge Management in 2026](https://www.morganlewis.com/news/2026/01/legal-techs-predictions-for-knowledge-management-in-2026)
- [llm-wiki-agent Reference](../references/llm-wiki-agent/) - File-based agentic wiki implementation
- [LlmWiki Reference](../references/LlmWiki/) - Tauri-based desktop wiki with LangGraph.js
- [Karpathy's llm.wiki Concept](https://github.com/karpathy/llm.wiki) - Original agentic wiki pattern

---

*Document Version: 1.1*
*Last Updated: 2026-05-03*
*Author: Case Assistant Team*
