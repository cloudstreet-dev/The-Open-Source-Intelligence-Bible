# Chapter 20: Threat Intelligence and Cybersecurity Investigations

## Learning Objectives

By the end of this chapter, you will be able to:
- Apply OSINT to cybersecurity threat intelligence workflows
- Track threat actor infrastructure using domain and network intelligence techniques
- Build indicator of compromise (IOC) collections and pivot chains
- Integrate OSINT with threat intelligence platforms (TIPs)
- Conduct malware infrastructure analysis using public data
- Attribute campaigns to threat actors using open-source indicators

---

## 20.1 OSINT in Cybersecurity

Threat intelligence analysts, security operations center (SOC) analysts, and incident responders use OSINT differently from other OSINT practitioners: they are primarily working backwards from technical indicators — an IP address, a domain, a file hash, a malware family — to understand the adversary, their tools, techniques, and procedures (TTPs), and their broader infrastructure.

This investigative direction is sometimes called "threat actor attribution" when the goal is identifying who is behind an attack, and "campaign tracking" when the goal is mapping all infrastructure associated with a specific threat.

The OSINT methods from Parts II and III apply directly, but with cybersecurity-specific sources and pivoting patterns.

---

## 20.2 Cybersecurity OSINT Sources

Beyond the general sources covered in Part II, cybersecurity OSINT has a specialized source ecosystem:

### Threat Intelligence Platforms

**VirusTotal**: Crowdsourced malware intelligence database. Submit files or hashes to receive analysis from 70+ antivirus engines. Also provides domain, IP, and URL intelligence including passive DNS and associated file analysis.

**MISP (Malware Information Sharing Platform)**: Open-source threat intelligence sharing platform used by organizations and threat intelligence communities.

**AlienVault OTX (Open Threat Exchange)**: Community threat intelligence sharing platform. Free IOC feeds and community-contributed threat intelligence.

**Abuse.ch feeds**: Free threat intelligence feeds including URLhaus (malicious URLs), MalwareBazaar (malware samples), Feodo Tracker (botnet infrastructure).

**MITRE ATT&CK**: Framework and knowledge base of adversary tactics, techniques, and procedures based on real-world observations.

### Passive DNS and Infrastructure Intelligence

**Passive DNS** (as covered in Chapter 6) is particularly important in threat intelligence. Malicious infrastructure typically has distinctive passive DNS characteristics:

- Recently registered domains (freshly registered domains have higher fraud/malware association)
- Domains with suspicious TLDs (.xyz, .top, .click, .work in high-malware contexts)
- Domains on IP ranges with many other malicious domains
- Domains with very short TTLs (common in fast-flux infrastructure)
- Domains sharing infrastructure with known malicious domains

### Malware Analysis Platforms

**ANY.RUN**: Interactive malware sandbox with public results. Provides network indicators extracted from malware execution.

**Cuckoo Sandbox** (self-hosted or cloud): Open-source malware analysis. Network indicators in sandbox reports.

**Hybrid Analysis**: Free malware analysis platform with community result sharing.

**Joe Sandbox**: Commercial malware analysis with extensive network behavior reporting.

---

## 20.3 IOC Pivoting in Threat Intelligence

The pivot-based investigation methodology from Chapter 4 maps directly to threat intelligence practice.

### Threat Actor Infrastructure Pivoting

```python
import requests
import json
from typing import Dict, List, Set, Optional
from dataclasses import dataclass, field

@dataclass
class ThreatIndicator:
    """A threat intelligence indicator with context"""
    indicator: str
    indicator_type: str  # 'ip', 'domain', 'hash', 'url', 'email'
    confidence: str
    first_seen: Optional[str] = None
    last_seen: Optional[str] = None
    tags: List[str] = field(default_factory=list)
    source: str = ""
    related_indicators: List['ThreatIndicator'] = field(default_factory=list)

class ThreatIntelPivot:
    """
    Pivot through threat infrastructure using public intelligence sources
    """

    def __init__(self, vt_api_key: str = None, shodan_api_key: str = None):
        self.vt_api_key = vt_api_key
        self.shodan_api_key = shodan_api_key
        self.pivoted = set()  # Track what we've already pivoted on

    def virustotal_domain_report(self, domain: str) -> dict:
        """Get VirusTotal report for a domain"""
        if not self.vt_api_key:
            return {'error': 'VirusTotal API key required'}

        url = f"https://www.virustotal.com/api/v3/domains/{domain}"
        headers = {'x-apikey': self.vt_api_key}

        response = requests.get(url, headers=headers)
        if response.status_code == 200:
            data = response.json().get('data', {}).get('attributes', {})
            return {
                'domain': domain,
                'categories': data.get('categories', {}),
                'last_analysis_stats': data.get('last_analysis_stats', {}),
                'reputation': data.get('reputation', 0),
                'registrar': data.get('registrar', ''),
                'creation_date': data.get('creation_date', ''),
                'dns_records': data.get('last_dns_records', []),
                'communicating_files': [],  # Available in detailed query
                'popularity_ranks': data.get('popularity_ranks', {}),
            }
        return {'error': response.status_code}

    def virustotal_ip_report(self, ip: str) -> dict:
        """Get VirusTotal report for an IP address"""
        if not self.vt_api_key:
            return {'error': 'VirusTotal API key required'}

        url = f"https://www.virustotal.com/api/v3/ip_addresses/{ip}"
        headers = {'x-apikey': self.vt_api_key}

        response = requests.get(url, headers=headers)
        if response.status_code == 200:
            data = response.json().get('data', {}).get('attributes', {})
            return {
                'ip': ip,
                'asn': data.get('asn', ''),
                'as_owner': data.get('as_owner', ''),
                'country': data.get('country', ''),
                'reputation': data.get('reputation', 0),
                'last_analysis_stats': data.get('last_analysis_stats', {}),
                'total_votes': data.get('total_votes', {}),
                'network': data.get('network', ''),
                'regional_internet_registry': data.get('regional_internet_registry', ''),
            }
        return {'error': response.status_code}

    def virustotal_file_report(self, sha256: str) -> dict:
        """Get VirusTotal report for a file hash"""
        if not self.vt_api_key:
            return {'error': 'VirusTotal API key required'}

        url = f"https://www.virustotal.com/api/v3/files/{sha256}"
        headers = {'x-apikey': self.vt_api_key}

        response = requests.get(url, headers=headers)
        if response.status_code == 200:
            data = response.json().get('data', {}).get('attributes', {})
            return {
                'sha256': sha256,
                'md5': data.get('md5', ''),
                'sha1': data.get('sha1', ''),
                'meaningful_name': data.get('meaningful_name', ''),
                'type_description': data.get('type_description', ''),
                'size': data.get('size', 0),
                'first_submission_date': data.get('first_submission_date', ''),
                'last_analysis_stats': data.get('last_analysis_stats', {}),
                'popular_threat_classification': data.get('popular_threat_classification', {}),
                'names': data.get('names', [])[:10],
            }
        return {'error': response.status_code}

    def pivot_from_hash(self, file_hash: str, depth: int = 2) -> Dict:
        """
        Pivot from a malware hash to associated infrastructure
        """
        if file_hash in self.pivoted:
            return {}
        self.pivoted.add(file_hash)

        results = {
            'starting_hash': file_hash,
            'file_data': {},
            'contacted_domains': [],
            'contacted_ips': [],
            'dropped_files': [],
            'sibling_files': []  # Files that contact the same C2
        }

        # Get file report
        file_data = self.virustotal_file_report(file_hash)
        results['file_data'] = file_data

        if not self.vt_api_key:
            return results

        # Get contacted domains
        domain_url = f"https://www.virustotal.com/api/v3/files/{file_hash}/contacted_domains"
        headers = {'x-apikey': self.vt_api_key}

        response = requests.get(domain_url, headers=headers)
        if response.status_code == 200:
            domains = response.json().get('data', [])
            for domain_data in domains:
                domain = domain_data.get('id', '')
                results['contacted_domains'].append(domain)

                # Pivot on each domain (limited by depth)
                if depth > 0 and domain not in self.pivoted:
                    domain_intel = self.pivot_from_domain(domain, depth=depth-1)
                    results['domain_intel'] = domain_intel

        # Get contacted IPs
        ip_url = f"https://www.virustotal.com/api/v3/files/{file_hash}/contacted_ips"
        response = requests.get(ip_url, headers=headers)
        if response.status_code == 200:
            ips = response.json().get('data', [])
            results['contacted_ips'] = [ip.get('id') for ip in ips]

        return results

    def pivot_from_domain(self, domain: str, depth: int = 1) -> Dict:
        """
        Pivot from a suspicious domain to associated infrastructure
        """
        if domain in self.pivoted:
            return {}
        self.pivoted.add(domain)

        results = {
            'domain': domain,
            'vt_report': {},
            'passive_dns': [],
            'related_domains': [],
            'certificate_data': {},
        }

        # VirusTotal domain report
        results['vt_report'] = self.virustotal_domain_report(domain)

        # Certificate transparency for related subdomains
        try:
            crt_response = requests.get(
                f"https://crt.sh/?q=%.{domain}&output=json",
                timeout=15
            )
            if crt_response.status_code == 200:
                certs = crt_response.json()
                results['certificate_data'] = {
                    'cert_count': len(certs),
                    'earliest_cert': min((c.get('not_before', '') for c in certs), default=''),
                    'latest_cert': max((c.get('not_before', '') for c in certs), default=''),
                }
        except Exception:
            pass

        return results

    def abusech_check(self, indicator: str, indicator_type: str) -> dict:
        """
        Check abuse.ch databases for a known indicator
        """
        results = {}

        if indicator_type == 'domain' or indicator_type == 'url':
            # URLhaus
            response = requests.post(
                'https://urlhaus-api.abuse.ch/v1/url/',
                data={'url': indicator}
            )
            if response.status_code == 200:
                results['urlhaus'] = response.json()

        if indicator_type == 'ip':
            # Feodo Tracker
            response = requests.post(
                'https://feodotracker.abuse.ch/api/v1/host/info/',
                data={'host': indicator}
            )
            if response.status_code == 200:
                results['feodo'] = response.json()

        if indicator_type == 'hash':
            # MalwareBazaar
            response = requests.post(
                'https://mb-api.abuse.ch/api/v1/',
                data={'query': 'get_info', 'hash': indicator}
            )
            if response.status_code == 200:
                results['malwarebazaar'] = response.json()

        return results
```

---

## 20.4 Campaign Attribution and Actor Tracking

Threat actor attribution — determining who is behind an attack — is one of the most challenging tasks in cybersecurity. Open-source indicators can support attribution when they:

- Match known actor TTPs in MITRE ATT&CK
- Share infrastructure with previously attributed campaigns
- Use malware families with known origin communities
- Follow registration patterns associated with specific actor groups

### MITRE ATT&CK Integration

```python
import requests

class MITREATTACKLookup:
    """Query MITRE ATT&CK framework data"""

    def __init__(self):
        self.base_url = "https://raw.githubusercontent.com/mitre/cti/master"
        self._cached_data = {}

    def get_technique(self, technique_id: str) -> dict:
        """Look up ATT&CK technique details"""
        # MITRE provides ATT&CK data as STIX JSON
        if 'enterprise' not in self._cached_data:
            response = requests.get(
                f"{self.base_url}/enterprise-attack/enterprise-attack.json"
            )
            if response.status_code == 200:
                self._cached_data['enterprise'] = response.json()
            else:
                return {}

        data = self._cached_data.get('enterprise', {})
        objects = data.get('objects', [])

        for obj in objects:
            if obj.get('type') == 'attack-pattern':
                external_refs = obj.get('external_references', [])
                for ref in external_refs:
                    if ref.get('external_id') == technique_id:
                        return {
                            'id': technique_id,
                            'name': obj.get('name', ''),
                            'description': obj.get('description', ''),
                            'platforms': obj.get('x_mitre_platforms', []),
                            'detection': obj.get('x_mitre_detection', ''),
                            'kill_chain_phases': obj.get('kill_chain_phases', []),
                        }
        return {}

    def find_groups_using_technique(self, technique_id: str) -> list:
        """Find threat actor groups that use a specific technique"""
        data = self._cached_data.get('enterprise', {})
        if not data:
            return []

        groups = []
        relationships = [obj for obj in data.get('objects', []) if obj.get('type') == 'relationship']
        techniques = {obj['id']: obj for obj in data.get('objects', []) if obj.get('type') == 'attack-pattern'}
        group_objects = {obj['id']: obj for obj in data.get('objects', []) if obj.get('type') == 'intrusion-set'}

        for rel in relationships:
            if rel.get('relationship_type') == 'uses':
                # Check if target is the technique we're looking for
                target_ref = rel.get('target_ref', '')
                if target_ref in techniques:
                    tech = techniques[target_ref]
                    for ext_ref in tech.get('external_references', []):
                        if ext_ref.get('external_id') == technique_id:
                            source_ref = rel.get('source_ref', '')
                            if source_ref in group_objects:
                                group = group_objects[source_ref]
                                groups.append({
                                    'id': source_ref,
                                    'name': group.get('name', ''),
                                    'aliases': group.get('aliases', []),
                                    'description': group.get('description', '')[:200]
                                })

        return groups
```

---

## 20.5 Threat Intelligence Reporting

```python
def generate_threat_intelligence_report(
    initial_indicator: str,
    indicator_type: str,
    pivoted_infrastructure: dict,
    mitre_techniques: list = None
) -> str:
    """Generate a structured threat intelligence report"""

    import anthropic
    client = anthropic.Anthropic()

    infra_text = json.dumps(pivoted_infrastructure, indent=2)
    techniques_text = json.dumps(mitre_techniques or [], indent=2)

    prompt = f"""You are a threat intelligence analyst documenting an investigation.

INITIAL INDICATOR: {initial_indicator} ({indicator_type})

DISCOVERED INFRASTRUCTURE:
{infra_text[:3000]}

OBSERVED MITRE ATT&CK TECHNIQUES:
{techniques_text}

Write a structured threat intelligence report including:

## Executive Summary
[Brief overview of the threat, affected indicators, and significance]

## Technical Analysis

### Indicator Overview
[Analysis of the initial indicator and its significance]

### Infrastructure Analysis
[Discovered infrastructure, hosting patterns, and relationships]

### Malware Analysis (if applicable)
[Technical characteristics of any identified malware]

### TTPs (Tactics, Techniques, Procedures)
[Observed techniques mapped to MITRE ATT&CK where possible]

## Attribution Assessment
[Any attribution indicators, with confidence level]
Note: Attribution is difficult and requires multiple independent indicators.
Clearly mark attribution confidence level.

## IOC Summary
[Structured list of indicators for blocklist/detection use]

## Defensive Recommendations
[Specific detection and mitigation recommendations]

Write in a professional threat intelligence style.
Clearly distinguish confirmed facts from analytical assessments.
Mark confidence levels for attribution claims."""

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=3000,
        messages=[{"role": "user", "content": prompt}]
    )

    return response.content[0].text
```

---

## 20.6 Integration with Security Tools

```python
import json

def export_iocs_to_stix(indicators: List[ThreatIndicator], report_name: str) -> dict:
    """
    Export collected indicators to STIX 2.1 format for sharing
    """
    from datetime import datetime
    import uuid

    stix_bundle = {
        "type": "bundle",
        "id": f"bundle--{uuid.uuid4()}",
        "objects": []
    }

    # Create indicator objects
    for indicator in indicators:
        stix_indicator_type_map = {
            'domain': 'domain-name',
            'ip': 'ipv4-addr',
            'url': 'url',
            'hash': 'file',
            'email': 'email-addr'
        }

        stix_type = stix_indicator_type_map.get(indicator.indicator_type, 'indicator')

        stix_object = {
            "type": "indicator",
            "spec_version": "2.1",
            "id": f"indicator--{uuid.uuid4()}",
            "created": datetime.now().strftime("%Y-%m-%dT%H:%M:%S.000Z"),
            "modified": datetime.now().strftime("%Y-%m-%dT%H:%M:%S.000Z"),
            "name": indicator.indicator,
            "pattern": f"[{stix_type}:value = '{indicator.indicator}']",
            "pattern_type": "stix",
            "valid_from": datetime.now().strftime("%Y-%m-%dT%H:%M:%S.000Z"),
            "confidence": {"confirmed": 85, "probable": 65, "possible": 35}.get(
                indicator.confidence, 50
            ),
            "labels": indicator.tags,
        }

        stix_bundle["objects"].append(stix_object)

    return stix_bundle

def export_to_misp_format(indicators: List[ThreatIndicator]) -> List[dict]:
    """Format indicators for MISP import"""
    misp_attributes = []

    type_map = {
        'domain': 'domain',
        'ip': 'ip-dst',
        'url': 'url',
        'hash': 'sha256',
        'email': 'email-src',
    }

    for indicator in indicators:
        misp_type = type_map.get(indicator.indicator_type, 'other')
        misp_attributes.append({
            'type': misp_type,
            'value': indicator.indicator,
            'comment': f"Source: {indicator.source}. Tags: {', '.join(indicator.tags)}",
            'to_ids': indicator.indicator_type in ('domain', 'ip', 'hash', 'url'),
            'category': 'Network activity' if indicator.indicator_type in ('domain', 'ip', 'url') else 'Payload delivery',
        })

    return misp_attributes
```

---

## Summary

Threat intelligence represents one of the most technically demanding OSINT applications, requiring deep integration of domain and network intelligence with specialized cybersecurity knowledge bases. The pivot-based investigation methodology applies directly to threat actor infrastructure tracking — moving from a known malicious indicator to map all associated infrastructure.

Key data sources for cybersecurity OSINT include VirusTotal, passive DNS databases, abuse.ch feeds, certificate transparency logs, and MITRE ATT&CK. Professional threat intelligence products are formatted in standardized ways (STIX/TAXII, MISP) to enable sharing across platforms.

Attribution requires multiple independent indicators and should always be presented with explicit confidence levels. OSINT-based attribution is typically categorized as probable rather than confirmed, absent government-level intelligence access.

---

## Common Mistakes and Pitfalls

- **Attribution overconfidence**: Claiming attribution certainty from infrastructure overlap that could have many explanations
- **IOC staleness**: Using indicators without verifying current relevance — attackers change infrastructure rapidly
- **Indicator pollution**: Adding low-confidence indicators to shared feeds that become sources of false positives
- **Missing context**: Reporting raw indicators without context about why they are significant
- **MITRE ATT&CK misapplication**: Mapping observed behaviors to techniques incorrectly inflates apparent technique usage

---

## Further Reading

- MITRE ATT&CK framework — attack.mitre.org
- SANS FOR578 Cyber Threat Intelligence course
- Recorded Future, Mandiant, and CrowdStrike threat intelligence blogs
- The Diamond Model of Intrusion Analysis — Caltagirone et al. (2013)
- Shodan + VirusTotal + RiskIQ integration guides
