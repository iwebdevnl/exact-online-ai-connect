# Microsoft Copilot Studio

Microsoft 365 Copilot has no end-user "paste a URL in chat" option. Instead, integration goes through the agent path in Copilot Studio: a maker adds Exact Online AI Connect as an MCP server to an agent, then publishes that agent for use.

This is a heavier, maker-oriented setup compared to the other clients, and requires M365 Copilot licensing plus admin governance for the tenant.

## Steps

1. In Copilot Studio, create or open an agent.
2. Add Exact Online AI Connect as an MCP server (URL: `https://mcp-exactonline.iwebdevelopment.com/mcp`).
3. Publish the agent so end users in your tenant can use it.
4. Each user authorizes with their own Exact Online account on first use.

See the [compatibility matrix](https://github.com/iwebdevnl/exact-online-ai-connect#client-compatibility) (last verified 2026-07).
