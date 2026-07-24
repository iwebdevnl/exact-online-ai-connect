# Zed

Recent Zed builds support remote MCP servers natively. Older builds need a small stdio bridge instead.

## Recent builds

1. Go to **Settings → AI → MCP Servers** and add the URL: `https://mcp-exactonline.iwebdevelopment.com/mcp`
2. Authorize with your Exact Online account when prompted.

## Older builds

Bridge the remote server as a stdio context server with `mcp-remote`:

```
npx mcp-remote https://mcp-exactonline.iwebdevelopment.com/mcp
```

See the [compatibility matrix](https://github.com/iwebdevnl/exact-online-ai-connect#client-compatibility) (last verified 2026-07).
