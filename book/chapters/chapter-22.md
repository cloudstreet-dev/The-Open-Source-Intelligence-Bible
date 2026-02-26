# Chapter 22: Corporate Security and Due Diligence

## Learning Objectives

By the end of this chapter, you will be able to:
- Apply OSINT methodology to corporate intelligence and business due diligence
- Research counterparties, acquisition targets, and business partners using public data
- Identify beneficial ownership structures and conflicts of interest
- Conduct vendor and third-party risk assessments using open sources
- Build competitive intelligence workflows within legal boundaries
- Assess executive and key person risk for business relationships

---

## 22.1 Corporate Intelligence in the Business Context

Corporate security and due diligence are among the highest-value applications of OSINT methodology. The business stakes — multimillion-dollar acquisitions, vendor contracts with critical infrastructure access, executive hiring decisions — justify structured, systematic research that would be disproportionate for casual investigations.

OSINT-based corporate intelligence operates in three principal contexts:

**Pre-transaction due diligence**: Mergers, acquisitions, joint ventures, and large contract awards require understanding of counterparty reputation, financial health, litigation history, regulatory compliance record, and beneficial ownership structure. Surprises discovered post-closing are expensive; OSINT-based pre-close research is cheap by comparison.

**Ongoing third-party risk management**: Vendors, suppliers, and business partners with access to systems or sensitive data represent ongoing risk. Supply chain compromise, data breaches, and reputational contagion through association all originate from third-party exposure.

**Competitive intelligence**: Understanding competitors' capabilities, strategy, hiring patterns, and market positioning using publicly available information — a legitimate business practice distinct from industrial espionage.

---

## 22.2 Counterparty Research Framework

### Entity Research Methodology

```python
import requests
import json
from dataclasses import dataclass, field
from typing import List, Dict, Optional
from datetime import datetime

@dataclass
class CorporateProfile:
    """Structured corporate intelligence profile"""
    company_name: str
    jurisdiction: str
    registration_number: Optional[str] = None

    # Corporate structure
    parent_company: Optional[str] = None
    subsidiaries: List[str] = field(default_factory=list)
    ultimate_beneficial_owner: Optional[str] = None

    # Registration data
    incorporation_date: Optional[str] = None
    registered_address: Optional[str] = None
    company_status: Optional[str] = None

    # Key people
    directors: List[Dict] = field(default_factory=list)
    officers: List[Dict] = field(default_factory=list)

    # Financial indicators
    annual_revenue: Optional[str] = None
    employee_count: Optional[int] = None
    financial_filings: List[Dict] = field(default_factory=list)

    # Legal and regulatory
    litigation: List[Dict] = field(default_factory=list)
    regulatory_actions: List[Dict] = field(default_factory=list)
    sanctions_hits: List[Dict] = field(default_factory=list)

    # Reputational
    news_coverage: List[Dict] = field(default_factory=list)
    adverse_media: List[Dict] = field(default_factory=list)

    # Source tracking
    sources: List[Dict] = field(default_factory=list)
    research_date: str = field(default_factory=lambda: datetime.now().isoformat()[:10])


class CorporateDueDiligence:
    """
    Corporate due diligence research workflow using public data sources
    """

    def __init__(self, company_name: str, jurisdiction: str = "us"):
        self.company_name = company_name
        self.jurisdiction = jurisdiction
        self.profile = CorporateProfile(company_name=company_name, jurisdiction=jurisdiction)
        self.session = requests.Session()
        self.session.headers.update({'User-Agent': 'CorporateResearch/1.0'})

    def _add_source(self, source_name: str, url: str, data_type: str):
        """Track all sources consulted"""
        self.profile.sources.append({
            'source': source_name,
            'url': url,
            'data_type': data_type,
            'accessed': datetime.now().isoformat()[:19]
        })

    def search_opencorporates(self) -> List[Dict]:
        """Search OpenCorporates for company registration data"""
        results = []
        try:
            url = "https://api.opencorporates.com/v0.4/companies/search"
            params = {
                'q': self.company_name,
                'jurisdiction_code': self.jurisdiction,
                'per_page': 10
            }
            response = self.session.get(url, params=params, timeout=15)

            if response.status_code == 200:
                data = response.json()
                for item in data.get('results', {}).get('companies', []):
                    company = item.get('company', {})
                    results.append({
                        'name': company.get('name'),
                        'jurisdiction': company.get('jurisdiction_code'),
                        'company_number': company.get('company_number'),
                        'incorporation_date': company.get('incorporation_date'),
                        'company_type': company.get('company_type'),
                        'current_status': company.get('current_status'),
                        'registered_address': company.get('registered_address_in_full'),
                        'opencorporates_url': company.get('opencorporates_url')
                    })

            self._add_source('OpenCorporates', url, 'company_registration')

        except Exception as e:
            print(f"OpenCorporates search error: {e}")

        return results

    def get_company_officers(self, company_number: str, jurisdiction: str) -> List[Dict]:
        """Retrieve officers from OpenCorporates"""
        officers = []
        try:
            url = f"https://api.opencorporates.com/v0.4/companies/{jurisdiction}/{company_number}"
            response = self.session.get(url, params={'sparse': False}, timeout=15)

            if response.status_code == 200:
                data = response.json()
                company_data = data.get('results', {}).get('company', {})

                for officer_item in company_data.get('officers', []):
                    officer = officer_item.get('officer', {})
                    officers.append({
                        'name': officer.get('name'),
                        'position': officer.get('position'),
                        'start_date': officer.get('start_date'),
                        'end_date': officer.get('end_date'),
                        'occupation': officer.get('occupation'),
                        'nationality': officer.get('nationality'),
                        'inactive': officer.get('inactive', False)
                    })

            self._add_source('OpenCorporates Officers', url, 'corporate_officers')

        except Exception as e:
            print(f"Officer lookup error: {e}")

        return officers

    def search_sec_edgar(self) -> Dict:
        """Search SEC EDGAR for public company filings"""
        edgar_results = {
            'is_public': False,
            'cik': None,
            'filings': []
        }

        try:
            # Full-text company search
            url = "https://efts.sec.gov/LATEST/search-index?q=%22{}%22&dateRange=custom&startdt=2020-01-01&forms=10-K,DEF14A,8-K".format(
                self.company_name.replace(' ', '+')
            )

            # CIK lookup via company name
            cik_url = "https://www.sec.gov/cgi-bin/browse-edgar"
            params = {
                'company': self.company_name,
                'action': 'getcompany',
                'type': '10-K',
                'dateb': '',
                'owner': 'include',
                'count': 10,
                'output': 'atom'
            }
            response = self.session.get(cik_url, params=params, timeout=15)

            # Also try company facts API
            facts_search = "https://efts.sec.gov/LATEST/search-index?q=%22{}%22&forms=10-K".format(
                self.company_name.replace(' ', '+')
            )

            self._add_source('SEC EDGAR', cik_url, 'public_company_filings')

        except Exception as e:
            print(f"EDGAR search error: {e}")

        return edgar_results

    def search_litigation(self, state: str = None) -> List[Dict]:
        """Search for litigation using CourtListener API"""
        cases = []
        try:
            url = "https://www.courtlistener.com/api/rest/v3/search/"
            params = {
                'q': f'"{self.company_name}"',
                'type': 'o',  # opinions
                'order_by': 'score desc',
                'stat_Precedential': 'on',
                'count': 20
            }

            response = self.session.get(url, params=params, timeout=15)

            if response.status_code == 200:
                data = response.json()
                for result in data.get('results', []):
                    cases.append({
                        'case_name': result.get('caseName'),
                        'court': result.get('court'),
                        'date_filed': result.get('dateFiled'),
                        'docket_number': result.get('docketNumber'),
                        'url': result.get('absolute_url'),
                        'status': result.get('status')
                    })

            self._add_source('CourtListener', url, 'litigation')

        except Exception as e:
            print(f"Litigation search error: {e}")

        return cases

    def adverse_media_search(self) -> List[Dict]:
        """Search for adverse media coverage"""
        adverse_results = []

        # Adverse media search queries
        negative_terms = [
            'fraud', 'scandal', 'lawsuit', 'fine', 'penalty', 'violation',
            'bankruptcy', 'arrest', 'investigation', 'settlement', 'corrupt'
        ]

        for term in negative_terms[:5]:  # Limit API calls
            try:
                # News API search
                url = "https://newsapi.org/v2/everything"
                # Note: Requires NEWS_API_KEY in environment
                import os
                api_key = os.getenv('NEWS_API_KEY', '')

                if api_key:
                    params = {
                        'q': f'"{self.company_name}" {term}',
                        'language': 'en',
                        'sortBy': 'relevancy',
                        'pageSize': 5,
                        'apiKey': api_key
                    }
                    response = self.session.get(url, params=params, timeout=10)

                    if response.status_code == 200:
                        data = response.json()
                        for article in data.get('articles', []):
                            adverse_results.append({
                                'title': article.get('title'),
                                'source': article.get('source', {}).get('name'),
                                'published': article.get('publishedAt'),
                                'url': article.get('url'),
                                'description': article.get('description'),
                                'adverse_term': term
                            })

            except Exception as e:
                print(f"News search error for '{term}': {e}")

        self.profile.adverse_media = adverse_results
        return adverse_results

    def check_sanctions(self, names_to_check: List[str]) -> List[Dict]:
        """Check names against OpenSanctions"""
        hits = []

        for name in names_to_check:
            try:
                url = "https://api.opensanctions.org/match/default"
                payload = {
                    "queries": {
                        "entity": {
                            "schema": "Company",
                            "properties": {
                                "name": [name]
                            }
                        }
                    }
                }

                response = self.session.post(
                    url,
                    json=payload,
                    timeout=15
                )

                if response.status_code == 200:
                    data = response.json()
                    responses = data.get('responses', {})
                    entity_results = responses.get('entity', {}).get('results', [])

                    for result in entity_results:
                        if result.get('score', 0) > 0.7:
                            hits.append({
                                'queried_name': name,
                                'matched_name': result.get('caption'),
                                'score': result.get('score'),
                                'datasets': result.get('datasets', []),
                                'schema': result.get('schema')
                            })

            except Exception as e:
                print(f"Sanctions check error: {e}")

        self.profile.sanctions_hits = hits
        return hits

    def research_key_executives(self, executives: List[str]) -> List[Dict]:
        """Research key executives using multiple sources"""
        exec_profiles = []

        for exec_name in executives:
            profile = {
                'name': exec_name,
                'linkedin_found': False,
                'regulatory_record': None,
                'litigation_involvement': [],
                'other_board_positions': [],
                'news_coverage': []
            }

            # 1. FINRA BrokerCheck
            try:
                bc_url = "https://api.brokercheck.finra.org/search/individual"
                params = {
                    'query': exec_name,
                    'hl': 'true',
                    'includePrevious': 'true',
                    'exactMatch': 'false',
                    'primary': 'true',
                    'type': 'individual',
                    'start': 0,
                    'count': 3
                }
                response = self.session.get(bc_url, params=params,
                                           headers={'Referer': 'https://brokercheck.finra.org/'},
                                           timeout=10)

                if response.status_code == 200:
                    data = response.json()
                    hits = data.get('hits', {}).get('hits', [])
                    if hits:
                        source = hits[0].get('_source', {})
                        profile['regulatory_record'] = {
                            'source': 'FINRA BrokerCheck',
                            'crd_number': source.get('ind_source_id'),
                            'has_disclosures': source.get('ind_bc_scope', '0') != '0',
                            'employer': [e.get('empl_nm') for e in source.get('ind_pc_employers', [])]
                        }
            except Exception:
                pass

            # 2. OpenCorporates director search
            try:
                oc_url = "https://api.opencorporates.com/v0.4/officers/search"
                params = {'q': exec_name, 'per_page': 10}
                response = self.session.get(oc_url, params=params, timeout=10)

                if response.status_code == 200:
                    data = response.json()
                    for officer_item in data.get('results', {}).get('officers', []):
                        officer = officer_item.get('officer', {})
                        company = officer.get('company', {})
                        profile['other_board_positions'].append({
                            'company': company.get('name'),
                            'jurisdiction': company.get('jurisdiction_code'),
                            'position': officer.get('position'),
                            'start_date': officer.get('start_date'),
                            'inactive': officer.get('inactive', False)
                        })
            except Exception:
                pass

            exec_profiles.append(profile)

        return exec_profiles

    def compile_due_diligence_report(self) -> str:
        """Generate structured due diligence report"""
        report = [
            f"# Corporate Due Diligence Report",
            f"**Subject**: {self.company_name}",
            f"**Jurisdiction**: {self.jurisdiction.upper()}",
            f"**Research Date**: {self.profile.research_date}",
            f"**Methodology**: Open source intelligence; public records only",
            "",
            "---",
            "",
            "## Executive Summary",
            "",
            "| Category | Status | Risk Level |",
            "|---|---|---|",
            f"| Corporate Registration | {'Verified' if self.profile.registration_number else 'Not found'} | - |",
            f"| Sanctions/Watchlists | {'HITS FOUND' if self.profile.sanctions_hits else 'No hits'} | {'HIGH' if self.profile.sanctions_hits else 'Low'} |",
            f"| Litigation | {len(self.profile.litigation)} cases found | {'Elevated' if len(self.profile.litigation) > 3 else 'Normal'} |",
            f"| Adverse Media | {len(self.profile.adverse_media)} results | {'Elevated' if len(self.profile.adverse_media) > 5 else 'Normal'} |",
            "",
            "---",
            "",
            "## Corporate Structure",
            "",
        ]

        if self.profile.incorporation_date:
            report.append(f"- **Incorporated**: {self.profile.incorporation_date}")
        if self.profile.registered_address:
            report.append(f"- **Registered Address**: {self.profile.registered_address}")
        if self.profile.company_status:
            report.append(f"- **Status**: {self.profile.company_status}")
        if self.profile.parent_company:
            report.append(f"- **Parent Entity**: {self.profile.parent_company}")

        if self.profile.sanctions_hits:
            report.extend([
                "",
                "## ⚠️ SANCTIONS / WATCHLIST HITS",
                ""
            ])
            for hit in self.profile.sanctions_hits:
                report.append(f"- **{hit['queried_name']}** matched `{hit['matched_name']}` (score: {hit['score']:.2f})")
                report.append(f"  Datasets: {', '.join(hit['datasets'])}")

        if self.profile.litigation:
            report.extend(["", "## Litigation History", ""])
            for case in self.profile.litigation[:10]:
                report.append(f"- **{case.get('case_name', 'Unknown')}** ({case.get('date_filed', 'Date unknown')})")
                report.append(f"  Court: {case.get('court', 'Unknown')} | Docket: {case.get('docket_number', 'N/A')}")

        if self.profile.adverse_media:
            report.extend(["", "## Adverse Media", ""])
            for item in self.profile.adverse_media[:10]:
                report.append(f"- [{item.get('title', 'No title')}]({item.get('url', '#')})")
                report.append(f"  {item.get('source', 'Unknown')} — {item.get('published', 'Unknown date')}")

        report.extend([
            "",
            "## Sources Consulted",
            ""
        ])
        for source in self.profile.sources:
            report.append(f"- **{source['source']}** ({source['data_type']}) — accessed {source['accessed'][:10]}")

        report.extend([
            "",
            "---",
            "",
            "*This report is based on publicly available information only. It does not constitute a comprehensive due diligence review and should be supplemented with professional legal, financial, and regulatory analysis as appropriate.*"
        ])

        return '\n'.join(report)
```

---

## 22.3 Vendor Risk Assessment

Third-party vendor risk assessment applies OSINT methodology to the ongoing problem of supply chain security. Every vendor with system access, data access, or operational dependencies represents risk.

### Vendor Risk Scoring Framework

```python
from dataclasses import dataclass, field
from typing import Dict, List
import json

@dataclass
class VendorRiskProfile:
    """Risk assessment profile for a vendor"""
    vendor_name: str
    vendor_type: str  # e.g., "SaaS", "IT Services", "Data Processor"
    data_access_level: str  # "None", "Restricted", "Sensitive", "Critical"

    # Risk factors — each scored 0-10
    financial_stability: float = 5.0
    regulatory_compliance: float = 5.0
    security_posture: float = 5.0
    reputation_risk: float = 5.0
    concentration_risk: float = 5.0

    # Supporting evidence
    financial_flags: List[str] = field(default_factory=list)
    compliance_flags: List[str] = field(default_factory=list)
    security_flags: List[str] = field(default_factory=list)
    reputation_flags: List[str] = field(default_factory=list)

    @property
    def overall_risk_score(self) -> float:
        """Weighted risk score — lower is better"""
        weights = {
            'financial_stability': 0.2,
            'regulatory_compliance': 0.25,
            'security_posture': 0.3,
            'reputation_risk': 0.15,
            'concentration_risk': 0.1
        }
        score = (
            self.financial_stability * weights['financial_stability'] +
            self.regulatory_compliance * weights['regulatory_compliance'] +
            self.security_posture * weights['security_posture'] +
            self.reputation_risk * weights['reputation_risk'] +
            self.concentration_risk * weights['concentration_risk']
        )
        return score

    @property
    def risk_rating(self) -> str:
        score = self.overall_risk_score
        if score <= 3:
            return "LOW"
        elif score <= 6:
            return "MEDIUM"
        elif score <= 8:
            return "HIGH"
        else:
            return "CRITICAL"


class VendorRiskAssessor:
    """
    Automated vendor risk assessment using open source data
    """

    SECURITY_BREACH_KEYWORDS = [
        'data breach', 'hack', 'ransomware', 'cyberattack',
        'unauthorized access', 'data leak', 'security incident'
    ]

    FINANCIAL_RISK_KEYWORDS = [
        'bankruptcy', 'insolvency', 'acquisition', 'layoffs',
        'restructuring', 'funding cut', 'shutdown'
    ]

    COMPLIANCE_KEYWORDS = [
        'FTC fine', 'GDPR penalty', 'HIPAA violation', 'SEC enforcement',
        'regulatory action', 'consent decree', 'settlement'
    ]

    def __init__(self, vendor_name: str, vendor_type: str, data_access: str):
        self.profile = VendorRiskProfile(
            vendor_name=vendor_name,
            vendor_type=vendor_type,
            data_access_level=data_access
        )

    def check_breach_history(self) -> None:
        """Check HaveIBeenPwned for breach history"""
        import requests

        # Check if vendor domain appears in known breaches
        # HIBP API v3 requires API key for domain search
        # Alternatively: search breach databases for vendor name

        vendor_domain = self.profile.vendor_name.lower().replace(' ', '') + '.com'

        try:
            # Check breach data sources
            # In production: HIBP Enterprise API, breach databases
            # For now: flag for manual check
            self.profile.security_flags.append(
                f"Manual check required: HIBP domain search for {vendor_domain}"
            )
        except Exception:
            pass

    def search_sec_data_breaches(self) -> None:
        """Check SEC 8-K filings for cybersecurity incident disclosures"""
        import requests

        try:
            # Since 2023, SEC requires public companies to disclose material cybersecurity incidents via 8-K
            url = "https://efts.sec.gov/LATEST/search-index"
            params = {
                'q': f'"{self.profile.vendor_name}" cybersecurity',
                'forms': '8-K',
                'dateRange': 'custom',
                'startdt': '2023-01-01'
            }

            response = requests.get(url, params=params, timeout=15)
            if response.status_code == 200:
                data = response.json()
                hits = data.get('hits', {}).get('hits', [])
                if hits:
                    self.profile.security_flags.append(
                        f"SEC 8-K cybersecurity disclosures found: {len(hits)} filings"
                    )
                    self.profile.security_posture = min(10, self.profile.security_posture + 2)

        except Exception as e:
            pass

    def check_shodan_exposure(self, ip_or_domain: str) -> None:
        """Check vendor's public-facing attack surface via Shodan"""
        import os, requests

        api_key = os.getenv('SHODAN_API_KEY', '')
        if not api_key:
            self.profile.security_flags.append("Shodan check skipped — no API key")
            return

        try:
            url = f"https://api.shodan.io/dns/resolve"
            params = {'hostnames': ip_or_domain, 'key': api_key}
            response = requests.get(url, params=params, timeout=10)

            if response.status_code == 200:
                ip_data = response.json()
                ip = ip_data.get(ip_or_domain)

                if ip:
                    host_url = f"https://api.shodan.io/shodan/host/{ip}"
                    host_resp = requests.get(host_url, params={'key': api_key}, timeout=10)

                    if host_resp.status_code == 200:
                        host_data = host_resp.json()
                        open_ports = host_data.get('ports', [])
                        vulns = host_data.get('vulns', {})

                        if vulns:
                            self.profile.security_flags.append(
                                f"Known CVEs in Shodan: {', '.join(list(vulns.keys())[:5])}"
                            )
                            self.profile.security_posture = min(10, self.profile.security_posture + len(vulns))

                        sensitive_ports = set(open_ports) & {21, 23, 3389, 5900, 6379, 27017, 9200}
                        if sensitive_ports:
                            self.profile.security_flags.append(
                                f"Sensitive ports exposed: {sensitive_ports}"
                            )
                            self.profile.security_posture = min(10, self.profile.security_posture + 2)

        except Exception as e:
            pass

    def generate_risk_report(self) -> str:
        """Generate vendor risk assessment report"""
        p = self.profile

        report = [
            f"# Vendor Risk Assessment: {p.vendor_name}",
            f"**Vendor Type**: {p.vendor_type}",
            f"**Data Access Level**: {p.data_access_level}",
            f"**Overall Risk Rating**: {p.risk_rating} ({p.overall_risk_score:.1f}/10)",
            "",
            "## Risk Scores",
            "",
            "| Risk Category | Score (0-10 = low-high risk) |",
            "|---|---|",
            f"| Financial Stability | {p.financial_stability:.1f} |",
            f"| Regulatory Compliance | {p.regulatory_compliance:.1f} |",
            f"| Security Posture | {p.security_posture:.1f} |",
            f"| Reputation Risk | {p.reputation_risk:.1f} |",
            f"| Concentration Risk | {p.concentration_risk:.1f} |",
            f"| **Overall** | **{p.overall_risk_score:.1f}** |",
        ]

        if p.security_flags:
            report.extend(["", "## Security Findings", ""])
            for flag in p.security_flags:
                report.append(f"- {flag}")

        if p.compliance_flags:
            report.extend(["", "## Compliance Findings", ""])
            for flag in p.compliance_flags:
                report.append(f"- {flag}")

        if p.financial_flags:
            report.extend(["", "## Financial Findings", ""])
            for flag in p.financial_flags:
                report.append(f"- {flag}")

        report.extend([
            "",
            "## Recommended Actions",
            ""
        ])

        if p.risk_rating == "CRITICAL":
            report.append("- **Immediate review required** — Escalate to CISO and Legal")
            report.append("- Consider contract review and potential off-boarding")
        elif p.risk_rating == "HIGH":
            report.append("- Schedule vendor security review within 30 days")
            report.append("- Review contract terms and data processing agreement")
            report.append("- Increase monitoring frequency")
        elif p.risk_rating == "MEDIUM":
            report.append("- Include in next scheduled vendor review cycle")
            report.append("- Request security questionnaire update")
        else:
            report.append("- Continue standard monitoring cadence")

        return '\n'.join(report)
```

---

## 22.4 Competitive Intelligence

Competitive intelligence (CI) uses publicly available information to understand competitor capabilities, strategy, and market position. It is distinct from industrial espionage (illegal) or social engineering (unethical).

### Legal CI Sources

**Job postings**: A competitor's job postings reveal technology stack, planned expansion, strategic priorities, and organizational gaps. A company posting ten machine learning engineering roles signals an AI initiative; posting multiple sales roles in a new geography signals market expansion.

**Patent filings**: USPTO Patent Full-Text Database reveals R&D investment directions years before product announcements.

**Conference presentations and technical papers**: Engineers and researchers publish methodologies, architectures, and findings at conferences — comprehensive public signals of technical capability.

**SEC filings**: Public companies disclose revenue by segment, significant customers, risk factors, and strategic priorities in 10-K and 10-Q filings.

**Press releases and news**: Product announcements, partnership agreements, customer wins.

**LinkedIn**: Hiring patterns, executive movements, org chart reconstruction.

```python
import anthropic
import json

def analyze_competitor_job_postings(company_name: str, job_data: List[Dict]) -> str:
    """
    Use AI to analyze competitor job postings for strategic intelligence
    """
    client = anthropic.Anthropic()

    # Format job posting data
    jobs_text = "\n\n".join([
        f"Title: {job.get('title', 'Unknown')}\n"
        f"Department: {job.get('department', 'Unknown')}\n"
        f"Location: {job.get('location', 'Unknown')}\n"
        f"Posted: {job.get('date_posted', 'Unknown')}\n"
        f"Requirements: {job.get('requirements', 'Not available')[:500]}"
        for job in job_data
    ])

    prompt = f"""You are a competitive intelligence analyst examining job posting data for strategic insights.

COMPANY: {company_name}
JOB POSTING DATA ({len(job_data)} postings):
{jobs_text[:5000]}

Analyze these job postings and provide:

1. TECHNOLOGY STACK SIGNALS
   - What technologies and platforms are they building with?
   - What technical capabilities are they building?
   - What tools are required across roles?

2. STRATEGIC INITIATIVES
   - What new products, features, or markets does the hiring suggest?
   - What business problems are they trying to solve?
   - What organizational structures are being built?

3. HIRING VELOCITY AND PATTERNS
   - Which departments are growing fastest?
   - What does the geographic distribution suggest?
   - What seniority levels are being targeted?

4. COMPETITIVE IMPLICATIONS
   - What should a competitor pay attention to?
   - What capabilities are they building that don't yet exist in their public products?

Be specific and evidence-based. Cite specific job titles or requirements that support each inference.
Distinguish between direct evidence (explicitly stated) and inference (reasonably derived).
All information is from public job postings — this is legitimate competitive intelligence."""

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2000,
        messages=[{"role": "user", "content": prompt}]
    )

    return response.content[0].text


def analyze_patent_filings(company_name: str, patent_data: List[Dict]) -> str:
    """
    Analyze USPTO patent filings for R&D direction intelligence
    """
    client = anthropic.Anthropic()

    patents_text = "\n\n".join([
        f"Patent: {p.get('title', 'Unknown')}\n"
        f"Filed: {p.get('filing_date', 'Unknown')}\n"
        f"Abstract: {p.get('abstract', 'Not available')[:400]}\n"
        f"CPC Classification: {p.get('cpc_codes', 'Unknown')}"
        for p in patent_data
    ])

    prompt = f"""You are analyzing patent filings for competitive intelligence purposes.

COMPANY: {company_name}
PATENT FILINGS:
{patents_text[:5000]}

Analyze these patents and provide:

1. TECHNOLOGY FOCUS AREAS
   - What technical domains are they investing in?
   - What problems are they trying to solve?

2. R&D TRAJECTORY
   - How has their focus changed over time?
   - What innovations appear to be approaching commercial readiness?

3. COMPETITIVE POSITIONING
   - What defensible advantages are they building?
   - What technologies might they be planning to license or productize?

4. TIMELINE SIGNALS
   - Which inventions filed 1-3 years ago might become products soon?

Note: Patents are public records. This analysis is standard competitive intelligence practice.
Be specific and cite patent titles or abstracts to support your analysis."""

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1500,
        messages=[{"role": "user", "content": prompt}]
    )

    return response.content[0].text
```

### Ethical Boundaries in Competitive Intelligence

The line between legitimate CI and illegal/unethical methods:

| Legitimate CI | Ethical gray area | Clearly wrong |
|---|---|---|
| Public job postings | Systematically creating fake LinkedIn profiles to connect with employees | Hiring competitor employees with intent to extract trade secrets |
| SEC filings | Eliciting strategy from employees at conferences via misdirection | Dumpster diving at competitor offices |
| Patent filings | Posing as journalist to interview executives | Wiretapping |
| Public conference talks | Anonymous tip lines targeting specific competitor employees | Social engineering admin access |
| News monitoring | Paying informants inside competitor | Bribery |

---

## 22.5 Executive and Key Person Risk

Executive background research is a core component of business due diligence — particularly for board appointments, C-suite hiring, major contract counterparties, and investment decisions.

```python
def executive_background_research(name: str, company: str, role: str) -> Dict:
    """
    Comprehensive executive background research using public sources
    """
    import requests

    research = {
        'subject': name,
        'current_role': f"{role} at {company}",
        'sources_checked': [],
        'findings': [],
        'flags': [],
        'risk_indicators': []
    }

    session = requests.Session()

    # 1. FINRA BrokerCheck
    try:
        response = session.get(
            "https://api.brokercheck.finra.org/search/individual",
            params={
                'query': name,
                'hl': 'true',
                'includePrevious': 'true',
                'type': 'individual',
                'start': 0,
                'count': 5
            },
            headers={'Referer': 'https://brokercheck.finra.org/'},
            timeout=10
        )

        if response.status_code == 200:
            data = response.json()
            hits = data.get('hits', {}).get('hits', [])
            research['sources_checked'].append('FINRA BrokerCheck')

            for hit in hits:
                source = hit.get('_source', {})
                if source.get('ind_bc_scope', '0') != '0':
                    research['flags'].append({
                        'source': 'FINRA BrokerCheck',
                        'flag': 'Regulatory disclosures on record',
                        'detail': f"CRD: {source.get('ind_source_id')}",
                        'severity': 'HIGH'
                    })
                else:
                    research['findings'].append({
                        'source': 'FINRA',
                        'finding': f"Registered broker-dealer: {source.get('ind_source_id')}, no disclosures"
                    })
    except Exception:
        pass

    # 2. SEC enforcement actions
    try:
        sec_url = "https://efts.sec.gov/LATEST/search-index"
        response = session.get(
            sec_url,
            params={
                'q': f'"{name}"',
                'forms': 'AAER,33-8986,34-44969',
                'dateRange': 'custom',
                'startdt': '2000-01-01'
            },
            timeout=10
        )
        research['sources_checked'].append('SEC Enforcement Actions')

        if response.status_code == 200:
            data = response.json()
            hits_count = data.get('hits', {}).get('total', {}).get('value', 0)
            if hits_count > 0:
                research['flags'].append({
                    'source': 'SEC',
                    'flag': f'Appears in {hits_count} SEC enforcement document(s)',
                    'severity': 'HIGH',
                    'detail': 'Manual review required'
                })
    except Exception:
        pass

    # 3. OpenCorporates officer search
    try:
        oc_response = session.get(
            "https://api.opencorporates.com/v0.4/officers/search",
            params={'q': name, 'per_page': 20},
            timeout=10
        )
        research['sources_checked'].append('OpenCorporates')

        if oc_response.status_code == 200:
            oc_data = oc_response.json()
            officers = oc_data.get('results', {}).get('officers', [])

            # Look for dissolved companies in portfolio
            dissolved_count = 0
            for officer_item in officers:
                officer = officer_item.get('officer', {})
                company = officer.get('company', {})
                if company.get('current_status') in ['Dissolved', 'Inactive', 'Revoked']:
                    dissolved_count += 1

            research['findings'].append({
                'source': 'OpenCorporates',
                'finding': f"Associated with {len(officers)} companies ({dissolved_count} dissolved/inactive)"
            })

            if dissolved_count > 3:
                research['risk_indicators'].append(
                    f"High number of dissolved company associations: {dissolved_count}"
                )
    except Exception:
        pass

    # 4. Federal court records
    try:
        cl_response = session.get(
            "https://www.courtlistener.com/api/rest/v3/search/",
            params={
                'q': f'"{name}"',
                'type': 'r',
                'order_by': 'score desc',
                'count': 10
            },
            timeout=10
        )
        research['sources_checked'].append('CourtListener (Federal)')

        if cl_response.status_code == 200:
            cl_data = cl_response.json()
            case_count = cl_data.get('count', 0)
            if case_count > 0:
                research['findings'].append({
                    'source': 'CourtListener',
                    'finding': f"Appears in {case_count} federal court filings"
                })
    except Exception:
        pass

    # Calculate summary risk level
    high_flags = [f for f in research['flags'] if f.get('severity') == 'HIGH']
    if len(high_flags) >= 2:
        research['overall_risk'] = 'HIGH'
    elif len(high_flags) == 1 or len(research['risk_indicators']) > 0:
        research['overall_risk'] = 'MEDIUM'
    else:
        research['overall_risk'] = 'LOW'

    return research
```

---

## 22.6 Beneficial Ownership and Shell Structure Analysis

Corporate opacity through multi-layered ownership structures is a risk factor in due diligence contexts — it can indicate tax optimization (legal), regulatory arbitrage (concerning), or outright fraud (disqualifying).

Beneficial ownership research uses:

- **FinCEN Beneficial Ownership Database**: New as of 2024, companies are required to report beneficial owners to FinCEN. The database is not public, but certain authorized parties can access it — an important development in corporate transparency.

- **OpenCorporates relationship network**: Directors and officers serve as pivot points — the same individual appearing as director across many companies in different jurisdictions is a significant indicator.

- **State Secretary of State databases**: Annual reports often list registered agents, officers, and sometimes beneficial owners.

- **Offshore leak databases**: ICIJ's Panama Papers, Pandora Papers, and Offshore Leaks databases contain ownership data from leaked corporate registries.

```python
def check_icij_offshore_leaks(name: str) -> List[Dict]:
    """
    Search ICIJ Offshore Leaks Database for an entity or person
    """
    import requests

    results = []
    try:
        url = "https://offshoreleaks.icij.org/api/search"
        params = {
            'q': name,
            'c': '',  # country filter
            'j': '',  # jurisdiction filter
            'cat': '1'
        }

        response = requests.get(url, params=params, timeout=15)

        if response.status_code == 200:
            data = response.json()
            for node in data.get('nodes', []):
                results.append({
                    'name': node.get('name'),
                    'type': node.get('type'),  # entity, officer, address, intermediary
                    'jurisdiction': node.get('jurisdiction'),
                    'datasets': node.get('datasets', []),
                    'link': f"https://offshoreleaks.icij.org/nodes/{node.get('id')}"
                })

    except Exception as e:
        print(f"ICIJ search error: {e}")

    return results
```

---

## Summary

Corporate due diligence and vendor risk assessment represent high-stakes applications of OSINT methodology. The same investigative techniques used in journalism, law enforcement, and academic research translate directly to business contexts — background investigations, entity research, litigation history, adverse media screening — with the addition of specialized commercial databases and regulatory filing systems.

Three principles govern effective corporate intelligence:

**Comprehensiveness over speed**: Unlike breaking news investigations, due diligence timelines allow systematic coverage of all relevant sources. Missed red flags are expensive.

**Source diversity**: Any single source can be incomplete, outdated, or manipulated. Beneficial ownership conclusions require corroboration across multiple jurisdictions and filing systems.

**Appropriate scope**: Competitive intelligence that stays within legal and ethical boundaries — public filings, job postings, published research — provides substantial value without the legal and reputational risk of aggressive methods.

---

## Common Mistakes and Pitfalls

- **Stopping at the presenting entity**: Due diligence on the contracting entity without tracing beneficial ownership misses the most significant risks
- **Recency bias**: Recent clean record doesn't erase historical problems; search across multiple years
- **Name matching without verification**: John Smith matches are meaningless without DOB, address, or other confirmatory data
- **Ignoring affiliate risk**: A clean primary company with troubled subsidiaries or affiliates shares risk
- **Treating CI as license to surveil**: Competitive intelligence has clear legal limits; exceeding them creates liability
- **Inadequate adverse media search**: Keyword searches for company name alone miss articles that reference executives or subsidiaries

---

## Further Reading

- ACFE (Association of Certified Fraud Examiners) due diligence guidelines
- Society of Competitive Intelligence Professionals (SCIP) ethical standards
- FinCEN Beneficial Ownership reporting rules (31 CFR Part 1010)
- ICIJ Offshore Leaks Database — icij.org/investigations
- OpenCorporates developer documentation — api.opencorporates.com
- SEC EDGAR full-text search — efts.sec.gov
