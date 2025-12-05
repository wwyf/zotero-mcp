---
name: zotero-mcp-guide
description: "MUST invoke this skill before using ANY mcp__zotero__ tools. This skill provides essential best practices for Zotero MCP usage: multi-strategy search patterns, context-efficient sub-agent workflows, and progressive retrieval. Triggers: (1) Any mention of Zotero, papers, citations, references, library (2) Any use of mcp__zotero__* tools (3) Research questions about academic papers (4) Finding, reading, or analyzing papers. Always follow this guide's patterns to avoid context bloat from long paper fulltexts."
---

# Zotero MCP Guide

Access and manage Zotero libraries through MCP tools. All tools are prefixed with `mcp__zotero__`.

## Research Search Workflow (Best Practice)

For comprehensive research queries, use the **dual search + orchestrated analysis** pattern.

### Phase 1: Discovery (Search Sub-Agent)

Spawn a search sub-agent to run all searches in parallel:

```
Task(
  subagent_type="general-purpose",
  prompt="""
  Search Zotero library for: "{USER_QUESTION}"

  STEP 1: Extract search terms
  - Original query: full user question
  - Segments: key phrases from query
  - Keywords: individual important terms
  - Generated: synonyms, related terms, acronyms

  STEP 2: Run ALL searches in PARALLEL

  a) Semantic search:
     - mcp__zotero__zotero_semantic_search(query="user question", limit=10)

  b) Keyword searches (3 tiers × 4 query types = 12 searches):

     TIER 1 - titleCreatorYear (title/creator/year):
     - mcp__zotero__zotero_search_items(query="original", qmode="titleCreatorYear")
     - mcp__zotero__zotero_search_items(query="segment", qmode="titleCreatorYear")
     - mcp__zotero__zotero_search_items(query="kw1 OR kw2", qmode="titleCreatorYear")
     - mcp__zotero__zotero_search_items(query="kw1 AND kw2", qmode="titleCreatorYear")

     TIER 2 - allfield (all metadata):
     - mcp__zotero__zotero_search_items(query="original", qmode="allfield")
     - mcp__zotero__zotero_search_items(query="segment", qmode="allfield")
     - mcp__zotero__zotero_search_items(query="kw1 OR kw2", qmode="allfield")
     - mcp__zotero__zotero_search_items(query="kw1 AND kw2", qmode="allfield")

     TIER 3 - everything (includes fulltext):
     - mcp__zotero__zotero_search_items(query="original", qmode="everything")
     - mcp__zotero__zotero_search_items(query="segment", qmode="everything")
     - mcp__zotero__zotero_search_items(query="kw1 OR kw2", qmode="everything")
     - mcp__zotero__zotero_search_items(query="kw1 AND kw2", qmode="everything")

  STEP 3: Merge & deduplicate by item_key

  STEP 4: Get metadata for top candidates
     - mcp__zotero__zotero_get_item_metadata for each unique item

  RETURN: List of candidates with:
  - item_key, title, authors, year, abstract snippet
  - Which search strategies found each paper (helps understand relevance)
  """,
  description="Search library: {SHORT_QUERY}"
)
```

### Main Context: Present to User

After search sub-agent returns:
```
1. Present candidates to user:
   - Show title, authors, year, abstract
   - Indicate how paper was found (semantic/keyword/both)

2. Let user select which papers to analyze
```

**Example Search Matrix:**

User query: "How does KV cache eviction work in LLM serving?"

| Tier | Query Type | Example |
|------|------------|---------|
| titleCreatorYear | Original | "KV cache eviction LLM serving" |
| titleCreatorYear | Segment | "KV cache eviction" |
| titleCreatorYear | OR | "cache OR eviction OR LRU" |
| titleCreatorYear | AND | "cache AND LLM" |
| allfield | ... | (same patterns) |
| everything | ... | (same patterns) |

### Phase 2: Analysis (Orchestrator Sub-Agent)

When user selects papers, spawn ONE orchestrator sub-agent:

```
Task(
  subagent_type="general-purpose",
  prompt="""
  You are an orchestrator. Analyze these papers for the user's question.

  Papers to analyze: {SELECTED_ITEM_KEYS}
  User's question: {USER_QUESTION}
  Keywords to search: {ORIGINAL_KEYWORDS}, {GENERATED_KEYWORDS}

  STEP 1: Spawn parallel worker sub-agents (max 3 papers each):
  - Worker 1: papers [{KEY1}, {KEY2}, {KEY3}]
  - Worker 2: papers [{KEY4}, {KEY5}, {KEY6}]
  (adjust based on number of papers)

  Each worker should:
  - Fetch fulltext using mcp__zotero__zotero_get_item_fulltext
  - Search for the keywords in the text
  - Extract relevant excerpts with context
  - Summarize findings per paper

  STEP 2: Collect all worker results

  STEP 3: Synthesize final answer:
  - Which papers are most relevant and why
  - Key findings organized by theme
  - Direct quotes with paper attribution
  - Gaps or contradictions between papers
  """,
  description="Orchestrate analysis of {N} papers"
)
```

### Why This Pattern?

| Search Type | Finds | Misses |
|-------------|-------|--------|
| Semantic only | Conceptually related | Exact terms, rare mentions |
| Keyword only | Exact matches | Synonyms, related concepts |
| **Multi-strategy** | Both + incremental depth | Minimal |

| Sub-Agent | Role | Context |
|-----------|------|---------|
| Search agent | 13 parallel searches, merge, dedupe | Isolated |
| Main context | Present candidates, user selection | Preserved |
| Orchestrator | Coordinate paper analysis | Medium |
| Workers | Fulltext processing (3 papers each) | Isolated |

## Quick Workflows

### Simple: Find a Specific Paper

```
zotero_search_items(query="author title year") → get item_key
zotero_get_item_metadata(item_key) → verify it's correct
```

### Simple: What is This Paper About?

```
zotero_get_item_metadata(item_key) → read abstract
zotero_get_annotations(item_key) → check user highlights
```

### Medium: Answer Question About One Paper

Spawn single sub-agent:
```
Task(
  subagent_type="general-purpose",
  prompt="""
  Fetch fulltext of "{ITEM_KEY}" using mcp__zotero__zotero_get_item_fulltext.
  Answer: "{USER_QUESTION}"
  Return: concise answer with key quotes and section references.
  """,
  description="Analyze: {PAPER_TITLE}"
)
```

### Complex: Research Question Across Library

Use full Research Search Workflow above.

## Handling Long Papers

Paper fulltext = 10-30k tokens. **Never load directly into main context.**

### Progressive Retrieval (stop when sufficient)

```
1. zotero_get_item_metadata  → Abstract (~200 tokens)
2. zotero_get_annotations    → User's highlights
3. zotero_get_notes          → User's summaries
4. Sub-agent analysis        → For specific questions
5. zotero_get_item_fulltext  → Last resort only
```

### Multi-Paper Analysis

| Papers | Strategy |
|--------|----------|
| 1 | Single sub-agent |
| 2-3 | Single sub-agent handles all |
| 4+ | Orchestrator + parallel workers (3 papers each) |

## Tool Reference

### Search Tools

**zotero_semantic_search** - AI-powered conceptual search
- `query`: Natural language question
- `limit`: Number of results (default 10)

**zotero_search_items** - Keyword search
- `query`: Search terms
- `qmode`: "titleCreatorYear" (default) or "everything" (includes fulltext)
- `limit`: Number of results

**zotero_search_by_tag** - Tag-based filtering
- `tag`: List of tags (ANDed; use `||` for OR, `-` for exclusion)

**zotero_advanced_search** - Multi-criteria search
- `conditions`: List of condition objects
- `join_mode`: "all" (AND) or "any" (OR)

### Content Retrieval

**zotero_get_item_metadata** - Bibliographic info + abstract (use first)
**zotero_get_item_fulltext** - Complete paper text (use via sub-agent)
**zotero_get_annotations** - PDF highlights and notes
**zotero_get_notes** - Standalone notes

### Organization

**zotero_get_collections** - List all collections
**zotero_get_collection_items** - Items in a collection
**zotero_get_tags** - All tags in library
**zotero_get_recent** - Recently added items
**zotero_get_item_children** - Attachments/notes for an item

### Modification

**zotero_create_note** - Add note to an item
**zotero_batch_update_tags** - Bulk tag operations

### Database

**zotero_update_search_database** - Refresh semantic search index
**zotero_get_search_database_status** - Check index status

## Tips

- Item keys: short alphanumeric (e.g., "TVSCINXC")
- If semantic results seem stale or missing recent papers, **suggest to user** they may want to update the search database (do not auto-run `zotero_update_search_database`)
- Always present search candidates to user before deep analysis
- Use parallel sub-agents for 4+ papers to save time
