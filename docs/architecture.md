# How the connection works

```mermaid
flowchart LR
  A[AI client<br/>Claude / ChatGPT / IDE] -->|MCP over HTTPS| B[Exact Online AI Connect<br/>hosted MCP server · EU]
  B -->|OAuth 2.x| C[Exact Online API]
  B -.->|Analytics plan only| D[(Analysis copy<br/>EU)]
```

You authorize access in your browser once (OAuth). No tokens are pasted into
your client config. Every write is prepared as a draft you approve before it is
posted. The authoritative terms are in iWeb's AV/DPA.
