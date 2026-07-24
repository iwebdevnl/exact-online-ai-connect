# Windsurf

Windsurf connects to remote MCP servers through its Tools settings, or by editing the raw config directly. Note the field name: Windsurf uses `serverUrl`, not `url`.

## Steps

1. Go to **Settings → Tools → Add Server**, or edit the raw config:

   ```json
   { "mcpServers": { "exact-online": { "serverUrl": "https://mcp-exactonline.iwebdevelopment.com" } } }
   ```

2. Authorize with your Exact Online account when Windsurf opens the browser.

Some Enterprise Windsurf plans may gate remote MCP servers by admin policy.

See the [compatibility matrix](https://github.com/iwebdevnl/exact-online-ai-connect#client-compatibility) (last verified 2026-07).
