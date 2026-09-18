# Installing the InferIndex MCP server

Instructions for an AI agent (Cline or similar) installing this server for a user.

InferIndex is a **remote** MCP server: there is nothing to download, build or run locally, and **no API key is
needed**.

- **Endpoint**: `https://mcp.inferindex.dev/mcp`
- **Transport**: Streamable HTTP
- **Access**: read-only, no authentication

## Cline

Add this entry to `cline_mcp_settings.json`, merging it into any existing `mcpServers` object:

```json
{
  "mcpServers": {
    "inferindex": {
      "type": "streamableHttp",
      "url": "https://mcp.inferindex.dev/mcp",
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

Keep `"type": "streamableHttp"`: without it, Cline assumes the legacy SSE transport.

Optional: if the user has an InferIndex API key (higher rate limits), add
`"headers": { "x-api-key": "THEIR_KEY" }` next to `"url"`. Never invent a key; leave it out if the user doesn't
provide one.

## Check that it works

After installing, the server should expose five tools: `search_models`, `cheapest`, `compare_providers`,
`price_history` and `estimate_cost`. Try one of these prompts:

1. "Using InferIndex, which provider is currently the cheapest for DeepSeek V3.2?"
2. "Compare every provider for `openai/gpt-oss-120b` that processes requests in the EU."
3. "Estimate the monthly cost of 1,000 requests a day with 2,000 input and 500 output tokens on
   `deepseek/deepseek-v3.2`, and show the three cheapest offers."

Each tool result includes `api_url`, the equivalent call to the public API
(`https://api.inferindex.dev`), which can be opened to check the answer.

## Notes

- Prices are in USD per million tokens and come from the providers' public pricing. Always confirm on the
  provider's own page before committing to real spend.
- Full documentation: https://github.com/InferIndex/inferindex-docs
