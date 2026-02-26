# Chapter 6: Domain and Network Reconnaissance

## Learning Objectives

By the end of this chapter, you will be able to:
- Conduct systematic domain and DNS reconnaissance
- Map organizational network infrastructure from public sources
- Use passive and active reconnaissance techniques appropriately
- Extract intelligence from certificate transparency logs
- Analyze historical DNS and WHOIS data
- Use Shodan and similar tools for internet infrastructure analysis
- Build automated recon pipelines for repeatable workflows

---

## 6.1 Why Network Reconnaissance Matters

Every organization that maintains a web presence, email infrastructure, or cloud presence generates a significant public footprint in the Domain Name System, IP address allocation records, certificate authorities, and internet-wide scanning databases. This footprint is often more revealing than organizations realize.

Network reconnaissance — the systematic mapping of an organization's internet infrastructure — serves several legitimate OSINT purposes:

- **Attack surface analysis**: Understanding what an organization exposes to the internet, used defensively and in authorized penetration testing
- **Corporate investigations**: Identifying the technical infrastructure of an organization under due diligence or investigation
- **Threat actor tracking**: Mapping the infrastructure of threat groups using C2 servers, phishing domains, or malware distribution sites
- **Brand protection**: Identifying unauthorized uses of a brand's domain patterns
- **Investigative journalism**: Understanding the technical infrastructure behind disinformation campaigns or fraud operations

The same techniques that security professionals use to harden defenses are used by investigators to map adversary infrastructure and by journalists to expose coordinated deception campaigns.

---

## 6.2 Domain Intelligence

### WHOIS Analysis

WHOIS is the protocol for querying domain registration information. Historically, WHOIS records contained registrant name, organization, address, phone, and email. GDPR compliance has significantly reduced the public availability of this data for European registrants, but:

- Many non-EU registrations still include full registrant data
- Historical WHOIS data (captured before GDPR) is available through specialized databases
- Organization names, nameservers, and technical contact emails often remain visible
- Business domain registrations may retain more data than personal registrations

**Key WHOIS data points**:
- Registrar (which company registered the domain)
- Registration and expiration dates
- Nameservers (which DNS infrastructure serves the domain)
- Registrant organization (often preserved even with individual data redacted)
- Privacy proxy service (presence of a privacy proxy reveals intent to obscure ownership)

**Tools**:
```bash
# Command-line WHOIS
whois example.com

# Domain-specific WHOIS query (for more detailed results)
whois -h whois.verisign-grs.com example.com

# Python whois library
python3 -c "import whois; w = whois.whois('example.com'); print(w)"
```

```python
import whois
import json
from datetime import datetime

def analyze_domain_whois(domain):
    """Extract and structure WHOIS data for analysis"""
    try:
        w = whois.whois(domain)
        return {
            'domain': domain,
            'registrar': w.registrar,
            'created': str(w.creation_date) if w.creation_date else None,
            'expires': str(w.expiration_date) if w.expiration_date else None,
            'updated': str(w.updated_date) if w.updated_date else None,
            'nameservers': list(w.name_servers) if w.name_servers else [],
            'registrant_org': w.org,
            'registrant_country': w.country,
            'status': w.status,
        }
    except Exception as e:
        return {'domain': domain, 'error': str(e)}

# Analyze a set of suspected related domains
domains = ['example.com', 'example-corp.com', 'example-login.com']
results = [analyze_domain_whois(d) for d in domains]
```

### Historical WHOIS Data

GDPR and privacy proxy services have obscured current WHOIS data, but historical snapshots remain accessible through commercial services:

- **DomainTools**: The most comprehensive historical WHOIS database. Commercial.
- **SecurityTrails**: Historical WHOIS data combined with DNS history. Free tier available, commercial for bulk access.
- **ViewDNS.info**: Free historical WHOIS queries with some limitations.
- **WhoisFreaks**: Historical WHOIS API access.

Historical WHOIS is particularly valuable for:
- Identifying who originally registered a now-privacy-protected domain
- Correlating domains registered with the same registrant information
- Tracking domain ownership changes over time
- Establishing timeline of domain use

### DNS Record Analysis

DNS records are fully public and revealing. A complete DNS analysis should collect:

**A and AAAA records**: IPv4 and IPv6 addresses the domain resolves to. These reveal hosting providers, CDN usage, and network relationships.

**MX records**: Mail exchange servers. These reveal email provider (Google Workspace, Microsoft 365, Proofpoint), which has implications for security posture and organizational setup.

**TXT records**: Text records used for SPF (which servers are authorized to send email for this domain), DKIM (email signing keys), DMARC (email authentication policy), domain ownership verification tokens, and sometimes internal infrastructure information.

**NS records**: Authoritative nameservers. These reveal DNS provider and sometimes organizational information.

**CNAME records**: Canonical name aliases, which often reveal use of specific CDN, hosting, or SaaS services.

**SOA record**: Start of Authority record containing the primary nameserver and administrative email (sometimes a real email address even on privacy-protected domains).

```python
import dns.resolver
import dns.zone
import dns.reversename

def full_dns_analysis(domain):
    """Comprehensive DNS record collection"""
    results = {}
    record_types = ['A', 'AAAA', 'MX', 'NS', 'TXT', 'SOA', 'CNAME']

    resolver = dns.resolver.Resolver()
    resolver.timeout = 5
    resolver.lifetime = 10

    for rtype in record_types:
        try:
            answers = resolver.resolve(domain, rtype)
            results[rtype] = [str(rdata) for rdata in answers]
        except (dns.resolver.NoAnswer, dns.resolver.NXDOMAIN,
                dns.resolver.NoNameservers, dns.exception.Timeout):
            results[rtype] = []

    # Parse SPF record for authorized senders
    if results.get('TXT'):
        for txt in results['TXT']:
            if txt.startswith('"v=spf1') or txt.startswith('v=spf1'):
                results['spf_includes'] = _parse_spf(txt)
                break

    return results

def _parse_spf(spf_record):
    """Extract included domains from SPF record"""
    includes = []
    parts = spf_record.replace('"', '').split()
    for part in parts:
        if part.startswith('include:') or part.startswith('redirect='):
            includes.append(part.split(':', 1)[1].split('=', 1)[-1])
    return includes

# Reverse DNS lookup
def reverse_dns(ip_address):
    """Get hostname from IP address"""
    try:
        reversed_name = dns.reversename.from_address(ip_address)
        answer = dns.resolver.resolve(reversed_name, 'PTR')
        return str(answer[0])
    except Exception:
        return None
```

### Certificate Transparency Analysis

Every TLS certificate issued by trusted Certificate Authorities (CAs) is logged to public Certificate Transparency (CT) logs. These logs are permanently searchable and reveal:

- All subdomains of a domain that have ever had certificates issued
- Historical certificate information even if the subdomain no longer exists
- Subject Alternative Names (SANs) — multiple domains covered by a single certificate
- Issuing CA and certificate validity periods

**Primary tools**:
- **crt.sh**: Free web interface and API for CT log searching
- **Censys**: Comprehensive certificate search with API access
- **certspotter** (Spackle Lab): Certificate monitoring tool

```python
import requests
import json
from datetime import datetime

def find_subdomains_via_crt(domain):
    """Query crt.sh for certificates containing the domain"""
    url = f"https://crt.sh/?q=%.{domain}&output=json"

    try:
        response = requests.get(url, timeout=30)
        data = response.json()
    except Exception as e:
        return {'error': str(e)}

    subdomains = set()
    certificates = []

    for entry in data:
        name_value = entry.get('name_value', '')
        not_before = entry.get('not_before', '')
        issuer = entry.get('issuer_name', '')

        # Extract individual names from the certificate
        for name in name_value.split('\n'):
            name = name.strip()
            if name.endswith(f'.{domain}') or name == domain:
                subdomains.add(name)

        certificates.append({
            'id': entry.get('id'),
            'logged_at': entry.get('entry_timestamp'),
            'not_before': not_before,
            'common_name': entry.get('common_name'),
            'issuer': issuer
        })

    return {
        'domain': domain,
        'unique_subdomains': sorted(list(subdomains)),
        'certificate_count': len(certificates),
        'oldest_certificate': min(c['not_before'] for c in certificates if c['not_before']),
        'latest_certificate': max(c['not_before'] for c in certificates if c['not_before'])
    }
```

---

## 6.3 IP Address Intelligence

### IP WHOIS and ASN Research

Each block of IP addresses is allocated to an organization by a Regional Internet Registry (RIR). ARIN (North America), RIPE (Europe), APNIC (Asia-Pacific), AFRINIC (Africa), and LACNIC (Latin America) maintain publicly searchable allocation databases.

Autonomous System Numbers (ASNs) identify organizations that control blocks of IP addresses. Large organizations have their own ASNs; smaller organizations are customers of ISPs that hold ASNs.

```python
import requests

def ip_intelligence(ip_address):
    """Gather intelligence about an IP address"""
    results = {}

    # IPInfo.io — geolocation and ASN data
    response = requests.get(f"https://ipinfo.io/{ip_address}/json")
    if response.status_code == 200:
        ipinfo = response.json()
        results['geolocation'] = {
            'country': ipinfo.get('country'),
            'region': ipinfo.get('region'),
            'city': ipinfo.get('city'),
            'org': ipinfo.get('org'),  # ASN and org name
            'hostname': ipinfo.get('hostname'),
        }

    # RIPE WHOIS lookup
    ripe_response = requests.get(
        f"https://rest.db.ripe.net/search.json?query-string={ip_address}&type-filter=inetnum"
    )
    if ripe_response.status_code == 200:
        results['ripe_data'] = ripe_response.json()

    return results

def asn_lookup(asn):
    """Look up organization controlling an ASN"""
    # BGPView API — free, good coverage
    response = requests.get(f"https://api.bgpview.io/asn/{asn}")
    if response.status_code == 200:
        data = response.json()['data']
        return {
            'asn': asn,
            'name': data.get('name'),
            'description': data.get('description'),
            'country': data.get('country_code'),
            'website': data.get('website'),
            'prefixes_ipv4': data.get('rir_allocation', {}).get('prefix'),
        }
```

### Passive DNS

Passive DNS databases record historical DNS resolutions — which domains have resolved to which IP addresses over time. This is enormously valuable for:

- Tracing domain/IP relationships across time
- Identifying domains that shared infrastructure with a target domain
- Pivoting from a known malicious IP to other domains hosted there
- Reconstructing historical infrastructure that has since changed

**Commercial Passive DNS services**:
- **SecurityTrails**: Comprehensive passive DNS with API access
- **RiskIQ PassiveTotal** (now Microsoft Defender): Enterprise passive DNS, particularly strong for threat intelligence
- **Farsight DNSDB**: Research-grade passive DNS with academic and commercial access tiers
- **VirusTotal**: Includes passive DNS data alongside malware intelligence

```python
import requests

class SecurityTrailsAPI:
    """SecurityTrails API client for passive DNS research"""

    def __init__(self, api_key):
        self.api_key = api_key
        self.base_url = "https://api.securitytrails.com/v1"
        self.headers = {"apikey": api_key, "Content-Type": "application/json"}

    def get_domain_history(self, domain):
        """Get DNS resolution history for a domain"""
        response = requests.get(
            f"{self.base_url}/history/{domain}/dns/a",
            headers=self.headers
        )
        return response.json()

    def get_ip_history(self, ip):
        """Get domains that have resolved to an IP"""
        response = requests.get(
            f"{self.base_url}/ips/nearby/{ip}",
            headers=self.headers
        )
        return response.json()

    def get_subdomains(self, domain):
        """Get all subdomains for a domain"""
        response = requests.get(
            f"{self.base_url}/domain/{domain}/subdomains",
            headers=self.headers
        )
        return response.json()

    def search_by_keyword(self, keyword, field="whois_email"):
        """Search for domains by WHOIS field content"""
        response = requests.post(
            f"{self.base_url}/domains/list",
            headers=self.headers,
            json={
                "filter": {field: keyword},
                "include_ips": False
            }
        )
        return response.json()
```

---

## 6.4 Shodan, Censys, and Internet Scanning Databases

The internet-wide scanning databases represent one of the most powerful tools in the technical OSINT arsenal. These services continuously scan the entire IPv4 address space (and portions of IPv6) and index what they find.

### Shodan

Shodan is the most well-known internet scanning database. It connects to every device exposed to the internet and records what it responds with — HTTP banners, SSH version information, TLS certificates, industrial control system protocols, and more.

**What Shodan reveals**:
- Web servers and their software versions
- Industrial control systems (SCADA, HMI devices)
- Network devices (routers, switches, cameras)
- Databases exposed to the internet
- IoT devices of all types
- Configuration information from banner responses
- TLS certificate chains

**Shodan search operators**:

```
# Basic domain search
hostname:example.com

# Search by ASN
asn:AS12345

# Find exposed databases
product:mongodb country:US

# Find industrial control systems
tag:ics country:US

# Find specific banners
"Server: Apache/2.4" country:DE

# Find devices with specific vulnerabilities
vuln:CVE-2021-44228 (Log4Shell)

# Combine filters
org:"Target Corporation" port:8080 http.title:"Admin"
```

**Python Shodan API**:

```python
import shodan

api = shodan.Shodan('YOUR_API_KEY')

def scan_organization(query):
    """Collect Shodan results for an organization"""
    results_list = []

    try:
        # Search Shodan
        results = api.search(query, limit=1000)

        print(f"Total results: {results['total']}")

        for result in results['matches']:
            entry = {
                'ip': result['ip_str'],
                'port': result['port'],
                'transport': result.get('transport', 'tcp'),
                'hostnames': result.get('hostnames', []),
                'org': result.get('org', ''),
                'isp': result.get('isp', ''),
                'country': result.get('location', {}).get('country_name', ''),
                'city': result.get('location', {}).get('city', ''),
                'last_update': result.get('timestamp', ''),
                'banner': result.get('data', '')[:500],  # Truncate long banners
                'product': result.get('product', ''),
                'version': result.get('version', ''),
                'cpes': result.get('cpe', []),
                'vulns': list(result.get('vulns', {}).keys()),
            }

            # Extract HTTP-specific data
            if 'http' in result:
                entry['http_title'] = result['http'].get('title', '')
                entry['http_server'] = result['http'].get('server', '')
                entry['http_status'] = result['http'].get('status', 0)

            # Extract SSL certificate info
            if 'ssl' in result:
                cert = result['ssl'].get('cert', {})
                subject = cert.get('subject', {})
                entry['ssl_cn'] = subject.get('CN', '')
                entry['ssl_org'] = subject.get('O', '')
                entry['ssl_expires'] = str(cert.get('expires', ''))

            results_list.append(entry)

    except shodan.APIError as e:
        print(f"Shodan error: {e}")

    return results_list

# Search for organization's infrastructure
results = scan_organization('org:"Target Company Name"')
```

### Censys

Censys provides similar internet-wide scanning data with a stronger focus on TLS certificates and a research-oriented API.

```python
from censys.search import CensysHosts
from censys.common.exceptions import CensysRateLimitExceededException

def censys_host_search(query, fields=None):
    """Search Censys hosts database"""
    h = CensysHosts()

    default_fields = [
        "ip", "services.port", "services.transport_protocol",
        "services.service_name", "services.banner",
        "services.tls.certificate.parsed.subject.common_name",
        "autonomous_system.name", "location.country"
    ]

    try:
        results = []
        for page in h.search(query, fields=fields or default_fields, pages=5):
            results.extend(page)
        return results
    except CensysRateLimitExceededException:
        print("Rate limit exceeded, implement backoff")
        return []
```

---

## 6.5 Subdomain Enumeration

Discovering all subdomains of a target domain reveals the full scope of an organization's web presence and often reveals forgotten, unsecured, or development infrastructure.

### Passive Subdomain Discovery

Passive methods collect subdomains without directly querying target infrastructure:

```python
import requests
from concurrent.futures import ThreadPoolExecutor, as_completed

def passive_subdomain_discovery(domain):
    """Aggregate subdomain data from multiple passive sources"""
    subdomains = set()

    # Source 1: Certificate Transparency
    try:
        crt_data = find_subdomains_via_crt(domain)  # From earlier in chapter
        subdomains.update(crt_data.get('unique_subdomains', []))
    except Exception as e:
        print(f"CRT.sh error: {e}")

    # Source 2: SecurityTrails (if API key available)
    # subdomains.update(securitytrails.get_subdomains(domain))

    # Source 3: VirusTotal
    try:
        vt_url = f"https://www.virustotal.com/api/v3/domains/{domain}/subdomains"
        headers = {"x-apikey": "YOUR_VT_API_KEY"}
        vt_response = requests.get(vt_url, headers=headers)
        if vt_response.status_code == 200:
            vt_data = vt_response.json()
            for sub in vt_data.get('data', []):
                subdomains.add(sub['id'])
    except Exception as e:
        print(f"VirusTotal error: {e}")

    # Source 4: HackerTarget
    try:
        ht_response = requests.get(
            f"https://api.hackertarget.com/hostsearch/?q={domain}"
        )
        for line in ht_response.text.strip().split('\n'):
            if ',' in line:
                subdomain = line.split(',')[0].strip()
                if subdomain.endswith(domain):
                    subdomains.add(subdomain)
    except Exception as e:
        print(f"HackerTarget error: {e}")

    return sorted(list(subdomains))

def resolve_subdomains(subdomains):
    """Resolve discovered subdomains to IP addresses"""
    import dns.resolver

    resolver = dns.resolver.Resolver()
    resolver.timeout = 3
    results = {}

    def resolve_single(subdomain):
        try:
            answers = resolver.resolve(subdomain, 'A')
            return subdomain, [str(r) for r in answers]
        except Exception:
            return subdomain, []

    with ThreadPoolExecutor(max_workers=50) as executor:
        futures = {executor.submit(resolve_single, sub): sub for sub in subdomains}
        for future in as_completed(futures):
            subdomain, ips = future.result()
            if ips:
                results[subdomain] = ips

    return results
```

---

## 6.6 Email Infrastructure Analysis

An organization's email infrastructure reveals hosting decisions, security posture, and sometimes organizational structure.

### MX Record Analysis

MX records identify which mail servers receive email for a domain:

- **Google Workspace**: MX records pointing to `aspmx.l.google.com` and similar indicate Google-hosted email
- **Microsoft 365**: MX records containing `protection.outlook.com` indicate Microsoft-hosted email
- **Proofpoint, Mimecast, Barracuda**: Email security gateways visible in MX records
- **Self-hosted**: Custom MX records pointing to the organization's own infrastructure

### SPF Analysis

SPF records enumerate which servers are authorized to send email as the domain. Analyzing `include:` statements reveals which third-party services the organization uses for email sending:

- Marketing platforms (Mailchimp's SPF include: `_spf.mcsv.net`)
- CRM platforms (Salesforce: `_spf.salesforce.com`)
- Transactional email services (SendGrid, Mailgun)
- Internal relay servers

### DMARC Analysis

DMARC records reveal email authentication policy:
- `p=none` — monitoring only, no enforcement (weak posture)
- `p=quarantine` — suspicious mail goes to spam
- `p=reject` — strict enforcement, unauthorized mail rejected
- Missing DMARC entirely — weakest possible posture

This reveals security sophistication and susceptibility to email spoofing.

---

## 6.7 Building Automated Recon Pipelines

Professional-grade reconnaissance combines multiple sources into automated pipelines:

```python
import asyncio
import aiohttp
import json
from dataclasses import dataclass, field
from typing import List, Dict

@dataclass
class DomainIntelReport:
    domain: str
    whois: Dict = field(default_factory=dict)
    dns_records: Dict = field(default_factory=dict)
    subdomains: List[str] = field(default_factory=list)
    certificates: List[Dict] = field(default_factory=list)
    shodan_results: List[Dict] = field(default_factory=list)
    related_domains: List[str] = field(default_factory=list)
    ip_addresses: List[str] = field(default_factory=list)
    analysis_timestamp: str = ""

class DomainReconPipeline:
    """Automated domain reconnaissance pipeline"""

    def __init__(self, shodan_key=None, securitytrails_key=None):
        self.shodan_key = shodan_key
        self.st_key = securitytrails_key

    async def run_full_recon(self, domain) -> DomainIntelReport:
        """Execute full reconnaissance pipeline for a domain"""
        from datetime import datetime

        report = DomainIntelReport(
            domain=domain,
            analysis_timestamp=datetime.utcnow().isoformat()
        )

        print(f"[*] Starting reconnaissance for {domain}")

        # Phase 1: WHOIS
        print("[*] Collecting WHOIS data...")
        report.whois = analyze_domain_whois(domain)

        # Phase 2: DNS
        print("[*] Collecting DNS records...")
        report.dns_records = full_dns_analysis(domain)

        # Extract IP addresses from A records
        if report.dns_records.get('A'):
            report.ip_addresses = report.dns_records['A']

        # Phase 3: Certificate transparency
        print("[*] Querying certificate transparency...")
        crt_data = find_subdomains_via_crt(domain)
        report.subdomains = crt_data.get('unique_subdomains', [])
        report.certificates = crt_data.get('certificates', [])

        # Phase 4: Resolve subdomains
        if report.subdomains:
            print(f"[*] Resolving {len(report.subdomains)} subdomains...")
            resolved = resolve_subdomains(report.subdomains)
            for subdomain, ips in resolved.items():
                report.ip_addresses.extend(ips)
            report.ip_addresses = list(set(report.ip_addresses))

        # Phase 5: Shodan lookup for discovered IPs
        if self.shodan_key and report.ip_addresses:
            print(f"[*] Querying Shodan for {len(report.ip_addresses)} IPs...")
            import shodan as shodan_lib
            api = shodan_lib.Shodan(self.shodan_key)
            for ip in report.ip_addresses[:20]:  # Limit to 20 IPs
                try:
                    host_data = api.host(ip)
                    report.shodan_results.append({
                        'ip': ip,
                        'ports': [p['port'] for p in host_data.get('data', [])],
                        'org': host_data.get('org', ''),
                        'hostnames': host_data.get('hostnames', []),
                    })
                except Exception:
                    pass

        print(f"[+] Reconnaissance complete for {domain}")
        return report

    def generate_report(self, intel_report: DomainIntelReport) -> str:
        """Generate formatted intelligence report"""
        lines = [
            f"# Domain Intelligence Report: {intel_report.domain}",
            f"**Generated**: {intel_report.analysis_timestamp}",
            "",
            "## WHOIS Summary",
            f"- Registrar: {intel_report.whois.get('registrar', 'Unknown')}",
            f"- Created: {intel_report.whois.get('created', 'Unknown')}",
            f"- Expires: {intel_report.whois.get('expires', 'Unknown')}",
            f"- Registrant Org: {intel_report.whois.get('registrant_org', 'Unknown')}",
            "",
            "## DNS Infrastructure",
            f"- IP Addresses: {', '.join(intel_report.dns_records.get('A', []))}",
            f"- Mail Servers: {', '.join(intel_report.dns_records.get('MX', []))}",
            f"- Nameservers: {', '.join(intel_report.dns_records.get('NS', []))}",
            "",
            f"## Subdomains Discovered ({len(intel_report.subdomains)})",
        ]

        for sub in intel_report.subdomains[:50]:
            lines.append(f"- {sub}")

        if len(intel_report.subdomains) > 50:
            lines.append(f"... and {len(intel_report.subdomains) - 50} more")

        lines.extend([
            "",
            f"## Internet-Exposed Services ({len(intel_report.shodan_results)} IPs analyzed)",
        ])

        for sh in intel_report.shodan_results:
            lines.append(f"- {sh['ip']} ({sh['org']}): ports {sh['ports']}")

        return '\n'.join(lines)
```

---

## 6.8 Threat Actor Infrastructure Research

One of the most important applications of network reconnaissance in cybersecurity is mapping threat actor infrastructure — the domains, IPs, and servers used by malicious actors for command-and-control, phishing, and malware distribution.

### Infrastructure Correlation Techniques

Threat actors often reuse infrastructure patterns that enable tracking:

**Nameserver clustering**: Malicious domains registered through specific registrars or using specific nameservers can be grouped and monitored.

**SSL certificate patterns**: Threat actors who use self-signed certificates or certificates from specific CAs with specific organizational details can be identified through certificate search.

**Registration pattern analysis**: Domains registered at the same time, with the same registrar, using similar naming patterns, or sharing WHOIS data (historical) reveal coordinated registration campaigns.

**IP neighborhood analysis**: Malicious domains often share IP space. Finding one malicious domain on an IP block suggests investigating neighboring IPs.

**RiskIQ / Microsoft Defender for Threat Intelligence** provides specialized passive DNS and certificate data tuned for threat intelligence use cases.

---

## Summary

Domain and network reconnaissance provides a technical layer of OSINT that is largely invisible through content-focused investigation. DNS records, WHOIS data, certificate transparency logs, passive DNS databases, and internet scanning services like Shodan reveal organizational infrastructure, hosting relationships, email systems, and historical presence that complement and enrich other OSINT findings.

Effective network reconnaissance requires passive data collection from multiple sources, combined into a coherent picture through systematic analysis. Automated pipelines enable repeatable, comprehensive coverage that would be impractical to achieve manually.

The same techniques that security professionals use for attack surface analysis serve broader OSINT purposes: corporate investigations, threat intelligence, fraud investigation, and organizational mapping.

---

## Common Mistakes and Pitfalls

- **Relying on current WHOIS only**: Historical WHOIS data often reveals what current privacy-protected records hide
- **Single-tool recon**: Any single tool has coverage gaps; always correlate across multiple sources
- **Ignoring certificate transparency**: CT logs are among the most overlooked and most valuable subdomain discovery sources
- **Active probing without authorization**: Active scanning of systems is potentially illegal without explicit authorization; passive reconnaissance from public databases is unambiguously legal
- **Treating Shodan results as current**: Shodan scans are periodic; a result from weeks ago may not reflect current status
- **Missing ASN analysis**: ASN research connects disparate IP ranges to the same organization

---

## Further Reading

- SANS FOR578 Threat Intelligence — network infrastructure analysis modules
- Krebs on Security — Brian Krebs's investigative reporting demonstrates infrastructure tracking methodology
- DomainTools blog — frequent practitioner-focused articles on domain intelligence
- MITRE ATT&CK — threat actor infrastructure patterns in documented groups
- Shodan documentation — comprehensive operator and API reference
