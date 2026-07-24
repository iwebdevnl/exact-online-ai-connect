# Privacy

This page is an informational summary of how Exact Online AI Connect handles data.
It is not a legal document. The authoritative, binding terms, and the current
list of subprocessors, live in iWebDevelopment's [Algemene Voorwaarden (AV)](https://www.iwebdevelopment.com/hubfs/144641067/Files/algemene-voorwaarden.pdf)
and the accompanying DPA (verwerkersovereenkomst). Where this page and the AV/DPA
differ, the AV/DPA governs. The DPA is available on request via
support@iwebdevelopment.com.

## Transient versus persisted data

**Transient data** is the content of a single request and its response, for
example the records needed to answer one question you asked. It is used to
produce that answer and is not kept afterwards as a stored copy.

**Persisted data** is what we actually keep in our systems. On all plans,
that is:

- Your OAuth tokens, encrypted at rest (AES-256-GCM in production), so we can
  keep your connection to Exact Online authorized.
- Your session, so the assistant knows which administration you are working
  in.
- Usage metrics, so quotas and plan limits can be enforced.
- An audit log of actions taken through the connection, so activity can be
  reviewed later.

None of this is kept indefinitely. It is pruned on a retention schedule: the
audit log for roughly 30 days, and usage metrics for roughly 180 days.

## The analysis copy (Analytics plan only)

On the **Analytics** plan, and during the 30-day trial, a synced copy of your
administration's data is kept to support large-dataset analysis and trend
overviews. On the **Essentials** plan there is no such copy: every request
passes straight through to Exact Online and back, nothing is synced or stored
in bulk.

## Where things run

Core infrastructure runs in the **EU**. We use a small number of infrastructure
and service providers to operate the connection and to send account
notifications. The authoritative, current list of subprocessors is maintained in
the AV/DPA, not on this page.

## Your AI assistant is a separate relationship

Exact Online AI Connect is the connector between your Exact Online administration
and your AI assistant. The assistant itself (for example Claude or ChatGPT)
is operated by its own vendor, under your own account and agreement with
that vendor. Connecting Exact Online AI Connect does not change or extend that
relationship, and does not make us a party to it.

## Data residency

Your data is hosted in the **EU**.
