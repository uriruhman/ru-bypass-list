# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A generator that produces `ip-list.json` — a list of IP ranges and domains for Russian online services, formatted for **AmneziaVPN split tunneling** (so traffic to these services bypasses the VPN and goes direct). The committed `ip-list.json` is regenerated weekly by GitHub Actions (`.github/workflows/update.yml`, Mondays 04:00 UTC) and committed only if changed.

## Commands

```bash
pip install -r requirements.txt   # or use the existing .venv/

python main.py                    # generate ip-list.json (amnezia format)
python main.py -o my-list.json    # custom output path
python main.py -f plain -o cidrs.txt   # plain text, one CIDR per line
python main.py -v                 # debug logging
python main.py -c config.yaml     # custom config path
```

There is no test suite, linter, or build step. The program is run end-to-end; it makes live network calls to RIPE/bgp.he.net/DNS.

## Architecture

The pipeline is config-driven. Adding or changing a service means editing `config.yaml`, not the code.

1. **`config.yaml`** — the source of truth. A list of `services`, each with a `name`, a list of `asn` numbers, and a list of `domains`. Grouped by category with comment headers. This is what you edit most often.

2. **`main.py`** — orchestrator. Loads config, then for each service: resolves every ASN to prefixes, resolves every domain to /32s, accumulates `{name, domains, networks}` results, and hands them to the output writer. Per-service errors are logged and counted, never fatal.

3. **`resolvers/asn.py`** — `resolve_asn(asn)` returns announced IPv4 prefixes. Tries **RIPE NCC RISstat API** first; falls back to scraping **bgp.he.net** only if RIPE returns empty. IPv6 prefixes are skipped. A module-global rate limiter enforces ≥1s between *all* outbound requests in this module.

4. **`resolvers/dns.py`** — `resolve_domains(domains)` returns DNS A-records as /32 networks. Always returns a list; never raises (per-domain failures are logged and skipped).

5. **`output/formatter.py`** — `write_output(...)` dispatches on format. Both formats run all collected networks through stdlib `collapse_addresses()` to dedupe/merge overlapping and adjacent subnets. AmneziaVPN format is a JSON array of `{"hostname": ..., "ip": ""}` with **domains listed first** (shown as readable labels in the app UI), then sorted aggregated CIDRs.

Key invariant: domains serve two roles — they are both resolved to IPs *and* emitted as standalone hostname entries in the amnezia output.

## Notes

- Everything is IPv4-only by design.
- The aggregation step means the final entry count is much smaller than the raw prefix count printed in the run summary.
- `.venv/` is committed to the working tree but Python caches are gitignored.
