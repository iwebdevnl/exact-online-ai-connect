# FAQ

**How do I connect in Claude web?**
Go to Settings → Connectors → Add custom connector, paste the base URL `https://mcp-exactonline.iwebdevelopment.com`, and authorize with your Exact Online account. See the [Claude client page](clients/claude.md) for details.

**My access expired, what do I do?**
Re-authorize through your client's connector settings (in Claude, open the connector and reconnect; in other clients, reconnect via `/mcp` or the equivalent reconnect action). You will be sent through the Exact Online login again and access resumes immediately.

**Which plan unlocks large-dataset analysis and trend overviews?**
The **Analytics** plan. You can also try it during the 30-day free trial that new organizations get automatically. See [Plans & quotas](tiers-and-quotas.md).

**Results look incomplete right after connecting, why?**
On the Analytics plan, the first sync of your administration's data can take a little while to complete. Ask again in a few minutes, results fill in as the sync finishes.

**Where is my data hosted?**
In the **EU**.

**Where can I find the terms?**
The authoritative terms are iWebDevelopment's [Algemene Voorwaarden (AV)](https://www.iwebdevelopment.com/hubfs/144641067/Files/algemene-voorwaarden.pdf) and the accompanying DPA (verwerkersovereenkomst). See [PRIVACY.md](https://github.com/iwebdevnl/exact-online-ai-connect/blob/main/PRIVACY.md) and [TRUST.md](https://github.com/iwebdevnl/exact-online-ai-connect/blob/main/TRUST.md) for a plain-language summary.

**Can the assistant post something to Exact Online without my approval?**
No. Every write, creating, updating, or deleting a record, is prepared as a draft first. Nothing is posted until you explicitly confirm it.

**Does the assistant see more than I can see in Exact Online myself?**
No. The connection runs under your own Exact Online permissions, so it cannot show or do anything that would otherwise be blocked for your user.
