# VS Code (GitHub Copilot)

GitHub Copilot in agent mode (VS Code 1.101 or later) can call remote MCP servers directly. Works on all Copilot plans, including the free plan. On Copilot Business and Enterprise, an admin must enable the "MCP servers in Copilot" policy, which is off by default.

## Steps

1. In your project (or user settings), create `.vscode/mcp.json`:

   ```json
   { "servers": { "exact-online": { "type": "http", "url": "https://mcp-exactonline.iwebdevelopment.com/mcp" } } }
   ```

2. VS Code prompts you to start the server; authorize with your Exact Online account in the browser.

See the [compatibility matrix](https://github.com/iwebdevnl/exact-online-ai-connect#client-compatibility) (last verified 2026-07).
