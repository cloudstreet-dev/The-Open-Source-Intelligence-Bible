# Chapter 24: Adversarial OSINT and Counter-Intelligence

## Learning Objectives

By the end of this chapter, you will be able to:
- Understand how malicious actors use OSINT techniques for targeting and social engineering
- Identify and mitigate your own organizational attack surface exposed through public sources
- Implement operational security measures that reduce exposure without sacrificing effectiveness
- Recognize disinformation, synthetic personas, and coordinated inauthentic behavior
- Conduct counter-intelligence assessments to identify who may be researching your organization
- Design privacy-preserving investigation workflows that protect investigator identity

---

## 24.1 The Adversarial OSINT Landscape

OSINT is a dual-use discipline. Every technique in this book that helps a legitimate investigator discover information is equally available to:

- **Social engineers** who harvest employee information to craft convincing phishing campaigns
- **Competitor intelligence operatives** who track hiring patterns, patent filings, and executive movements
- **Stalkers and harassers** who aggregate public social media data to locate and surveil targets
- **State-sponsored threat actors** who conduct reconnaissance before targeted intrusion campaigns
- **Fraudsters** who impersonate executives or vendors using publicly available organizational data

Understanding adversarial OSINT serves two purposes: it makes defensive measures comprehensible (you understand what you're defending against) and it prepares investigators to recognize when they are being profiled by adversaries.

---

## 24.2 Attack Surface Exposure Through Public Data

### Organizational Exposure Assessment

Most organizations have a significantly larger public OSINT exposure than they realize. A systematic exposure assessment treats the organization as a target:

```python
import requests
from dataclasses import dataclass, field
from typing import List, Dict, Set
from datetime import datetime

@dataclass
class ExposureAssessment:
    """Organizational OSINT exposure assessment"""
    organization: str
    domain: str

    # Personnel exposure
    employee_names_exposed: List[str] = field(default_factory=list)
    email_patterns_confirmed: List[str] = field(default_factory=list)
    executive_profiles: List[Dict] = field(default_factory=list)
    org_chart_fragments: List[Dict] = field(default_factory=list)

    # Technical exposure
    subdomains_discovered: Set[str] = field(default_factory=set)
    exposed_services: List[Dict] = field(default_factory=list)
    email_infrastructure: Dict = field(default_factory=dict)
    code_repositories: List[str] = field(default_factory=list)
    credential_exposures: List[Dict] = field(default_factory=list)

    # Physical exposure
    office_locations_public: List[str] = field(default_factory=list)
    employee_home_indicators: List[Dict] = field(default_factory=list)

    # Reputational exposure
    financial_indicators: List[Dict] = field(default_factory=list)
    legal_exposure: List[Dict] = field(default_factory=list)

    assessment_date: str = field(default_factory=lambda: datetime.now().isoformat()[:10])

    def exposure_score(self) -> int:
        """Calculate rough exposure score"""
        score = 0
        score += len(self.employee_names_exposed) * 2
        score += len(self.email_patterns_confirmed) * 5
        score += len(self.subdomains_discovered) * 1
        score += len(self.exposed_services) * 3
        score += len(self.credential_exposures) * 10
        return score

    def risk_level(self) -> str:
        score = self.exposure_score()
        if score < 20:
            return "LOW"
        elif score < 50:
            return "MEDIUM"
        elif score < 100:
            return "HIGH"
        else:
            return "CRITICAL"


class OrganizationExposureAuditor:
    """
    Assess an organization's OSINT exposure from an adversary's perspective
    For use by authorized red team assessors or internal security teams
    """

    def __init__(self, organization: str, domain: str):
        self.assessment = ExposureAssessment(organization=organization, domain=domain)
        self.session = requests.Session()
        self.session.headers.update({'User-Agent': 'ExposureAudit/1.0'})

    def discover_subdomains(self) -> Set[str]:
        """Find exposed subdomains via Certificate Transparency"""
        subdomains = set()
        try:
            response = self.session.get(
                f"https://crt.sh/?q=%.{self.assessment.domain}&output=json",
                timeout=30
            )
            if response.status_code == 200:
                for cert in response.json():
                    for name in cert.get('name_value', '').split('\n'):
                        name = name.strip().lstrip('*.')
                        if name.endswith(self.assessment.domain) and '.' in name:
                            subdomains.add(name.lower())

        except Exception as e:
            print(f"CT log error: {e}")

        self.assessment.subdomains_discovered = subdomains
        return subdomains

    def check_email_exposure(self) -> List[Dict]:
        """
        Check for email exposure in breach databases
        Uses HaveIBeenPwned domain search (requires Enterprise API key)
        """
        exposures = []

        # Check Hunter.io for confirmed email patterns (free tier)
        try:
            import os
            api_key = os.getenv('HUNTER_API_KEY', '')
            if api_key:
                response = self.session.get(
                    "https://api.hunter.io/v2/domain-search",
                    params={
                        'domain': self.assessment.domain,
                        'api_key': api_key,
                        'limit': 10
                    },
                    timeout=15
                )
                if response.status_code == 200:
                    data = response.json().get('data', {})
                    pattern = data.get('pattern')
                    if pattern:
                        self.assessment.email_patterns_confirmed.append(f"{pattern}@{self.assessment.domain}")

                    for email_obj in data.get('emails', []):
                        exposures.append({
                            'email': email_obj.get('value'),
                            'first_name': email_obj.get('first_name'),
                            'last_name': email_obj.get('last_name'),
                            'position': email_obj.get('position'),
                            'source': 'Hunter.io'
                        })
                        if email_obj.get('first_name') or email_obj.get('last_name'):
                            name = f"{email_obj.get('first_name', '')} {email_obj.get('last_name', '')}".strip()
                            if name:
                                self.assessment.employee_names_exposed.append(name)

        except Exception as e:
            print(f"Hunter.io error: {e}")

        self.assessment.credential_exposures.extend(exposures)
        return exposures

    def check_github_exposure(self) -> List[Dict]:
        """Check GitHub for inadvertent code/secret exposures"""
        exposures = []
        sensitive_queries = [
            f'org:{self.assessment.organization.replace(" ", "")} password',
            f'org:{self.assessment.organization.replace(" ", "")} api_key',
            f'org:{self.assessment.organization.replace(" ", "")} secret',
            f'"{self.assessment.domain}" password filename:.env',
        ]

        headers = {
            'Accept': 'application/vnd.github.v3+json',
            'Authorization': f"token {__import__('os').getenv('GITHUB_TOKEN', '')}"
        }

        for query in sensitive_queries:
            try:
                response = self.session.get(
                    'https://api.github.com/search/code',
                    params={'q': query, 'per_page': 5},
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
                            'severity': 'HIGH' if 'password' in query.lower() else 'MEDIUM'
                        })

            except Exception as e:
                print(f"GitHub search error: {e}")

        self.assessment.code_repositories.extend([e['url'] for e in exposures])
        return exposures

    def check_shodan_exposure(self) -> List[Dict]:
        """Check Shodan for exposed services"""
        exposed = []
        import os

        api_key = os.getenv('SHODAN_API_KEY', '')
        if not api_key:
            return []

        try:
            response = self.session.get(
                'https://api.shodan.io/shodan/host/search',
                params={
                    'key': api_key,
                    'query': f'hostname:{self.assessment.domain}',
                    'facets': 'port,product',
                    'minify': False
                },
                timeout=15
            )

            if response.status_code == 200:
                data = response.json()
                for host in data.get('matches', []):
                    service = {
                        'ip': host.get('ip_str'),
                        'port': host.get('port'),
                        'product': host.get('product', 'Unknown'),
                        'hostnames': host.get('hostnames', []),
                        'vulns': list(host.get('vulns', {}).keys()),
                        'last_seen': host.get('timestamp')
                    }
                    exposed.append(service)

                    if service['vulns']:
                        self.assessment.exposed_services.append({
                            **service,
                            'severity': 'HIGH',
                            'note': f"Known CVEs: {', '.join(service['vulns'][:3])}"
                        })

        except Exception as e:
            print(f"Shodan error: {e}")

        return exposed

    def generate_exposure_report(self) -> str:
        """Generate the organizational exposure assessment report"""
        a = self.assessment

        report = [
            f"# Organizational OSINT Exposure Assessment",
            f"**Organization**: {a.organization}",
            f"**Domain**: {a.domain}",
            f"**Assessment Date**: {a.assessment_date}",
            f"**Exposure Score**: {a.exposure_score()}",
            f"**Risk Level**: {a.risk_level()}",
            "",
            "---",
            "",
            "## Executive Summary",
            "",
            f"This assessment identified {len(a.employee_names_exposed)} employee names exposed in public sources, "
            f"{len(a.subdomains_discovered)} subdomains, {len(a.exposed_services)} services with security concerns, "
            f"and {len(a.credential_exposures)} credential/email exposures.",
            "",
        ]

        if a.exposed_services:
            report.extend([
                "## ⚠️ High-Priority Technical Exposures",
                ""
            ])
            for service in a.exposed_services:
                report.append(f"- **{service['ip']}:{service['port']}** — {service.get('product', 'Unknown')}")
                if service.get('vulns'):
                    report.append(f"  CVEs: {', '.join(service['vulns'][:5])}")

        if a.credential_exposures:
            report.extend([
                "",
                "## Personnel/Email Exposures",
                ""
            ])
            for exp in a.credential_exposures[:10]:
                report.append(f"- {exp.get('email', 'Unknown')} — {exp.get('position', 'Position unknown')}")

        if a.email_patterns_confirmed:
            report.extend([
                "",
                "## Confirmed Email Patterns",
                ""
            ])
            for pattern in a.email_patterns_confirmed:
                report.append(f"- `{pattern}` — Enables automated employee email construction")

        report.extend([
            "",
            "## Remediation Priorities",
            "",
            "1. Address all exposed services with known CVEs immediately",
            "2. Rotate any credentials found in code repositories",
            "3. Review and restrict public LinkedIn information for high-value employees",
            "4. Implement DMARC/DKIM email authentication to prevent spoofing",
            "5. Conduct employee social media awareness training"
        ])

        return '\n'.join(report)
```

---

## 24.3 Synthetic Personas and Disinformation Detection

Adversarial actors use synthetic personas — fake social media identities, fabricated news sources, astroturfing campaigns — to spread disinformation, manufacture consensus, and obscure attribution.

### Identifying Synthetic Personas

```python
import anthropic
from datetime import datetime
from typing import List, Dict
import re

def analyze_account_for_synthetic_indicators(account_data: Dict) -> Dict:
    """
    Analyze a social media account for synthetic persona indicators
    """
    indicators = {
        'risk_level': 'LOW',
        'confidence': 0.0,
        'flags': [],
        'positive_signals': []
    }

    # 1. Account age vs. follower count ratio
    created_date = account_data.get('created_at')
    follower_count = account_data.get('followers_count', 0)
    following_count = account_data.get('following_count', 0)
    post_count = account_data.get('post_count', 0)

    if created_date:
        try:
            created = datetime.fromisoformat(created_date.replace('Z', '+00:00'))
            age_days = (datetime.now(created.tzinfo) - created).days

            if age_days < 30 and follower_count > 1000:
                indicators['flags'].append("Very new account with high follower count")

            posts_per_day = post_count / max(age_days, 1)
            if posts_per_day > 50:
                indicators['flags'].append(f"Extremely high posting rate: {posts_per_day:.0f}/day")

        except (ValueError, TypeError):
            pass

    # 2. Following/follower ratio
    if following_count > 0 and follower_count > 0:
        ratio = follower_count / following_count
        if following_count > 5000 and follower_count < 100:
            indicators['flags'].append("Follow-for-follow pattern: many following, few followers")

    # 3. Username patterns
    username = account_data.get('username', '')
    if re.search(r'\d{4,}$', username):
        indicators['flags'].append("Username ends with many digits — possible auto-generated")

    # 4. Profile completeness
    if not account_data.get('bio'):
        indicators['flags'].append("Empty bio")
    if not account_data.get('profile_image_url'):
        indicators['flags'].append("No profile image")
    elif 'default' in account_data.get('profile_image_url', '').lower():
        indicators['flags'].append("Default profile image")

    # 5. Geographic consistency
    location = account_data.get('location', '')
    if location and account_data.get('timezone'):
        # Check if stated location matches timezone
        pass  # Would require timezone lookup

    # Calculate risk
    flag_count = len(indicators['flags'])
    if flag_count == 0:
        indicators['risk_level'] = 'LOW'
        indicators['confidence'] = 0.1
    elif flag_count <= 2:
        indicators['risk_level'] = 'MEDIUM'
        indicators['confidence'] = 0.4
    elif flag_count <= 4:
        indicators['risk_level'] = 'HIGH'
        indicators['confidence'] = 0.7
    else:
        indicators['risk_level'] = 'VERY HIGH'
        indicators['confidence'] = 0.85

    return indicators


def detect_coordinated_behavior(accounts: List[Dict]) -> Dict:
    """
    Detect coordinated inauthentic behavior across a set of accounts
    """
    coordination_signals = {
        'detected': False,
        'confidence': 0.0,
        'patterns': [],
        'suspect_clusters': []
    }

    # 1. Creation date clustering
    creation_dates = []
    for account in accounts:
        created = account.get('created_at')
        if created:
            try:
                date = datetime.fromisoformat(created.replace('Z', '+00:00'))
                creation_dates.append(date)
            except (ValueError, TypeError):
                pass

    if len(creation_dates) > 3:
        # Sort and look for clusters
        creation_dates.sort()
        clusters = []
        current_cluster = [creation_dates[0]]

        for date in creation_dates[1:]:
            prev = current_cluster[-1]
            if (date - prev).days <= 7:
                current_cluster.append(date)
            else:
                if len(current_cluster) > 2:
                    clusters.append(current_cluster)
                current_cluster = [date]

        if len(current_cluster) > 2:
            clusters.append(current_cluster)

        if clusters:
            coordination_signals['patterns'].append(
                f"Creation date clustering: {len(clusters)} cluster(s) of accounts created within 7 days of each other"
            )
            coordination_signals['detected'] = True

    # 2. Content similarity
    contents = []
    for account in accounts:
        recent_posts = account.get('recent_posts', [])
        content = ' '.join([p.get('text', '') for p in recent_posts[:5]])
        if content.strip():
            contents.append({'account': account.get('username'), 'content': content})

    if len(contents) > 2:
        # Simple content similarity check — in production use proper NLP similarity
        # Check for identical phrases across accounts
        phrase_counts: Dict[str, List[str]] = {}
        for item in contents:
            words = item['content'].lower().split()
            # Generate 5-grams
            for i in range(len(words) - 4):
                phrase = ' '.join(words[i:i+5])
                if len(phrase) > 20:  # Skip short phrases
                    if phrase not in phrase_counts:
                        phrase_counts[phrase] = []
                    phrase_counts[phrase].append(item['account'])

        shared_phrases = {phrase: accounts for phrase, accounts in phrase_counts.items()
                        if len(accounts) > 1}

        if shared_phrases:
            coordination_signals['patterns'].append(
                f"Content sharing: {len(shared_phrases)} identical phrases shared across multiple accounts"
            )
            coordination_signals['detected'] = True

    # Calculate confidence
    if coordination_signals['detected']:
        coordination_signals['confidence'] = min(0.9, 0.3 * len(coordination_signals['patterns']))

    return coordination_signals


def ai_analyze_disinformation(content: str, context: str = "") -> str:
    """
    Use AI to analyze content for disinformation indicators
    """
    client = anthropic.Anthropic()

    prompt = f"""You are an analyst specializing in information integrity and disinformation detection.

CONTENT TO ANALYZE:
{content[:3000]}

{f'CONTEXT: {context}' if context else ''}

Analyze this content for potential disinformation indicators:

1. FACTUAL ACCURACY SIGNALS
   - Claims that are easily falsifiable
   - Claims that require extraordinary evidence
   - Internal logical inconsistencies
   - Misrepresentation of sources or statistics

2. PERSUASION TECHNIQUE INDICATORS
   - Emotional manipulation without substantive argument
   - False dichotomies or straw man arguments
   - Appeals to authority without verification
   - Urgency creation to bypass critical thinking

3. ATTRIBUTION ISSUES
   - Vague or unverifiable sourcing ("sources say", "experts confirm")
   - Source impersonation indicators
   - Missing context that changes meaning

4. NARRATIVE PATTERNS
   - Is this content consistent with known disinformation narratives?
   - Does this amplify existing conspiracy theories?
   - Who benefits if this content is widely believed?

5. VERDICT
   - Low/Medium/High risk of being disinformation
   - Key reasons for assessment
   - Recommended verification steps

Be analytical and evidence-based. Distinguish between content that is wrong, misleading, or deliberately deceptive. Note that high emotional content alone is not proof of disinformation."""

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1500,
        messages=[{"role": "user", "content": prompt}]
    )

    return response.content[0].text
```

---

## 24.4 Operational Security for Investigators

When investigators conduct sensitive research, they become potential targets for counter-surveillance by subjects who are alert to being investigated.

### OPSEC Threat Model

Before implementing OPSEC measures, define your threat model:

1. **Who are your adversaries?** State actors, organized crime, corporate security teams, and aggressive individuals all have different capabilities.

2. **What information are you protecting?** Your identity, your investigation targets, your sources, your methodology, your findings.

3. **What are the consequences of exposure?** Legal action, harassment, physical danger, source compromise, investigation compromise.

4. **What capabilities do adversaries have?** Technical monitoring, social engineering, legal subpoena, physical surveillance.

### Technical OPSEC Measures

```python
"""
OPSEC infrastructure for sensitive OSINT investigations
"""

import os
import subprocess
from typing import Dict, List

class InvestigationOPSEC:
    """
    OPSEC checklist and guidance for sensitive investigations
    """

    OPSEC_CHECKLIST = {
        'identity_protection': [
            "Use dedicated investigation identity (email, account) separate from personal identity",
            "Never access investigation accounts from personal devices or networks",
            "Use Tor Browser or VPN for sensitive browsing — understand capability limits of each",
            "Create research accounts on platforms using anonymous email",
            "Do not reuse passwords or usernames across investigation accounts",
        ],
        'device_security': [
            "Use dedicated investigation device or VM, isolated from personal data",
            "Keep investigation OS patched and updated",
            "Disable location services during investigation sessions",
            "Use encrypted storage for all investigation data",
            "Enable full-disk encryption on investigation devices",
        ],
        'network_security': [
            "Understand that VPNs protect from ISP monitoring but not from VPN provider",
            "Tor provides stronger anonymity but is slower and detectable as Tor traffic",
            "Tails OS is the gold standard for leave-no-trace investigation sessions",
            "Never log into personal accounts while using investigation network config",
            "Understand that browser fingerprinting can identify you beyond IP address",
        ],
        'source_protection': [
            "Document source communications in encrypted storage only",
            "Use Signal or other E2E encrypted communication for sensitive source contact",
            "Never include source-identifying details in notes stored on networked systems",
            "Use SecureDrop where available for document receipt from sources",
        ],
        'operational_security': [
            "Do not discuss investigation scope or targets outside secure channels",
            "Be aware of social engineering — subjects may probe your identity through third parties",
            "Compartmentalize: not everyone on the team needs all information",
            "Pre-plan what to say if asked about the investigation unexpectedly",
            "Document OPSEC exceptions and incidents when they occur",
        ]
    }

    def generate_opsec_assessment(self, threat_level: str) -> str:
        """Generate OPSEC recommendations based on threat level"""
        levels = {
            'LOW': ['identity_protection'],
            'MEDIUM': ['identity_protection', 'device_security', 'network_security'],
            'HIGH': ['identity_protection', 'device_security', 'network_security',
                    'source_protection', 'operational_security'],
            'CRITICAL': list(self.OPSEC_CHECKLIST.keys())
        }

        applicable_categories = levels.get(threat_level.upper(), ['identity_protection'])

        report = [
            f"# OPSEC Assessment — Threat Level: {threat_level.upper()}",
            "",
        ]

        for category in applicable_categories:
            category_name = category.replace('_', ' ').title()
            report.append(f"## {category_name}")
            report.append("")
            for item in self.OPSEC_CHECKLIST[category]:
                report.append(f"- [ ] {item}")
            report.append("")

        return '\n'.join(report)


def check_investigation_exposure(investigation_terms: List[str]) -> Dict:
    """
    Check if investigation is detectable through your own digital footprint
    What would you find if you searched for your own investigation activity?
    """
    exposure_risks = []

    # Check if you've searched investigation terms from your main browser
    # This is a reminder checklist — not an automated check
    checks = [
        "Have you searched investigation subjects from your personal Google account?",
        "Have you visited subject's LinkedIn profile while logged into your personal account?",
        "Have you posted about this investigation on personal social media?",
        "Have you discussed investigation details via unencrypted communication?",
        "Have you created documents about this investigation in personal cloud storage?",
    ]

    return {
        'manual_checks_required': checks,
        'investigation_terms': investigation_terms,
        'reminder': "OPSEC exposure assessment requires manual review — no automated tool can check everything"
    }
```

---

## 24.5 Counter-Intelligence: Detecting Who is Investigating You

Organizations and high-profile individuals can detect when they are being researched through several signals:

**LinkedIn profile views**: Professional profiles show who has viewed them. Sudden view patterns from competitors, journalists, or unknown entities can signal incoming investigation.

**Website traffic analysis**: Organizational websites log visitor IP addresses. Unusual access patterns — someone systematically visiting staff pages, executive bios, and contact information — can indicate reconnaissance.

**Honeypot documents and canary tokens**: Planting false-but-plausible documents with unique URLs allows detection of when those documents are accessed.

**Google Alerts**: Simple but effective — organizations can monitor their own name for emerging coverage.

```python
def setup_counter_intelligence_monitoring(organization: str, terms: List[str]) -> Dict:
    """
    Framework for organizational counter-intelligence monitoring
    Monitoring your own exposure in real-time
    """
    monitoring_config = {
        'organization': organization,
        'setup_date': datetime.now().isoformat()[:10],
        'monitoring_channels': []
    }

    # 1. Google Alerts (manual setup required)
    google_alert_terms = [
        f'"{organization}"',
        f'"{organization}" lawsuit',
        f'"{organization}" investigation',
        f'"{organization}" breach',
    ]
    monitoring_config['google_alert_terms'] = google_alert_terms

    # 2. News API monitoring
    monitoring_config['news_api_terms'] = [
        f'"{organization}"',
        *[f'"{term}"' for term in terms[:5]]
    ]

    # 3. Social media monitoring
    monitoring_config['social_monitoring'] = {
        'twitter_searches': [f'"{organization}"', f'@{organization.replace(" ", "")}'],
        'reddit_subreddits': ['investing', 'cybersecurity', 'news']
    }

    # 4. Canary token recommendations
    monitoring_config['canary_recommendations'] = [
        "Deploy canary tokens in sensitive documents (canarytokens.org)",
        "Monitor web server logs for systematic scraping patterns",
        "Set up Google Alerts for executive names",
        "Monitor domain registration for typosquatting (similar domain names)",
        "Check HaveIBeenPwned for domain breach exposure quarterly"
    ]

    return monitoring_config
```

---

## Summary

Adversarial OSINT is the mirror image of investigative OSINT. The same techniques that make investigators effective make organizations and individuals vulnerable. Understanding this dual nature drives both effective investigation and effective defense.

Three principles govern adversarial OSINT awareness:

**Exposure is cumulative**: No single data point creates vulnerability; it is the aggregation of public information that creates the actionable intelligence picture an adversary needs. Individual exposure is acceptable; combined exposure is the threat.

**Synthetic content is increasingly sophisticated**: AI-generated personas, deepfakes, and automated content farms make disinformation detection harder. Forensic verification requires systematic methodology, not visual inspection.

**OPSEC is a discipline, not a product**: VPNs, Tor, and burner phones are tools, not solutions. Operational security requires consistent discipline across every channel, every interaction, and every device — one slip creates attribution.

---

## Common Mistakes and Pitfalls

- **Assuming platform privacy settings protect you**: Public data can often be accessed regardless of settings through APIs, scrapers, or third-party aggregators
- **Using personal accounts for investigation research**: LinkedIn views, Google searches, and website visits leave traces that attribute to your account or IP
- **Ignoring the aggregation problem defensively**: Approving individually innocuous public disclosures without modeling their combined exposure
- **Under-estimating adversary sophistication**: Assuming only nation-states can conduct sophisticated OSINT underestimates organized crime, corporate intelligence, and advanced individuals
- **Disinformation overconfidence**: Confident identification of "obviously fake" content without methodical verification leads to false positives

---

## Further Reading

- EFF's Surveillance Self-Defense (ssd.eff.org) — practical OPSEC guides
- The Citizen Lab — adversarial surveillance research and reports
- Bellingcat OSINT methodology — disinformation investigation case studies
- First Draft — disinformation detection methodology
- Canary tokens (canarytokens.org) — free honeypot infrastructure
- DFRLab (Atlantic Council) — digital forensics and disinformation research
