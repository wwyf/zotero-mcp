---
name: zotero-mcp-guide
description: Guide for using Zotero library via MCP tools. Covers searching items (semantic, keyword, tag-based), retrieving full text and metadata, managing collections and tags, working with annotations and notes, and exporting citations. Use when users need to find papers, read paper content, view annotations/highlights, organize research, search their library, or work with their Zotero reference manager.
---

# Zotero MCP Guide

Access and manage Zotero libraries through MCP tools. All tools are prefixed with `mcp__zotero__`.

## Research Search Workflow (Best Practice)

For comprehensive research queries, use the **dual search + orchestrated analysis** pattern.

### Phase 1: Discovery (Main Context)

```
1. KEYWORD EXTRACTION from user's question:
   - Original keywords: user's exact terms
   - Generated keywords: synonyms, related terms, acronyms

2. DUAL SEARCH (run in parallel):
   - zotero_semantic_search(query="user question", limit=10)
   - zotero_search_items(query="keywords", qmode="everything", limit=10)

3. MERGE & DEDUPLICATE by item_key

4. GET METADATA for candidates:
   - zotero_get_item_metadata for top results (includes abstracts)

5. PRESENT CANDIDATES to user:
   - Show title, authors, year, abstract snippet
   - Let user select which papers to analyze
```

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
| **Dual search** | Both | Minimal |

| Context Location | Cost | Best For |
|------------------|------|----------|
| Main context | Preserved | Search results, user interaction |
| Orchestrator | Medium | Coordination, synthesis |
| Workers (parallel) | Isolated | Heavy fulltext processing |

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
