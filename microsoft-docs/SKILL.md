---
name: microsoft-docs
description: Look up current Microsoft Learn documentation (.NET, MSBuild, Windows/WinUI, Azure, M365, Power Platform, etc.) instead of relying on training data that may be stale or version-mismatched. Use when answering "how does X work", API/config/limit questions, or anything where the exact current behavior for a specific SDK/tool version matters.
---

# Microsoft Docs Lookup

Training data goes stale, especially for fast-moving surfaces like .NET, MSBuild, and Azure SDKs. When a question hinges on current, version-specific behavior, check Microsoft Learn rather than guessing from memory.

## How to look things up

1. **Prefer a Microsoft Learn MCP server if one is connected** (tools named like `microsoft_docs_search` / `microsoft_docs_fetch`, or similar). Check with ToolSearch if unsure. It returns live, current doc content.
2. **Otherwise use WebSearch/WebFetch against `learn.microsoft.com`** — search for the specific API/feature, then fetch the page for full detail when the summary isn't enough.

## When to reach for docs instead of guessing

- Understanding a concept — "how does X actually work"
- API signatures, config options, CLI flags — especially anything version-specific
- Limits, quotas, defaults
- Confirming something changed between versions (e.g. .NET 8 → .NET 10, an MSBuild SDK bump)
- Anything where being wrong would waste a build/test cycle rather than just being imprecise in conversation

Skip this for things you're confident about from well-established, stable APIs (e.g. basic C# syntax) — the point is to catch drift on things that actually change, not to look up everything.

## Query effectiveness

Be specific — vague queries return vague results.

```
❌ "MSBuild items"
✅ "MSBuild ItemGroup Include exclude wildcard syntax"

❌ ".NET HttpClient"
✅ ".NET 10 HttpClient SocketsHttpHandler connection pooling defaults"
```

Include version and platform when relevant (`.NET 10`, `EF Core 9`, `Windows App SDK 1.6`) — docs for the same API surface can differ meaningfully across versions.

## Fetching the full page

Search results are often excerpts. Fetch the full page when:
- It's a tutorial/quickstart and you need every step, not a fragment
- A config/CLI reference needs the complete option list
- The excerpt is visibly cut off mid-explanation
