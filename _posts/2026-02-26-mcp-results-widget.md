---
title: 'How to return large results from your MCP server without flooding the context window (Part 2/2)'
date: 26/02/2026
permalink: /posts/mcp-results-widget/
tags:
  - MCP
  - Claude
---

*Part 2 of a two-part series on handling large datasets with MCP. [Part 1](/posts/mcp-large-dataset-upload/) covers getting data in. This post covers getting results out.*

**tl;dr:** Large structured data doesn't belong in the LLM's context window. Instead of dumping thousands of rows into the tool response: 1) give the user an interactive widget rendered directly in Claude.ai and Claude Desktop to inspect the full dataset, 2) keep the model's context lean with a small preview, and 3) provide a download URL so the model can pull the data into its sandbox and process it with real code.

<video controls width="100%" style="max-width: 720px;">
  <source src="/images/mcp-results-widget/demo.mp4" type="video/mp4">
</video>

---

You run an MCP tool. It completes. 5,000 rows of results are ready. You return them in the tool response, and watch as Claude's next reply starts with "Here are your results:" followed by a wall of JSON that consumes half the context window. The user scrolls past it. Claude can barely reason about anything else in the conversation.

## The naive approach (and why it breaks)

The simplest way to return results from an MCP tool is to serialize everything into the tool response:

```python
@mcp.tool()
async def my_tool() -> list[TextContent]:
    df = run_query()  # 5,000 rows
    return [TextContent(
        type="text",
        text=json.dumps(df.to_dict(orient="records"))
    )]
```

This works for 20 rows. At 200 rows it starts to hurt. At 5,000 rows you've consumed the entire context window with data the model can barely parse. LLMs aren't spreadsheets, and dumping thousands of rows into the context doesn't help them reason better. Worse, the user can't sort, filter, or search the results. They're locked behind whatever summary the model decides to write.

## The core idea: split the audience

Rows in the context window are inert text. Rows in the sandbox are executable data. Claude.ai has a code execution sandbox, so if the data lives as a file there, the model can run pandas, compute aggregates, and build charts. Data should be processed with proper tools, not treated as text.

The key insight is that there are three consumers of the same result set, and each needs a different format:

- **The model's context** needs a small sample with column names, a few representative rows, and metadata (total count, pagination hints). This is enough to answer "what are the columns?", "show me the top results", or "are there any errors?".
- **The user** needs the full dataset in an interactive format with sortable columns, per-column filters, global search, CSV download, clickable links.
- **The model's sandbox** needs the full dataset as a file it can process with real code. A download URL lets the model pull the data with a single `curl` and immediately work with it:

```bash
curl -o results.csv "https://mcp-server/api/results/{task_id}/download?token=..."
```

```python
df = pd.read_csv("results.csv")
df.groupby("country")["revenue"].sum().sort_values(ascending=False)
```

**Serving all three from one response is the mistake.** Instead, return a compact text summary for the context, a rich widget for the user, and a download URL for the sandbox.

## The architecture

```python
@mcp.tool()
async def my_tool() -> list[TextContent]:
    df = run_query()  # 5,000 rows
    preview = df.head(50).to_dict(orient="records")

    # TextContent #1: drives the interactive widget
    widget_json = json.dumps({
        "preview": preview,
        "total": len(df),
        "csv_url": build_download_url(task_id),
        "fetch_full_results": True,
    })

    # TextContent #2: what the model actually reads
    summary = (
        f"Results: {len(df)} rows, {len(df.columns)} columns. "
        f"Showing first 50. Full dataset visible in widget above."
    )

    return [
        TextContent(type="text", text=widget_json),
        TextContent(type="text", text=summary),
    ]
```

### What the model sees

The second TextContent is a plain text summary. It contains:

- **Token-bounded preview:** row count, column names, and a page of inline data controlled by `page_size`. If the rows contain long text fields, the server silently reduces the page to fit a token budget.
- **Pagination hints:** so the model can fetch more pages if the user asks ("show me the next 50").
- **Download URL:** the CSV endpoint for the sandbox, so the model can `curl` the full dataset when it needs to do real analysis.

### What the user sees

The first TextContent is a JSON blob that drives an interactive widget embedded directly in the conversation. Both TextContent items enter the model's context, but the client (Claude.ai) additionally renders the JSON through a registered widget. How? The tool is registered with a `resourceUri` pointing to an HTML resource hosted by the MCP server. Claude.ai fetches that HTML once at connection time and renders it in a sandboxed iframe. The HTML is a self-contained JavaScript application that listens for tool results via the [MCP ext-apps SDK](https://www.npmjs.com/package/@modelcontextprotocol/ext-apps), processes the JSON payload, and renders the table.

On first load, the widget shows the small preview baked into the JSON while it fetches the full dataset in the background. Once loaded, the user gets sorting, filtering, global search, CSV/JSON export, all running client-side with no server round-trips.

## How the widget gets its data

The widget needs the full dataset, but the MCP tool response only contains a small preview. The rest comes from a **dedicated REST endpoint** on the MCP server. The flow:

1. **Tool call completes.** The MCP response includes a small preview and a `poll_token` for the widget.
2. **Widget renders the preview** immediately.
3. **Widget mints a download token** by calling `GET /api/results/{task_id}/download-token` with the poll token as a Bearer header. The server returns a fresh single-use token.
4. **Widget fetches full data** via `GET /api/results/{task_id}/download?token={download_token}&format=json`. The server consumes the token and returns the full dataset.
5. **Widget replaces the preview** with the complete dataset. Sorting, filtering, and search now operate on all rows.

MCP handles the tool call; plain HTTP handles the bulk data transfer. The widget requests JSON for rendering; users can hit the same endpoint without `format=json` to download a CSV. Download tokens are single-use and short-lived, so stale URLs can't leak data.

## What about Claude Code?

The widget pattern targets Claude.ai and Claude Desktop. They both have a rendering layer that can display interactive HTML via [MCP Apps](https://modelcontextprotocol.io/docs/extensions/apps). Claude Code is a terminal. It can't render widgets, and it ignores `structuredContent` in tool responses.

But the core problem of keeping large data out of the context still applies. Claude Code solves it differently: it has direct filesystem access and can run `curl` locally. So instead of three lanes, you get two:

1. **Model context** gets the compact text summary (same as Claude.ai).
2. **Local execution** replaces both the widget and the sandbox. Claude Code downloads the CSV and processes it with pandas on your machine.

In the case of Claude Code the widget JSON blob becomes dead weight, it enters the context as plain text and wastes tokens. To avoid this, check who's connecting during the MCP `initialize` handshake. Clients that support widgets advertise it via `capabilities.extensions["io.modelcontextprotocol/ui"]`. Alternatively, you can check `clientInfo.name`, Claude Code identifies as `"claude-code"` while Claude.ai and Claude Desktop both identify as `"claude-ai"`. If the client doesn't support widgets, return only the text summary and download URL.

## When you don't need this

If your result set is under 100 rows and mostly numeric, dumping inline is fine. This pattern matters when:

- **Row count > 500:** context cost dominates and the model can't reason across all of them anyway.
- **Wide schemas or long text fields:** a single row can burn hundreds of tokens.
- **Follow-up analysis is expected:** the user will want to group, filter, or chart the data, which means the sandbox needs the file.

## Putting it together

The pattern for returning large results from MCP tools:

1. **Don't dump everything into the context.** The LLM can't usefully process thousands of rows, and you're burning tokens for nothing.
2. **Split the response.** A compact text summary for the model. A rich widget for the user. A download URL for the model's sandbox.
3. **Serve the full data over plain HTTP**, separate from MCP. Both the widget and the model's sandbox fetch via REST.

The context window is for reasoning, not storage. The sandbox is for processing. The widget is for the user. Keep each in its lane.

If you want to find out more, visit [everyrow.io/docs](https://everyrow.io/docs).
