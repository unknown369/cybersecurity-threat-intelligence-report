# Security Header Scanner

A lightweight Python CLI tool that audits any website's HTTP response headers against [OWASP's Secure Headers](https://owasp.org/www-project-secure-headers/) recommendations — flagging missing or misconfigured headers that leave a site exposed to XSS, clickjacking, MIME-sniffing, and protocol-downgrade attacks.

Built as part of exploring automated web security tooling, and later adapted into a module for a larger [Automated Web VAPT pipeline](#) (Nmap + OWASP ZAP + SQLMap).

## Why this matters

Missing security headers are one of the most common — and easiest to fix — web application weaknesses. They don't require a vulnerable endpoint to exploit; a browser's default trust in an unprotected response is enough. Yet they're frequently overlooked because checking them manually across a site (and staying current on best-practice values) is tedious. This tool automates that check in a few seconds.

## What it checks

| Header | Why it matters |
|---|---|
| `Content-Security-Policy` | Mitigates XSS and data-injection by restricting allowed script/style/frame sources |
| `Strict-Transport-Security` | Forces HTTPS, preventing protocol-downgrade and cookie-hijacking |
| `X-Content-Type-Options` | Blocks MIME-sniffing that can turn a non-executable response into an XSS vector |
| `X-Frame-Options` | Prevents clickjacking via iframe embedding |
| `Referrer-Policy` | Limits leakage of URLs/tokens via the `Referer` header |
| `Permissions-Policy` | Restricts browser feature access (camera, mic, geolocation) |
| `X-XSS-Protection` | Legacy browser XSS filter (still checked for older stacks) |

For each header, the tool reports whether it's **missing**, **present but weakly configured** (e.g. a CSP allowing `unsafe-inline`), or **OK**.

## Usage

```bash
pip install requests

# Basic scan
python3 security_header_scanner.py https://example.com

# Save the full report as JSON
python3 security_header_scanner.py https://example.com --json report.json
```

### Example output

```
Security Header Report for https://example.com/
HTTP Status: 200   Score: 3/7
------------------------------------------------------------
[OK]   Content-Security-Policy: default-src 'self'
[MISS] Strict-Transport-Security
        -> Forces browsers to use HTTPS, preventing protocol-downgrade and cookie-hijacking attacks.
[OK]   X-Content-Type-Options: nosniff
[OK]   X-Frame-Options: DENY
[MISS] Referrer-Policy
        -> Controls how much referrer information (which can leak URLs, tokens, or internal paths) is sent to other sites.
[MISS] Permissions-Policy
        -> Restricts which browser features/APIs (camera, mic, geolocation) the page and any embedded iframes can use.
[MISS] X-XSS-Protection
        -> Legacy browser XSS filter. Largely superseded by CSP, but its absence is still worth noting on older stacks.
```

## Pipeline integration

`header_scan_module.py` exposes the same checks as an importable function, returning findings in a standard `{source, finding, severity, detail, remediation}` schema — designed to slot into a larger vulnerability-scanning pipeline alongside tools like Nmap, OWASP ZAP, or SQLMap:

```python
from header_scan_module import run_header_scan

findings = []
findings += run_header_scan(target_url)
findings += run_nmap_scan(target_host)
findings += run_zap_scan(target_url)
```

Severity levels and finding names are aligned with OWASP ZAP's own alert conventions, so output merges cleanly into a combined report.

## Project structure

```
.
├── security_header_scanner.py   # Standalone CLI tool
├── header_scan_module.py        # Importable module for pipeline integration
└── README.md
```

## Limitations

- Checks header *presence and basic configuration* only — it does not validate that a CSP's allowed sources are actually safe beyond flagging `unsafe-inline`/`unsafe-eval`/wildcard.
- Single-request scan; does not crawl a site to check headers across multiple pages.
- No authentication support yet — scans the page as an anonymous visitor.

## Responsible use

Only scan domains you own or have explicit permission to test.

## License

MIT
