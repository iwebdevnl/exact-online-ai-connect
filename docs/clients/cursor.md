# Cursor

Cursor connects to remote MCP servers by URL. Add Exact Online AI Connect through the settings UI, or edit the project config directly. There is no documented plan gate for MCP in Cursor.

## Steps

1. Go to **Settings → Tools & MCP → New MCP Server**, or edit `.cursor/mcp.json`:

   ```json
   { "mcpServers": { "exact-online": { "url": "https://mcp-exactonline.iwebdevelopment.com" } } }
   ```

2. On first connect, Cursor opens your browser for OAuth authorization.

You can also use the "Add to Cursor" badge in the [README](https://github.com/iwebdevnl/exact-online-ai-connect) for a one-click add.

See the [compatibility matrix](https://github.com/iwebdevnl/exact-online-ai-connect#client-compatibility) (last verified 2026-07).
