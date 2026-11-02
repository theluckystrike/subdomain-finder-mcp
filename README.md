# subdomain-finder-mcp

Passive subdomain finder for AI assistants: live certificate transparency lookups (certspotter, crt.sh) with DNS checks over Cloudflare DNS over HTTPS.

This is a Model Context Protocol (MCP) server for subdomain enumeration in bug bounty and security recon. It reads public certificate transparency (CT) logs, deduplicates the names, handles wildcard certificates, and reports first seen, last seen and issuer for each host. It can then resolve the hosts to mark which are live and which have dangling CNAMEs. It needs no API key.

## Passive and read-only

- It only reads public CT logs and public DNS. It sends nothing to the target's servers.
- No port scanning, no HTTP probing, no brute-force wordlists.
- Use it on domains you own or are authorised to test, for example inside a bug bounty program's scope.

## Tools

| Tool | What it returns |
| --- | --- |
| `find_subdomains` | Deduplicated hostnames for a domain from CT logs, with first seen, last seen, valid until, certificate count, issuers and a wildcard flag. Optional DNS check of up to 40 names. |
| `resolve_subdomains` | A, AAAA and CNAME chain for up to 40 hosts. Marks live, NXDOMAIN, no records and dangling CNAMEs, names cloud services behind CNAMEs, and detects wildcard DNS. |
| `get_certificate_history` | Certificates for one host: issuer, validity dates, every SAN, revocation status and SHA-256, plus first and latest issuance and the issuers used. |
| `summarize_attack_surface` | The domain's CT hostnames grouped into development and test, staging, admin, API, auth, mail, remote access, storage and legacy, with naming clusters, issuers and the newest names. |

Every result has structured JSON, a one-paragraph summary, `as_of`, `source` (the CT or DNS query URL) and `more` (the matching tool on [bugbountyreality.com](https://bugbountyreality.com/?utm_source=mcp&utm_medium=subdomain-finder-mcp)).

## What it knows that your model does not

- A domain's current subdomains. New hosts appear in CT logs within minutes of getting a certificate; a model's training data is months old and never had most of them.
- Which hosts resolve right now, which CNAMEs point at a deleted cloud resource, and whether the zone has wildcard DNS.
- Certificate history: when each name first got a certificate, which CAs issued it, and when the current one expires.
- How to read CT data cleanly: precertificate and certificate rows merged, e-mail SANs and out-of-scope names dropped, `*.` entries reported as wildcards instead of hosts.

## Install

### Hosted (Claude.ai custom connector, ChatGPT developer mode)

Paste this URL. No token, no sign-up.

```
https://subdomain-finder-mcp.lipmichal.workers.dev/mcp
```

### Claude Code

```
claude mcp add --transport http subdomain-finder https://subdomain-finder-mcp.lipmichal.workers.dev/mcp
```

Or run it locally over stdio:

```
claude mcp add subdomain-finder -- npx -y github:theluckystrike/subdomain-finder-mcp
```

### Claude Desktop

Add to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "subdomain-finder": { "command": "npx", "args": ["-y", "github:theluckystrike/subdomain-finder-mcp"] }
  }
}
```

### Cursor

Add to `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "subdomain-finder": { "url": "https://subdomain-finder-mcp.lipmichal.workers.dev/mcp" }
  }
}
```

### VS Code

Add to `.vscode/mcp.json`:

```json
{
  "servers": {
    "subdomain-finder": { "type": "http", "url": "https://subdomain-finder-mcp.lipmichal.workers.dev/mcp" }
  }
}
```

Node 20 or newer is needed for the local stdio version. The local version uses your own IP for the CT and DNS lookups, so it has its own rate-limit allowance.

## Example prompts

- "Find subdomains of hackerone.com from certificate transparency and tell me which are live."
- "Which of these hosts have dangling CNAMEs: assets.example.com, docs.example.com, status.example.com?"
- "Show the certificate history for api.example.com. Did they change CA this year?"
- "Summarise the attack surface of example.com. Any staging, admin or VPN hosts?"
- "Include expired certificates and list hosts under example.com that first appeared before 2020."

## Data sources

All public and keyless. Full list with fetch dates in [DATA_SOURCES.md](DATA_SOURCES.md).

- SSLMate certspotter CT Search API (api.certspotter.com), unexpired certificates. Primary.
- crt.sh, including expired certificates. Fallback, and used when `include_expired` is true.
- Cloudflare DNS over HTTPS (cloudflare-dns.com/dns-query).

## Limitations

- Certificate transparency only shows names that got a publicly trusted certificate. Hosts behind a wildcard certificate (`*.example.com`) or with no TLS never appear by name. Pair it with other passive sources for full coverage.
- certspotter's keyless tier is rate limited per IP (`x-ratelimit-limit: 10` observed). Results are cached for an hour, the tool reads at most 5 pages per call, and it falls back to crt.sh when certspotter throttles. If both throttle, the error says to wait a few minutes.
- crt.sh is often slow or overloaded, and two calls minutes apart can return different rows (its database replicas lag). Calls time out after 25 seconds, retry once, then fail with a clear message.
- Very large domains return the first pages only; the result says when it was truncated.
- DNS checks are capped at 40 hosts per call. A "dangling CNAME" is a lead to verify inside an authorised scope, not proof of a vulnerability.

## More

The web versions of these tools, a subdomain takeover checker and a manual recon checklist are on [bugbountyreality.com](https://bugbountyreality.com/tools/subdomain-finder-real/?utm_source=mcp&utm_medium=subdomain-finder-mcp). See also the [certificate transparency lookup](https://bugbountyreality.com/tools/certificate-transparency-lookup/?utm_source=mcp&utm_medium=subdomain-finder-mcp) and the [subdomain enumeration checklist](https://bugbountyreality.com/tools/subdomain-enumeration-checklist-manual-recon-guide?utm_source=mcp&utm_medium=subdomain-finder-mcp).

Part of the Zovo MCP collection: https://mcp.zovo.one

## Development

```
npm install
npm test            # builds, then runs the fixture tests against saved CT and DNS payloads
node scripts/smoke-stdio.mjs   # live stdio check, one real call per tool
npx wrangler deploy            # Cloudflare Worker, Streamable HTTP at /mcp
```

## License

MIT
