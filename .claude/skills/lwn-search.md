---
name: lwn-knowledge
description: MCP integration for LWN.net (Linux Weekly News) article search and retrieval. Use when the user wants to (1) fetch a specific LWN article by URL, or (2) search LWN articles by keywords with pagination and sorting options. Triggers on queries about Linux news, kernel development, open source articles, or explicit LWN references.
---
# LWN MCP Integration

Access LWN.net articles through the `lwn-knowledge` MCP server.

## Available Tools

### 1. `read_LWN_Article`

Fetch a specific LWN article by URL.

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `url` | string | Yes | Full LWN article URL (e.g., `https://lwn.net/Articles/123456/`) |

**Example:**

```json
{
  "name": "read_LWN_Article",
  "arguments": {
    "url": "https://lwn.net/Articles/998456/"
  }
}
```

**Returns:** Article content including title, author, date, and body text.

---

### 2. `search_LWN_Articles`

Search LWN articles by keywords with pagination and sorting.

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `term` | string | Yes | - | Search keywords (e.g., `"kernel memory management"`) |
| `offset` | integer | Yes | `-1` | Pagination offset for results, Use `-1` for first search |
| `orderby` | string | Yes | `"relevance"` | Sort order: `"date"` (newest first) or `"relevance"` (best match) |

**Example:**

```json
{
  "name": "search_LWN_Articles",
  "arguments": {
    "term": "eBPF security",
    "offset": -1,
    "orderby": "date"
  }
}
```

**Returns:** Array of article objects:

```json
{
  "articles": [
    {
      "title": "Article Title",
      "url": "https://lwn.net/Articles/123456/",
      "summary": "Brief description of the article content..."
    }
  ]
}
```

## Usage Patterns

### Fetch specific article

```
User: "Read this LWN article: https://lwn.net/Articles/998456/"
Action: Call lwn_fetch_article with the URL
```

### Search by topic

```
User: "Find recent LWN articles about Rust in the kernel"
Action: Call lwn_search with terms="Rust kernel", sort_by="date"
```

### Paginate through results

```
User: "Show me more results"
Action: Call lwn_search with same terms, offset incremented by result count.
offset -1 -> first page
offset 0 -> from item 26
offset 25 -> from item 51
```

## Notes

- The MCP server must be running and configured in Claude Code
- Search results return summaries; use `read_LWN_Article` for full content
- Use `orderby="date"` for recent developments, `orderby="relevance"` for comprehensive topic research
