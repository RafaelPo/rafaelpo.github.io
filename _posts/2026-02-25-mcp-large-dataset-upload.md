---
title: 'How to get large files to your MCP server without blowing up the context window (Part 1/2)'
date: 25/02/2026
permalink: /posts/mcp-large-dataset-upload/
tags:
  - MCP
  - Claude
---

*Part 1 of a two-part series on handling large datasets with MCP. This post covers getting data in. [Part 2](/posts/mcp-results-widget/) covers getting results out — including an interactive widget for Claude.ai and Claude Desktop.*

**tl;dr:** Inlining data in MCP tool calls eats the LLM's context window. Instead, use a presigned URL so Claude can upload the file directly to your server. The entire dataset enters the context as a 36-character artifact ID. We cover Claude Code (stdio and HTTP), Claude.ai, and Claude Desktop.

<video controls width="100%" style="max-width: 720px;">
  <source src="/images/mcp-large-dataset-upload/demo.mp4" type="video/mp4">
</video>

## The Inline Path (and Why It Breaks)

The simplest way to pass data to an MCP tool is to inline it. In [everyrow](https://everyrow.io) our `everyrow_agent` tool can accept data in this format:

```python
  data: list[dict[str, Any]] | None = Field(
      default=None,
      description="Inline data as a list of row objects.",
  )
```

So the LLM could provide:

```json
{
  "task": "Find the CEO of each company",
  "data": [
    {"company": "Stripe", "url": "stripe.com"},
    {"company": "Plaid", "url": "plaid.com"}
  ]
}
```

This works. Claude serialises the data, the server deserialises it, and processing begins. Inline data works well for small datasets but every row lives in the LLM's context window. A few hundred rows can easily consume tens of thousands of tokens, crowding out the AI's ability to reason about anything else. For anything beyond a small dataset, you need a different path.

## How Anthropic Handles Large Attachments

When a user attaches a file to a Claude.ai conversation, Anthropic routes it through one of two paths. Small text files, PDFs, and images are embedded directly into the context — the model can see every word, but it consumes tokens. Datasets and other code-processable files (CSV, Excel, JSON, XML) use a different mechanism: the [`container_upload`](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool) content block. Instead of embedding the file, Anthropic writes it to the filesystem of a **sandboxed code execution container**. Claude accesses it by running code inside the sandbox, such as `pd.read_csv('/path/to/data.csv')`. The file never enters the context window at all.

This is exactly the problem we need to solve, but we can't use Anthropic's sandbox filesystem because our server is a separate service. We need our own mechanism to get data from Claude's environment to our server without routing it through the context.

## The Presigned Upload Flow

Our solution is `everyrow_request_upload_url`. An MCP tool that gives Claude a signed URL to upload a file directly to our server. The flow has three steps:

### Step 1: Request a URL

Claude calls the tool with just a filename:

```json
{
  "filename": "companies.csv"
}
```

The server generates an HMAC-signed URL with a 5-minute TTL:

```json
{
  "upload_url": "https://mcp.everyrow.com/api/uploads/abc123?expires=1740000000&sig=...",
  "upload_id": "abc123",
  "expires_in": 300,
  "max_size_bytes": 52428800,
  "curl_command": "curl -X PUT -T companies.csv 'https://mcp.everyrow.com/api/uploads/abc123?expires=...&sig=...'"
}
```

### Step 2: Upload the file

**Claude.ai has access to a code execution sandbox.** The file the user attached is sitting on that sandbox's filesystem. So Claude can execute the curl command directly:

```bash
curl -X PUT -T /path/to/companies.csv \
  'https://mcp.everyrow.com/api/uploads/abc123?expires=1740000000&sig=...'
```

The file travels from the sandbox filesystem to our server over HTTPS without ever entering in the context window.

**Note**: The code execution sandbox blocks outbound network requests by default. The user must **whitelist your MCP server's URL** in their Claude.ai or Claude Desktop settings: **Settings → Capabilities → Code execution and file creation → Additional allowed domains** (e.g. `mcp-staging.everyrow.io/`). Without this, the upload will fail with a network error even though the presigned URL is valid.

### Step 3: Use the artifact

The upload endpoint validates the file (signature, size, content type, rate limits), parses the CSV, creates a server-side artifact, and returns an artifact ID:

```json
{
  "artifact_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "rows": 10000,
  "columns": ["company", "url", "industry", "founded"]
}
```

Now Claude passes that 36-character artifact ID instead of 10,000 rows of JSON to any processing tool.

## Security: Signing, SSRF, and Credential Handling

The presigned URL is signed with HMAC-SHA256, but signing alone isn't enough. The upload endpoint also enforces size limits, content-type validation, rate limiting, and single-use consumption.

We also support URL-based uploads — if the data lives at a public URL or a Google Sheet, the server fetches it directly. But URL fetching opens an SSRF attack surface: a user could provide `http://169.254.169.254/latest/meta-data/` and probe your cloud environment. We defend against this with DNS validation, IP blocklists, connection pinning, and redirect validation. Presigned uploads avoid this entirely as the server never makes outbound requests, it only *receives* data pushed to it.

## Common Pitfalls

- **Sandbox network whitelist not configured.** The most common failure. Claude will get the presigned URL successfully but the curl upload will fail with a network error. The fix is adding your domain under **Settings → Capabilities → Code execution and file creation → Additional allowed domains**.
- **Presigned URL expires mid-upload.** URLs have a 5-minute TTL. If Claude takes too long between requesting the URL and uploading, the signature will be rejected. The tool returns the expiry time so Claude can request a fresh URL if needed.
- **File too large.** The upload endpoint enforces a 50MB limit. For larger files, consider splitting or pre-processing before upload.

## Three Claude Clients

Each Claude client has different capabilities. The upload path adapts accordingly, but they all converge on the same `artifact_id` interface — the processing tools don't know or care how the data arrived.

### Claude Code (CLI)

Claude Code has direct filesystem access, so the flow is the simplest. It calls `everyrow_upload_data` with a local path:

```json
{
  "source": "/Users/me/data/companies.csv"
}
```

The MCP server reads the file directly (in stdio mode), creates an artifact via the [everyrow API](https://everyrow.io/docs), and returns the artifact ID. No presigned URLs, no sandbox, no intermediate steps. The filesystem *is* the upload mechanism.

### Claude.ai (Web) & Claude Desktop (Native App)

Neither client has a local filesystem or terminal. But both have the same **code execution sandbox** described [above](#how-anthropic-handles-large-attachments) — when the user attaches a CSV, Anthropic writes it to the sandbox filesystem via `container_upload`. Claude can then run curl inside the sandbox to push the file to our server:

1. User attaches CSV in browser
2. Anthropic writes file to sandbox filesystem (`container_upload`)
3. Claude calls `everyrow_request_upload_url`
4. Server returns HMAC-signed URL (5-min TTL)
5. Claude runs curl in sandbox — file uploaded to server
6. Server validates, parses CSV, creates artifact
7. Returns `artifact_id` (36 chars in context)

### The Convergence Point

| Client | Upload mechanism | Transport | File access |
|---|---|---|---|
| Claude Code (stdio) | Local file path | stdio | Direct filesystem |
| Claude Code (HTTP) | Presigned URL + curl in terminal | streamable-http | Direct filesystem |
| Claude.ai | Presigned URL + curl in sandbox | streamable-http | Code execution sandbox |
| Claude Desktop | Presigned URL + curl in sandbox | streamable-http | Code execution sandbox |

See the MCP docs on [stdio](https://modelcontextprotocol.io/docs/concepts/transports#standard-io-stdio) and [streamable-http](https://modelcontextprotocol.io/docs/concepts/transports#streamable-http) transports.

---

**The core pattern**: MCP tools work through the context window, but large datasets shouldn't live there. Use a side channel — local file reads, presigned URLs, URL fetching — to move data to the server, and pass a lightweight reference (artifact ID) through the protocol. The context window is for control messages, not bulk data.

If you want to find out more, visit [everyrow.io/docs](https://everyrow.io/docs).
