# Chapter 7: Public Records and Data Aggregation

## Learning Objectives

By the end of this chapter, you will be able to:
- Navigate the major categories of public records across U.S. and international jurisdictions
- Apply systematic public records research methodology
- Use commercial aggregation platforms effectively and critically
- Build automated public records collection workflows
- Triangulate across records to verify and enrich findings
- Understand the legal and quality constraints of different record types

---

## 7.1 The Public Records Foundation

Public records represent the most legally unambiguous and often the most authoritative OSINT sources. These records exist because governments require their creation as a condition of legal recognition — incorporation, property ownership, professional licensing, litigation, regulatory compliance — and most jurisdictions have determined that these records should be accessible to the public.

The tradeoff: public records tend to be authoritative but narrow. A property record confirms ownership but not what the owner does there. A court record confirms a legal action occurred but may not describe its background. A business registration confirms an entity exists but may not reveal who actually controls it.

The OSINT practitioner's job is to combine records from multiple sources to build a picture that no single record could provide. This aggregation process is where analytical skill matters most.

---

## 7.2 Corporate and Business Records

### U.S. Secretary of State Filings

Every U.S. state maintains a business registry through its Secretary of State (or equivalent) office. Business entities — corporations, LLCs, partnerships, sole proprietorships — must file formation documents and periodic reports.

**What's in the records**:
- Entity name and any registered alternate names (DBAs)
- Entity type (LLC, corporation, partnership)
- State and date of incorporation/formation
- Registered agent name and address
- Officer and director names (varies by state and entity type)
- Annual report filings (may include officer updates)
- Dissolution or status changes

**Investigative value**:
- Identifies who is legally connected to a business entity
- Reveals historical officers and directors (useful for investigating departed executives)
- Shows entity structure (is Company A a subsidiary of Company B?)
- Registered agent address can link otherwise unconnected entities that share an agent

**Access**:
- Each state has its own system, varying in search capability and completeness
- Most have free web search interfaces
- Some require per-record fees
- National aggregators like OpenCorporates and BizApedia compile multi-state search

```python
import requests
from bs4 import BeautifulSoup
import json

def search_opencorporates(company_name, jurisdiction=None, page=1):
    """Search OpenCorporates API for company information"""
    params = {
        'q': company_name,
        'format': 'json',
        'page': page
    }
    if jurisdiction:
        params['jurisdiction_code'] = jurisdiction

    response = requests.get(
        'https://api.opencorporates.com/v0.4/companies/search',
        params=params,
        timeout=30
    )

    if response.status_code != 200:
        return {'error': f'HTTP {response.status_code}'}

    data = response.json()
    companies = data.get('results', {}).get('companies', [])

    return [{
        'name': c['company']['name'],
        'jurisdiction': c['company']['jurisdiction_code'],
        'company_number': c['company']['company_number'],
        'incorporation_date': c['company']['incorporation_date'],
        'company_type': c['company']['company_type'],
        'status': c['company']['current_status'],
        'registered_address': c['company'].get('registered_address', {}).get('in_full', ''),
        'opencorporates_url': c['company']['opencorporates_url'],
    } for c in companies]

def get_company_officers(jurisdiction, company_number):
    """Get officer/director history for a company"""
    url = f"https://api.opencorporates.com/v0.4/companies/{jurisdiction}/{company_number}"
    params = {'api_token': 'YOUR_TOKEN', 'sparse': False}

    response = requests.get(url, params=params)
    if response.status_code == 200:
        data = response.json()['results']['company']
        return {
            'name': data['name'],
            'officers': [
                {
                    'name': o['officer']['name'],
                    'position': o['officer']['position'],
                    'start_date': o['officer']['start_date'],
                    'end_date': o['officer']['end_date'],
                }
                for o in data.get('officers', [])
            ]
        }
```

### SEC EDGAR

The Securities and Exchange Commission's EDGAR database contains filings from all public U.S. companies. For corporate investigations, EDGAR is one of the most valuable free public databases in existence.

**Key filing types**:

| Form | Content |
|------|---------|
| 10-K | Annual report — comprehensive company overview, risk factors, financials |
| 10-Q | Quarterly report — updated financial data |
| 8-K | Current report — material events (executive changes, major contracts, legal actions) |
| DEF 14A | Proxy statement — executive compensation, board composition, related party transactions |
| S-1 | IPO registration — company history, business description, financials |
| Form 4 | Insider trading — officer and director share purchases/sales |
| 13D/13G | Large shareholder disclosures — who owns >5% of a company |
| SC 13E-3 | Going-private transactions |

**EDGAR full-text search**:

```python
import requests
from datetime import datetime

class EDGARIntelligence:
    """EDGAR document analysis for corporate intelligence"""

    BASE_URL = "https://efts.sec.gov/LATEST/search-index"
    DATA_URL = "https://data.sec.gov"

    def search_filings(self, query, form_type=None, date_from=None, date_to=None):
        """Search EDGAR full-text search"""
        params = {
            'q': f'"{query}"',
            'dateRange': 'custom',
            'startdt': date_from or '2010-01-01',
            'enddt': date_to or datetime.now().strftime('%Y-%m-%d'),
        }
        if form_type:
            params['forms'] = form_type

        response = requests.get(
            "https://efts.sec.gov/LATEST/search-index?q=%22{}&forms={}&dateRange=custom&startdt={}".format(
                query, form_type or '', params['startdt']
            )
        )

        # Use the EDGAR EFTS API
        response = requests.get(
            "https://efts.sec.gov/LATEST/search-index",
            params={'q': f'"{query}"', 'forms': form_type or '10-K,DEF 14A,8-K'}
        )

        return response.json() if response.status_code == 200 else {}

    def get_company_facts(self, cik):
        """Get all SEC submissions for a company by CIK"""
        response = requests.get(
            f"{self.DATA_URL}/submissions/CIK{str(cik).zfill(10)}.json"
        )
        return response.json() if response.status_code == 200 else {}

    def search_proxy_statements(self, company_name):
        """Search proxy statements for executive compensation and related party data"""
        # Proxy statements (DEF 14A) contain related party transactions
        # which often reveal undisclosed relationships
        cik_response = requests.get(
            f"https://www.sec.gov/cgi-bin/browse-edgar?company={company_name}&CIK=&type=DEF+14A&dateb=&owner=include&count=40&search_text=&action=getcompany"
        )
        return cik_response.text  # Parse HTML for results

edgar = EDGARIntelligence()
```

### UCC Financing Statements

Uniform Commercial Code (UCC) filings are largely overlooked outside financial investigations, but they are extraordinarily revealing about financial relationships.

When a business pledges assets as collateral for a loan, the lender files a UCC-1 financing statement with the state. These filings are public and reveal:

- Who has lent money to whom
- What assets serve as collateral
- When business relationships were established
- Financial distress indicators (multiple lenders, unusual collateral)

Most Secretary of State offices maintain searchable UCC filing databases. Commercial services aggregate UCC data nationally.

### International Corporate Records

For investigations with international dimensions:

**UK Companies House**: Among the world's most open business registries. Free API access. All filings including accounts are public. Director appointments, resignations, and addresses are fully public.

```python
import requests

def uk_companies_house_search(company_name, api_key):
    """Search UK Companies House"""
    response = requests.get(
        f"https://api.company-information.service.gov.uk/search/companies",
        params={'q': company_name, 'items_per_page': 20},
        auth=(api_key, '')
    )
    return response.json() if response.status_code == 200 else {}

def uk_get_company_officers(company_number, api_key):
    """Get all officers for a UK company"""
    response = requests.get(
        f"https://api.company-information.service.gov.uk/company/{company_number}/officers",
        auth=(api_key, '')
    )
    return response.json() if response.status_code == 200 else {}

def uk_filing_history(company_number, api_key):
    """Get all filings for a UK company"""
    response = requests.get(
        f"https://api.company-information.service.gov.uk/company/{company_number}/filing-history",
        auth=(api_key, '')
    )
    return response.json() if response.status_code == 200 else {}
```

**Panama Papers and Pandora Papers databases**: The ICIJ (International Consortium of Investigative Journalists) Offshore Leaks Database contains data from multiple major leak investigations. Searchable at offshoreleaks.icij.org. Covers offshore entities, addresses, and their officers.

---

## 7.3 Court Records

Litigation is one of the most informative windows into an individual's or organization's history. Court records reveal disputes, creditors, criminal history, relationships, and financial status that appear nowhere else.

### U.S. Federal Courts: PACER

PACER (Public Access to Court Electronic Records) provides access to all U.S. federal court documents:

- Civil lawsuits
- Criminal cases (public portions)
- Bankruptcy filings
- Appellate court decisions

Access requires a PACER account and charges 10 cents per page (capped at $3 per document). The cost is modest but adds up for bulk research.

**Alternative free access**: Many federal court opinions are published free on CourtListener (courtlistener.com), which also has RECAP — a browser extension that uploads PACER documents to a free public archive.

### U.S. State Courts

State courts handle most civil litigation, criminal cases (misdemeanors and felonies), family court, and probate matters. Access varies enormously:

- Some states (New York's NYSCEF, Texas, California) have comprehensive online systems
- Many states require county-by-county access
- Some states charge for access; others are free
- Small claims court records are often not online at all

**Practical approach**: Search your target state's court system first, then county-level courts in the subject's known counties of residence and business activity.

### Bankruptcy Records

Bankruptcy filings are among the richest public records in existence. A Chapter 7 or Chapter 11 petition includes:

- Schedule A/B: All assets owned (real property, vehicles, bank accounts, investments, intellectual property)
- Schedule D: Secured creditors (mortgage holders, secured lenders)
- Schedule E/F: Unsecured creditors (credit card companies, trade creditors, lawsuit judgments)
- Statement of Financial Affairs: Income sources, prior lawsuits, prior bankruptcies, transfers of property in the prior two years
- Statement of Current Monthly Income: Employment and income details

This is a comprehensive financial disclosure that reveals assets, liabilities, income, and financial relationships that are otherwise invisible.

### Criminal Records

Criminal history research is subject to significant legal constraints:

- Federal criminal records are accessible via PACER for federal charges
- State criminal records vary in accessibility — many states have online searchable databases, others require formal request
- Arrest records (without conviction) may be restricted under state law
- Expunged records are sealed by court order and should not appear in legitimate searches
- Some states restrict use of criminal records for employment or housing decisions even when publicly accessible

**Commercial criminal background check services** aggregate criminal records across jurisdictions. These require permissible purpose documentation and often FCRA compliance.

---

## 7.4 Property and Real Estate Records

Property records are among the most consistently public and useful records in the OSINT toolkit.

### What Property Records Reveal

- **Ownership**: Current and historical owners of real property
- **Purchase history**: Sale dates and prices, which reveal when a person acquired a property
- **Mortgage information**: Whether a property is encumbered, by whom, and for how much
- **Tax assessment**: Assessed value and tax payment history
- **Legal descriptions**: Lot dimensions, legal descriptions
- **Transfer history**: Complete chain of title going back decades or centuries

### Accessing Property Records

In the U.S., property records are maintained by county-level recorder, assessor, or clerk offices. Most major counties have online search systems.

**Direct access**:
- County assessor/recorder websites — free, primary source
- Some counties charge for document copies

**Aggregated access**:
- **Zillow/Redfin**: Free access to transaction history and ownership information
- **ATTOM Data Solutions**: Commercial property data API with comprehensive coverage
- **PropertyRadar**: Property ownership and foreclosure data
- **LPS/Black Knight**: Comprehensive mortgage and property data (enterprise pricing)

```python
import requests

def attom_property_search(address, api_key):
    """Search ATTOM property database"""
    response = requests.get(
        "https://api.attomdata.com/propertyapi/v1.0.0/property/detail",
        headers={
            'accept': 'application/json',
            'apikey': api_key
        },
        params={'address1': address}
    )
    return response.json() if response.status_code == 200 else {}

def build_address_history(person_name, known_addresses):
    """Build address history from property records"""
    history = []
    for address in known_addresses:
        # Query property records
        prop_data = attom_property_search(address, 'YOUR_KEY')
        if prop_data:
            history.append({
                'address': address,
                'data': prop_data
            })
    return history
```

---

## 7.5 Financial and Regulatory Records

### Campaign Finance Records

The Federal Election Commission (FEC) publishes complete data on political contributions:

- All contributions to federal campaigns over $200
- Contributor name, employer, occupation, address, date, amount
- Searchable at FEC.gov and via FEC API

State-level campaign finance records are maintained by state election boards with varying accessibility.

**Investigative value**: Political contributions reveal associations, priorities, and relationships. Unusual patterns (individuals donating to both opposing candidates, large contributions from shell companies) may indicate coordination or foreign influence concerns.

```python
import requests

def fec_contribution_search(name=None, employer=None, zip_code=None, min_amount=0):
    """Search FEC individual contributions database"""
    params = {
        'sort_hide_null': 'false',
        'per_page': 100,
        'sort': '-contribution_receipt_date',
        'api_key': 'DEMO_KEY'  # Replace with actual key
    }

    if name:
        params['contributor_name'] = name
    if employer:
        params['contributor_employer'] = employer
    if zip_code:
        params['contributor_zip'] = zip_code
    if min_amount:
        params['min_amount'] = min_amount

    response = requests.get(
        'https://api.open.fec.gov/v1/schedules/schedule_a/',
        params=params
    )

    if response.status_code == 200:
        data = response.json()
        return [{
            'contributor': r.get('contributor_name'),
            'employer': r.get('contributor_employer'),
            'occupation': r.get('contributor_occupation'),
            'amount': r.get('contribution_receipt_amount'),
            'date': r.get('contribution_receipt_date'),
            'recipient': r.get('committee', {}).get('name'),
            'city': r.get('contributor_city'),
            'state': r.get('contributor_state'),
        } for r in data.get('results', [])]
    return []
```

### FINRA BrokerCheck

The Financial Industry Regulatory Authority (FINRA) BrokerCheck provides public access to registration and disciplinary history for broker-dealers, investment advisers, and their representatives.

BrokerCheck reveals:
- Employment history at registered firms
- Regulatory actions and sanctions
- Customer complaints and arbitration awards
- Criminal disclosures and civil court actions
- Examination failures and regulatory violations

For anyone operating in financial services, BrokerCheck is an essential research tool.

### Federal Contracts and Grants

**USASpending.gov**: All federal contracts, grants, loans, and other financial assistance. Searchable by recipient name, NAICS code, program, and location. Reveals which organizations do business with the federal government and on what terms.

**SAM.gov**: The System for Award Management maintains registration requirements for federal contractors. Includes debarment and exclusion lists — organizations and individuals barred from federal contracting.

---

## 7.6 Professional Licensing and Regulatory Records

### Medical Licensing

State medical boards maintain public databases of licensed physicians. These databases typically include:

- License number and status (active, suspended, revoked)
- License issue and expiration dates
- Medical school and graduation year
- Specialties and board certifications
- Disciplinary actions and sanctions
- Malpractice judgments (in some states)

The Federation of State Medical Boards provides a national practitioner data bank query service. The National Practitioner Data Bank (NPDB) is available only to authorized healthcare entities, but state board databases are publicly accessible.

### Legal Licensing

State bar associations maintain attorney licensing databases with:

- License status and bar number
- Law school and graduation year
- Disciplinary history
- Admission dates in each state
- Areas of practice

The ABA and state bars publish this information publicly because attorneys are officers of the court with professional accountability obligations.

### Financial Services Licensing

**FINRA BrokerCheck** (discussed above)
**IAPD (Investment Adviser Public Disclosure)**: SEC's database for registered investment advisers
**NMLS Consumer Access**: Nationwide Multistate Licensing System for mortgage and financial services professionals

---

## 7.7 Aggregation Strategy: Building Comprehensive Profiles

The true power of public records lies not in any single record type but in systematic aggregation across multiple sources. A well-constructed aggregation workflow can build remarkably complete pictures of individuals and organizations.

### Individual Research Aggregation Workflow

```python
import json
from dataclasses import dataclass, field
from typing import List, Dict, Optional

@dataclass
class PersonProfile:
    """Structured profile built from public records aggregation"""
    full_name: str
    name_variants: List[str] = field(default_factory=list)

    # Identity
    dob: Optional[str] = None
    ssn_last4: Optional[str] = None  # If legally accessible

    # Address history
    addresses: List[Dict] = field(default_factory=list)

    # Professional history
    employment: List[Dict] = field(default_factory=list)
    licenses: List[Dict] = field(default_factory=list)
    professional_disciplinary: List[Dict] = field(default_factory=list)

    # Corporate affiliations
    business_affiliations: List[Dict] = field(default_factory=list)

    # Legal history
    court_records: List[Dict] = field(default_factory=list)
    bankruptcies: List[Dict] = field(default_factory=list)

    # Financial
    properties: List[Dict] = field(default_factory=list)
    campaign_contributions: List[Dict] = field(default_factory=list)
    regulatory_records: List[Dict] = field(default_factory=list)

    # Online presence
    social_media: List[Dict] = field(default_factory=list)

    # Source documentation
    sources: List[Dict] = field(default_factory=list)

    def add_source(self, source_type, url, date, notes=""):
        self.sources.append({
            'type': source_type,
            'url': url,
            'date': date,
            'notes': notes
        })

    def confidence_assessment(self):
        """Assess overall confidence in profile accuracy"""
        confirmed_items = sum([
            len(self.addresses),
            len(self.employment),
            len(self.business_affiliations),
        ])
        multi_source_items = sum([
            1 for addr in self.addresses if addr.get('sources_count', 1) > 1
        ])

        return {
            'total_data_points': confirmed_items,
            'multi_source_confirmed': multi_source_items,
            'confidence_level': 'HIGH' if multi_source_items > 3 else 'MEDIUM' if multi_source_items > 0 else 'LOW'
        }

class PublicRecordsAggregator:
    """Orchestrates public records collection across multiple sources"""

    def __init__(self, fec_key=None, attom_key=None):
        self.fec_key = fec_key
        self.attom_key = attom_key
        self.profile = None

    def build_individual_profile(self, name: str, state: str = None) -> PersonProfile:
        """Build a comprehensive profile from public records"""
        profile = PersonProfile(full_name=name)

        print(f"[*] Building profile for: {name}")

        # Step 1: Business affiliations (Secretary of State)
        print("[*] Searching business registries...")
        corps = search_opencorporates(name, jurisdiction=state)
        for corp in corps[:10]:  # Limit results
            profile.business_affiliations.append(corp)
            profile.add_source('opencorporates', corp.get('opencorporates_url'), 'current')

        # Step 2: Campaign contributions (FEC)
        if self.fec_key:
            print("[*] Searching FEC campaign contributions...")
            contributions = fec_contribution_search(name=name)
            profile.campaign_contributions = contributions
            if contributions:
                profile.add_source('FEC', 'https://www.fec.gov', 'current',
                                  f"{len(contributions)} contributions found")

        # Step 3: Properties (if ATTOM key available)
        # ... property search

        # Step 4: Court records
        # ... court records search

        print(f"[+] Profile built with {len(profile.sources)} documented sources")
        return profile
```

---

## 7.8 Data Quality and Verification

Public records aggregation creates a significant quality management problem. Records from different jurisdictions, different time periods, and different source systems contain errors, inconsistencies, and gaps. Systematic verification is not optional.

### Common Data Quality Issues

**Name matching errors**: "John Smith" who owned a property in Texas is not necessarily the same "John Smith" who was disciplined by the Florida medical board. Without date of birth or other disambiguating information, name-based matching produces false positives.

**Address lag**: Property records, license databases, and court records all have different update frequencies. An address that appears in one database as current may have been superseded in another.

**Entity disambiguation**: "ABC Corp" in one state is not necessarily related to "ABC Corp" in another state. Corporate names are registered state-by-state, and name conflicts are common.

**Record completeness**: Not all states have the same digitization depth. A complete records search requires knowing what is not digitized in a given jurisdiction and potentially requesting physical records.

**Intentional obfuscation**: Sophisticated subjects may use legal name changes, shell companies, trust structures, or nominee services to obscure connections that public records would otherwise reveal.

### Verification Methodology

For every significant finding from public records:

1. **Primary source confirmation**: Can the finding be confirmed directly from the primary source (the actual agency database or document), not just a commercial aggregator's copy?

2. **Corroborating source**: Does at least one other independent source confirm the same finding?

3. **Internal consistency**: Is the finding consistent with other confirmed findings? An employment date that conflicts with a known address change, for example, requires investigation.

4. **Date evaluation**: Is the record current or historical? What is the investigative relevance of the record date?

5. **Identity disambiguation**: When multiple individuals share a name, is there clear evidence that this specific record belongs to the investigation subject?

---

## Summary

Public records represent the most authoritative and legally unambiguous OSINT sources, providing a foundation of government-generated data about businesses, property, litigation, professional credentials, and financial activities.

Effective public records investigation requires systematic navigation of multiple record types and jurisdictions: corporate registries, court records, property records, professional licensing databases, financial regulators, and campaign finance records. No single source provides complete coverage — the power lies in aggregation and triangulation across sources.

Commercial aggregation platforms accelerate research but introduce data quality issues that require careful verification. Understanding the provenance, accuracy, and limitations of commercial data is essential for producing reliable investigative findings.

Building structured profiles that link evidence to sources enables quality review, supports legal defensibility, and reveals gaps that require additional collection.

---

## Common Mistakes and Pitfalls

- **False identity matches**: Attributing records belonging to a different person with the same name
- **Treating commercial aggregator data as primary records**: Aggregators copy and often corrupt data; always verify at the primary source for important findings
- **Missing jurisdictional coverage**: Assuming a national search when relevant records may only exist in specific state or county systems
- **Ignoring negative findings**: A person with no court records in the searched jurisdictions may have significant legal history in unsearched jurisdictions
- **UCC neglect**: UCC filings are routinely overlooked but reveal financial relationships not visible anywhere else
- **Timestamp blindness**: Using records without noting their date, leading to conflation of historical and current information

---

## Further Reading

- BRB Publications, *The Sourcebook to Public Record Information*
- National Center for State Courts — state court system directories and access information
- OpenCorporates API documentation
- FEC API documentation (api.open.fec.gov)
- ICIJ Offshore Leaks Database documentation
