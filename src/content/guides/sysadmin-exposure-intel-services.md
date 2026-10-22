---
title: 'Sysadmin Toolkit: Exposure, Threat Intel, and Connectivity'
description: 'How to use internet-wide visibility tools to audit attack surface, investigate incidents, and validate connectivity'
pubDate: 2026-10-22
updatedDate: 2026-10-22
---

This section covers tools that help you understand what the internet can see about your infrastructure.

## Exposure Discovery

### <a href="https://www.shodan.io/" target="_blank" rel="noopener noreferrer">Shodan</a>
Best for external attack surface inspection.

Use when:
- You want to find internet-exposed services and banners
- You need to verify that intended ports are the only visible ones

Caveat:
- Use responsibly and within legal/organizational boundaries.

### <a href="https://securitytrails.com/" target="_blank" rel="noopener noreferrer">SecurityTrails</a>
Best for historical DNS and asset relationship context.

Use when:
- You are investigating infrastructure drift or old exposure paths

Caveat:
- Free usage is limited; budget for deeper investigations.

## IP And ASN Context

### <a href="https://ipinfo.io/" target="_blank" rel="noopener noreferrer">IPinfo</a>
Best for fast IP ownership/ASN/geography context.

Use when:
- You are triaging suspicious traffic sources
- You need quick enrichment in incident notes

## Global Connectivity And Macro Signals

### <a href="https://radar.cloudflare.com/" target="_blank" rel="noopener noreferrer">Cloudflare Radar</a>
Best for internet trend and outage context.

Use when:
- You need external macro signals during widespread incidents
- You want to compare protocol/browser/traffic shifts over time

### <a href="https://radar.qrator.net/" target="_blank" rel="noopener noreferrer">radar.qrator.net</a>
Best for an additional viewpoint on connectivity/AS relationships.

Use when:
- You need a second external perspective for route/connectivity assumptions

## Threat Intel Adjacent Additions

### <a href="https://www.virustotal.com/" target="_blank" rel="noopener noreferrer">VirusTotal</a>
Use when:
- You need quick reputation checks for files, domains, IPs, or URLs

### <a href="https://eth0.me/" target="_blank" rel="noopener noreferrer">eth0.me</a> / <a href="https://ifconfig.me/" target="_blank" rel="noopener noreferrer">ifconfig.me</a> style endpoints
Use when:
- You need script-friendly external IP checks from hosts and CI jobs

### <a href="https://digwebinterface.com/" target="_blank" rel="noopener noreferrer">digwebinterface.com</a>, <a href="https://ping.eu/" target="_blank" rel="noopener noreferrer">ping.eu</a>, <a href="https://ping.pe/" target="_blank" rel="noopener noreferrer">ping.pe</a>
Use when:
- You need browser-based DNS and ping/traceroute checks without local tooling

## Operational Guardrails

- Never rely on a single external data source for high-impact decisions.
- Cross-check at least two services before escalating findings.
- Preserve screenshots or exported evidence in incident records.
- Re-validate "known good" assumptions after major provider or routing changes.

## Lightweight Weekly Routine

1. Check exposure footprint for critical domains/IP ranges.
2. Review newly observed certificates for key domains.
3. Validate DNS/mail health for primary services.
4. Spot-check one high-traffic endpoint performance from two regions.
5. Update your runbook with any changed tool behavior or limits.

Back to the main guide: [Sysadmin Toolkit: Essential Online Services (2026)](/guides/sysadmin-online-services/).
