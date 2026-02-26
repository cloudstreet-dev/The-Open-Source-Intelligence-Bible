# Chapter 19: Bounty Hunting and Vulnerability Research

## Learning Objectives

By the end of this chapter, you will be able to:
- Apply OSINT methodology to authorized security research and bug bounty programs
- Conduct systematic attack surface enumeration within defined scope boundaries
- Use passive and active reconnaissance techniques appropriately with program authorization
- Build recon workflows that produce actionable vulnerability leads
- Document security findings to professional disclosure standards
- Understand the legal and ethical framework for authorized security research

---

## 19.1 Security Research and OSINT

Bug bounty programs and authorized penetration testing are among the most clearly defined contexts for technical OSINT application. Program sponsors explicitly authorize security researchers to investigate their systems within defined scope boundaries, providing legal clarity that eliminates ambiguity about authorization.

OSINT reconnaissance is typically the first phase of security research — before any active testing, researchers build a comprehensive picture of the target's attack surface using only passive, publicly available data. This reconnaissance phase:

- Identifies all assets within scope (domains, subdomains, IP ranges, application endpoints)
- Reveals technology stack and software versions
- Discovers potentially forgotten or deprecated systems
- Maps organizational structure for social engineering context
- Identifies third-party integrations and dependencies

The better the reconnaissance, the more targeted and effective the subsequent active testing.

---

## 19.2 Legal and Authorization Framework

### Bug Bounty Authorization

Bug bounty programs provide explicit authorization for security testing within defined scope. This authorization is critical — without it, security research activities that look identical from a technical standpoint may be violations of the Computer Fraud and Abuse Act and equivalent international statutes.

**Reading and respecting scope**: Bug bounty scopes define what is in and out of scope. Testing out-of-scope systems is not authorized, even on the same organization's infrastructure. Common exclusions:
- Third-party vendor systems
- Customer data systems
- Physical security
- Social engineering against employees
- Denial of service testing

**Safe harbor provisions**: Well-structured programs include safe harbor language that commits not to pursue legal action for research conducted within scope and following disclosure guidelines.

**HackerOne and Bugcrowd programs**: These platforms host most major bug bounty programs with standardized legal frameworks and dispute resolution.

### Authorized Penetration Testing

Authorized pen testing is governed by a formal Statement of Work or Rules of Engagement document. Before beginning any active reconnaissance or testing, ensure:
- Written authorization from an authorized representative of the target organization
- Defined scope including explicit IP ranges, domains, and application names
- Defined testing windows if applicable
- Emergency contact procedures if critical issues are discovered
- Defined exclusions from scope

**Never assume authorization carries over**: Authorization for one engagement does not imply authorization for future engagements, expanded scope, or other systems of the same organization.

---

## 19.3 Attack Surface Enumeration Workflow

```python
import asyncio
import json
import requests
from datetime import datetime
from dataclasses import dataclass, field
from typing import List, Dict, Set, Optional
import subprocess

@dataclass
class ReconTarget:
    """Target organization for security research reconnaissance"""
    organization: str
    program_url: str
    in_scope_domains: List[str]
    in_scope_ip_ranges: List[str]
    out_of_scope: List[str]
    notes: str = ""

@dataclass
class AttackSurface:
    """Discovered attack surface for a target"""
    target: ReconTarget
    domains: Set[str] = field(default_factory=set)
    subdomains: Set[str] = field(default_factory=set)
    ip_addresses: Set[str] = field(default_factory=set)
    open_ports: Dict[str, List[int]] = field(default_factory=dict)
    web_technologies: Dict[str, Dict] = field(default_factory=dict)
    interesting_endpoints: List[str] = field(default_factory=list)
    certificates: List[Dict] = field(default_factory=list)
    github_exposures: List[Dict] = field(default_factory=list)
    discovery_timestamp: str = field(default_factory=lambda: datetime.now().isoformat())

class SecurityReconWorkflow:
    """
    Authorized security research reconnaissance workflow
    All methods are for use within authorized bug bounty or pen test scope only
    """

    def __init__(self, target: ReconTarget):
        self.target = target
        self.surface = AttackSurface(target=target)
        self.log = []

    def log_action(self, action: str, details: str):
        entry = {
            'timestamp': datetime.now().isoformat(),
            'action': action,
            'details': details
        }
        self.log.append(entry)
        print(f"[{entry['timestamp'][:19]}] {action}: {details}")

    def verify_scope(self, domain_or_ip: str) -> bool:
        """
        Verify an asset is within scope before any testing
        CRITICAL: Always verify scope before taking any action
        """
        # Check if in explicitly out-of-scope list
        for exclusion in self.target.out_of_scope:
            if exclusion.lower() in domain_or_ip.lower():
                return False

        # Check if domain is in scope
        for scope_domain in self.target.in_scope_domains:
            if domain_or_ip.endswith(scope_domain) or domain_or_ip == scope_domain:
                return True

        # Check if IP is in scope ranges
        try:
            import ipaddress
            for ip_range in self.target.in_scope_ip_ranges:
                network = ipaddress.ip_network(ip_range, strict=False)
                if ipaddress.ip_address(domain_or_ip) in network:
                    return True
        except ValueError:
            pass  # Not an IP address

        return False

    def passive_subdomain_discovery(self) -> Set[str]:
        """
        Discover subdomains using only passive/public data sources
        No active probing of target systems
        """
        discovered = set()

        for root_domain in self.target.in_scope_domains:
            self.log_action("passive_recon", f"Discovering subdomains for {root_domain}")

            # 1. Certificate Transparency
            try:
                response = requests.get(
                    f"https://crt.sh/?q=%.{root_domain}&output=json",
                    timeout=30
                )
                if response.status_code == 200:
                    for cert in response.json():
                        for name in cert.get('name_value', '').split('\n'):
                            name = name.strip().lstrip('*.')
                            if name.endswith(root_domain) and self.verify_scope(name):
                                discovered.add(name)

                self.log_action("ct_discovery", f"Found {len(discovered)} subdomains via CT logs")
            except Exception as e:
                self.log_action("ct_error", str(e))

            # 2. HackerTarget subdomain search (free API)
            try:
                response = requests.get(
                    f"https://api.hackertarget.com/hostsearch/?q={root_domain}",
                    timeout=30
                )
                for line in response.text.strip().split('\n'):
                    if ',' in line:
                        subdomain = line.split(',')[0].strip()
                        if subdomain.endswith(root_domain) and self.verify_scope(subdomain):
                            discovered.add(subdomain)
            except Exception as e:
                self.log_action("hackertarget_error", str(e))

            # 3. SecurityTrails (if API key available)
            # ... additional passive sources

        self.surface.subdomains.update(discovered)
        self.log_action("subdomain_discovery_complete", f"Total: {len(self.surface.subdomains)} unique subdomains")
        return discovered

    def technology_fingerprinting(self, urls: List[str]) -> Dict[str, Dict]:
        """
        Identify web technologies from public HTTP headers and responses
        Uses only HEAD/GET requests to discovered URLs
        """
        technologies = {}

        for url in urls[:50]:  # Limit initial batch
            # Verify scope before any request
            from urllib.parse import urlparse
            domain = urlparse(url).netloc
            if not self.verify_scope(domain):
                continue

            try:
                response = requests.head(url, timeout=10, allow_redirects=True,
                                        headers={'User-Agent': 'Security Research Bot'})

                tech = {
                    'url': response.url,
                    'status': response.status_code,
                    'server': response.headers.get('Server', ''),
                    'powered_by': response.headers.get('X-Powered-By', ''),
                    'framework': response.headers.get('X-Framework', ''),
                    'content_type': response.headers.get('Content-Type', ''),
                    'security_headers': {
                        'HSTS': bool(response.headers.get('Strict-Transport-Security')),
                        'CSP': bool(response.headers.get('Content-Security-Policy')),
                        'X-Frame-Options': bool(response.headers.get('X-Frame-Options')),
                        'X-Content-Type-Options': bool(response.headers.get('X-Content-Type-Options')),
                    }
                }

                technologies[url] = tech

            except Exception as e:
                technologies[url] = {'error': str(e)}

        self.surface.web_technologies.update(technologies)
        return technologies

    def github_code_search(self, organization: str) -> List[Dict]:
        """
        Search GitHub for code exposures related to the target
        Uses GitHub's public search API
        """
        exposures = []
        headers = {'Accept': 'application/vnd.github.v3+json'}

        # Queries that commonly reveal sensitive information
        queries = [
            f'org:{organization} password',
            f'org:{organization} secret_key',
            f'org:{organization} api_key',
            f'org:{organization} DB_PASSWORD',
            f'"{organization}" api_key site:github.com',
        ]

        for query in queries:
            try:
                response = requests.get(
                    'https://api.github.com/search/code',
                    params={'q': query, 'per_page': 10},
                    headers=headers,
                    timeout=10
                )

                if response.status_code == 200:
                    results = response.json()
                    for item in results.get('items', []):
                        exposures.append({
                            'query': query,
                            'repo': item.get('repository', {}).get('full_name'),
                            'file': item.get('path'),
                            'url': item.get('html_url'),
                            'note': 'Potential credential/secret exposure'
                        })

            except Exception as e:
                self.log_action("github_search_error", str(e))

        self.surface.github_exposures = exposures
        if exposures:
            self.log_action("github_exposure", f"Found {len(exposures)} potential code exposures")
        return exposures

    def generate_recon_report(self) -> str:
        """Generate a structured recon report for security research"""
        report = [
            f"# Reconnaissance Report",
            f"**Organization**: {self.target.organization}",
            f"**Program**: {self.target.program_url}",
            f"**Discovery Date**: {self.surface.discovery_timestamp[:10]}",
            "",
            "## Attack Surface Summary",
            f"- **Domains enumerated**: {len(self.surface.domains)}",
            f"- **Subdomains discovered**: {len(self.surface.subdomains)}",
            f"- **IP addresses identified**: {len(self.surface.ip_addresses)}",
            f"- **Potential code exposures**: {len(self.surface.github_exposures)}",
            "",
            "## Subdomains Discovered",
            "",
        ]

        for subdomain in sorted(self.surface.subdomains):
            tech = self.surface.web_technologies.get(f"https://{subdomain}", {})
            server = tech.get('server', 'Unknown')
            status = tech.get('status', 'Not probed')
            report.append(f"- `{subdomain}` — Status: {status}, Server: {server}")

        if self.surface.github_exposures:
            report.extend([
                "",
                "## Potential Code Exposures (GitHub)",
                "**Note**: Verify these are valid findings before reporting",
                ""
            ])
            for exposure in self.surface.github_exposures:
                report.append(f"- [{exposure['repo']}/{exposure['file']}]({exposure['url']})")
                report.append(f"  Query: `{exposure['query']}`")

        report.extend([
            "",
            "## Security Header Analysis",
            "",
            "| Endpoint | HSTS | CSP | X-Frame | X-Content-Type |",
            "|---|---|---|---|---|"
        ])

        for url, tech in list(self.surface.web_technologies.items())[:20]:
            headers = tech.get('security_headers', {})
            report.append(
                f"| {url[:50]} | {'✓' if headers.get('HSTS') else '✗'} | "
                f"{'✓' if headers.get('CSP') else '✗'} | "
                f"{'✓' if headers.get('X-Frame-Options') else '✗'} | "
                f"{'✓' if headers.get('X-Content-Type-Options') else '✗'} |"
            )

        report.extend([
            "",
            "## Reconnaissance Log",
            ""
        ])
        for entry in self.log:
            report.append(f"- [{entry['timestamp'][:19]}] **{entry['action']}**: {entry['details']}")

        return '\n'.join(report)
```

---

## 19.4 Responsible Disclosure

When vulnerability research uncovers a valid security issue, responsible disclosure is the professional and ethical standard:

**Disclosure process**:

1. **Validate the finding**: Confirm the vulnerability is real, within scope, and has genuine security impact
2. **Document the finding**: Create a clear, reproducible description with proof of concept
3. **Report to the program**: Submit through the bug bounty platform or organization's security contact
4. **Allow remediation time**: Standard coordinated disclosure gives organizations 90 days to remediate before public disclosure
5. **Follow up**: If no response after reasonable time, escalate through appropriate channels
6. **Public disclosure**: After remediation or expired timeline, consider responsible public disclosure to benefit the security community

**Writing effective bug reports**:

```markdown
## Vulnerability Report Template

### Title
[Clear, concise description: e.g., "SQL Injection in /api/users endpoint allowing authentication bypass"]

### Severity
[Critical/High/Medium/Low] — Based on CVSS score if applicable

### Description
[Clear description of the vulnerability and its security impact]

### Affected Component
- URL: https://target.example.com/affected/endpoint
- Parameter: [name of vulnerable parameter]
- Method: POST/GET/etc.

### Steps to Reproduce
1. [Step 1]
2. [Step 2]
3. [Observe the impact]

### Proof of Concept
[Code, request, or screenshot demonstrating the vulnerability]
[Ensure PoC is clearly limited to demonstrating impact without causing damage]

### Impact
[What can an attacker do with this vulnerability?]
[Practical business impact, not just theoretical risk]

### Recommended Fix
[Suggested remediation approach]

### Supporting Evidence
[Screenshots, HTTP request/response logs, code snippets]
```

---

## 19.5 AI Assistance in Security Research

```python
import anthropic

def ai_assisted_attack_surface_analysis(recon_data: dict) -> str:
    """
    Use AI to analyze recon findings and suggest areas of interest
    """
    client = anthropic.Anthropic()

    # Format recon data for AI analysis
    recon_text = json.dumps(recon_data, indent=2)

    prompt = f"""You are an experienced security researcher analyzing reconnaissance data for a bug bounty program.

RECONNAISSANCE DATA:
{recon_text[:4000]}

Based on this reconnaissance data, analyze:

1. HIGH-VALUE TARGETS
   - Which subdomains or endpoints are most likely to contain vulnerabilities?
   - What patterns suggest less-maintained or forgotten systems?
   - What technologies suggest known vulnerability classes?

2. ATTACK VECTORS TO INVESTIGATE
   - Based on the technology stack identified, what vulnerability classes should be prioritized?
   - Which security header gaps are most significant?
   - What test cases should be constructed?

3. INFORMATION LEAKAGE
   - What information in the recon data could be sensitive?
   - What code exposure findings warrant immediate investigation?

4. RECOMMENDED NEXT STEPS
   - Prioritized list of investigation targets
   - Specific test scenarios to try within scope

Focus on actionable, specific recommendations. Note that all testing must remain within the defined program scope."""

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{"role": "user", "content": prompt}]
    )

    return response.content[0].text
```

---

## Summary

Bug bounty and authorized security research represents one of the clearest authorized contexts for technical OSINT application. Within program scope boundaries, comprehensive attack surface enumeration using passive sources — certificate transparency, public DNS, GitHub code search, Shodan — provides the foundation for effective vulnerability discovery.

Scope compliance is non-negotiable: testing out-of-scope systems transforms legitimate research into illegal unauthorized access. Responsible disclosure standards protect both researchers and affected organizations.

AI assistance accelerates the analysis phase — synthesizing reconnaissance data, suggesting attack vectors, and prioritizing investigation targets — while human judgment guides testing approach and validates findings before disclosure.

---

## Common Mistakes and Pitfalls

- **Scope creep**: Testing systems outside the defined scope because they appear related to in-scope systems
- **Passive vs. active confusion**: Treating Shodan-indexed data as authorization for active exploitation
- **Severity inflation**: Reporting low-impact findings as critical to inflate bounty payouts — damages researcher credibility
- **Inadequate reproduction steps**: Submitting reports that program teams cannot reproduce
- **Missing business impact**: Describing what a vulnerability allows without explaining why it matters to the organization
- **Disclosure timeline violations**: Publishing findings before the agreed remediation period expires

---

## Further Reading

- HackerOne and Bugcrowd platform documentation — program policies and safe harbor provisions
- OWASP Testing Guide — comprehensive web application security testing methodology
- PortSwigger Web Security Academy — free learning platform for web security
- Nahamsec and Jason Haddix — bug bounty methodology resources
- Synack Red Team Blog — professional security research methodology
