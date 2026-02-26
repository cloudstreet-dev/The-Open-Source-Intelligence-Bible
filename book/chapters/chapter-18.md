# Chapter 18: Private Investigator Workflows

## Learning Objectives

By the end of this chapter, you will be able to:
- Apply OSINT methodology to the primary case types handled by private investigators
- Navigate the legal requirements and constraints specific to PI practice
- Build efficient research workflows for common PI investigation scenarios
- Use commercial data platforms appropriately within PI licensing constraints
- Document investigations to professional and legal standards
- Integrate AI tools into traditional PI practice

---

## 18.1 The Modern Private Investigator

The image of the private investigator as a fedora-wearing surveillance specialist stalking subjects in parking garages is outdated. Modern private investigation is heavily digital. Experienced PIs estimate that 70-80% of modern investigative work involves digital research, public records, and OSINT — with physical surveillance reserved for cases that require it and cannot be resolved through digital methods.

This shift has created both opportunity and challenge. The opportunity: digital methods can achieve in hours what once required days of physical work. The challenge: the tools and methods have changed faster than the professional training available for the field, creating a capability gap between technologically sophisticated practitioners and those who have not updated their methods.

This chapter applies OSINT methodology to the primary case types handled by licensed private investigators, within the legal and ethical constraints of the profession.

---

## 18.2 Legal Framework for PI Practice

Private investigators must operate within two overlapping legal frameworks:

**State PI licensing law**: Most U.S. states require PI licensure. Requirements vary significantly by state — some require extensive prior law enforcement experience, others only a background check and examination. Practitioners must be licensed in the state where investigative activities take place, not just the state where they are registered.

**Data access law**: PIs have access to some data sources not available to the general public (like certain commercial databases requiring professional subscription) but do not have the access rights of law enforcement. PIs cannot legally:
- Access non-public law enforcement databases (NCIC, DMV records in most states without a specific statutory authorization)
- Place wiretaps or intercept electronic communications
- Access private communications without consent of at least one party (and in all-party consent states, consent of all parties)
- Access computer systems without authorization

**FCRA compliance**: When PI findings will be used for employment, housing, or credit decisions, the report and process must comply with the FCRA.

---

## 18.3 Core PI Case Types and Workflows

### 1. Background Investigations

Background investigations are among the most common PI assignments — verifying an individual's history for employment, due diligence, or relationship purposes.

**Standard workflow**:

```
Phase 1: Identity Verification
├── Name variants and DOB confirmation
├── SSN verification (requires professional database access)
├── Address history verification
└── Biometric matching (photos from online sources)

Phase 2: Professional Background
├── Employment history verification
│   ├── LinkedIn profile analysis
│   ├── Corporate records (Secretary of State filings)
│   ├── SEC filings for executive positions
│   └── Professional licensing verification
├── Education verification
│   ├── National Student Clearinghouse (licensed PI access)
│   └── University graduation records (some are public)
└── Professional license status
    ├── State licensing boards
    └── FINRA BrokerCheck / SEC IAPD

Phase 3: Legal History
├── Federal court records (PACER)
├── State criminal records (state-specific systems)
├── Civil litigation history
├── Bankruptcy records
└── Sex offender registry check (public)

Phase 4: Financial Indicators
├── Property ownership
├── UCC filings
├── Judgment and lien searches
└── Bankruptcy history

Phase 5: Digital Presence
├── Social media profile analysis
├── Online presence mapping
└── News and media coverage
```

```python
class BackgroundInvestigationWorkflow:
    """Structured background investigation workflow"""

    def __init__(self, subject_name: str, dob: str = None, state: str = None):
        self.subject = subject_name
        self.dob = dob
        self.state = state
        self.findings = []
        self.sources = []

    def document_finding(self, category: str, finding: str, source: str,
                        confidence: str, source_url: str = None, date_accessed: str = None):
        """Document a finding with full source citation"""
        from datetime import datetime
        self.findings.append({
            'category': category,
            'finding': finding,
            'source': source,
            'confidence': confidence,
            'source_url': source_url,
            'date_accessed': date_accessed or datetime.now().strftime('%Y-%m-%d')
        })

    def check_professional_licenses(self, state: str, professions: list = None) -> list:
        """
        Check professional licensing status across common boards
        In production, this would query each state's licensing API
        """
        results = []

        # Medical license check
        # Most states have online license verification
        # Example: verify.tn.gov, elicense.ohio.gov, etc.

        # Legal license check (state bar association)
        # lawyer.com/find-a-lawyer or direct state bar search

        # Financial services check (FINRA BrokerCheck)
        import requests

        try:
            response = requests.get(
                f"https://api.brokercheck.finra.org/search/individual",
                params={
                    'query': self.subject,
                    'hl': 'true',
                    'includePrevious': 'true',
                    'exactMatch': 'false',
                    'primary': 'true',
                    'type': 'individual',
                    'start': 0,
                    'count': 5
                },
                headers={
                    'Referer': 'https://brokercheck.finra.org/'
                }
            )

            if response.status_code == 200:
                data = response.json()
                for hit in data.get('hits', {}).get('hits', []):
                    source = hit.get('_source', {})
                    results.append({
                        'type': 'FINRA_registered',
                        'name': source.get('ind_firstname', '') + ' ' + source.get('ind_lastname', ''),
                        'CRD': source.get('ind_source_id', ''),
                        'employed_by': [e.get('empl_nm') for e in source.get('ind_pc_employers', [])],
                        'has_disclosures': source.get('ind_bc_scope', '') != '0',
                    })
        except Exception as e:
            pass

        return results

    def search_court_records(self) -> dict:
        """Search federal and state court records"""
        court_results = {
            'federal': [],
            'state': [],
            'bankruptcy': []
        }

        # PACER federal courts
        # In production, requires PACER account and API access
        # pacer.uscourts.gov

        # For demonstration, structure the query approach
        federal_query = f"{self.subject}"
        if self.state:
            state_query = f"{self.subject} {self.state}"

        return court_results

    def compile_report(self) -> str:
        """Generate background investigation report"""
        report_lines = [
            f"# Background Investigation Report",
            f"**Subject**: {self.subject}",
            f"**DOB**: {self.dob or 'Not provided'}",
            f"**Investigation Date**: {__import__('datetime').datetime.now().strftime('%Y-%m-%d')}",
            "",
            "## Summary of Findings",
            ""
        ]

        # Group findings by category
        categories = {}
        for finding in self.findings:
            cat = finding['category']
            if cat not in categories:
                categories[cat] = []
            categories[cat].append(finding)

        for category, cat_findings in categories.items():
            report_lines.append(f"### {category.replace('_', ' ').title()}")
            for f in cat_findings:
                report_lines.append(f"- [{f['confidence'].upper()}] {f['finding']}")
                report_lines.append(f"  *Source: {f['source']} (accessed {f['date_accessed']})*")
            report_lines.append("")

        return '\n'.join(report_lines)
```

### 2. Asset Investigation

Asset investigations locate assets for judgment collection, divorce proceedings, or fraud recovery.

**Asset investigation workflow**:

**Real Property**: County assessor/recorder databases in all counties where subject has lived or done business.

**Corporate Assets**: Secretary of State filings for business entities owned by subject; UCC filings for pledged assets.

**Financial Indicators**: Bankruptcy filings (which require disclosure of all assets), property tax records, vehicle registration (restricted in most states without PI authorization).

**Digital/Cryptocurrency**: Social media for lifestyle indicators; public blockchain address analysis if cryptocurrency addresses are known.

```python
def conduct_asset_investigation(subject_name: str, known_states: list) -> dict:
    """
    Structure an asset investigation across multiple jurisdictions
    """
    asset_inventory = {
        'real_property': [],
        'business_interests': [],
        'vehicles': [],  # DMV access restricted — typically PI/attorney access only
        'financial_indicators': [],
        'bankruptcy_disclosures': []
    }

    for state in known_states:
        # 1. Property records search
        # Each county's recorder/assessor system
        print(f"Searching property records in {state}...")

        # 2. Business entity search
        corp_results = search_opencorporates(subject_name, jurisdiction=state.lower()[:2])
        for corp in corp_results:
            asset_inventory['business_interests'].append({
                'type': 'business_entity',
                'name': corp.get('name'),
                'jurisdiction': corp.get('jurisdiction'),
                'status': corp.get('status'),
                'source': 'OpenCorporates'
            })

    # 3. Bankruptcy search (federal — all jurisdictions in one place)
    # PACER bankruptcy court search
    print("Searching PACER for bankruptcy filings...")

    # 4. UCC search
    # Each state Secretary of State UCC database
    for state in known_states:
        print(f"Searching UCC filings in {state}...")

    return asset_inventory
```

### 3. Infidelity and Domestic Investigations

Domestic investigations are among the most ethically sensitive PI cases. They intersect with potential stalking behavior, emotional manipulation, and safety risks for all parties.

**Professional guidelines**:

- Work only for the spouse with legal standing; verify client identity and marital status
- Investigate in jurisdictions with appropriate PI licensing
- All-party consent recording laws apply to intercepted communications — do not instruct clients to record without legal advice
- Do not provide findings directly to children or family members who are not the client
- Document authorization chain carefully for potential litigation use

**OSINT components of domestic investigations**:

- Social media activity timeline analysis
- Digital footprint mapping for undisclosed accounts
- Geographic pattern analysis from geotagged posts
- Vehicle tracking via publicly available methods (not GPS tracking devices without authorization)
- Property records for undisclosed assets

### 4. Workers' Compensation and Insurance Fraud

Insurance fraud investigations use OSINT to identify behavioral inconsistencies with claimed disabilities or injuries.

**OSINT workflow**:

1. **Baseline review**: Social media presence and activity before the claimed incident
2. **Post-incident monitoring**: Public social media activity and posting patterns
3. **Physical activity indicators**: Tagged photos, geotagged activity, fitness apps, event attendance
4. **Second job detection**: LinkedIn, other job platforms, business registrations, reviews
5. **Behavioral timeline**: Activity patterns that contradict claimed limitations

```python
def social_media_activity_audit(subject_accounts: dict, incident_date: str, claim_details: str) -> dict:
    """
    Audit social media activity for consistency with workers' comp claim
    incident_date: ISO date string
    claim_details: description of claimed injury/limitation
    """
    audit_results = {
        'pre_incident': [],
        'post_incident_before_claim': [],
        'post_claim': [],
        'inconsistencies': []
    }

    # For each platform
    for platform, username in subject_accounts.items():
        print(f"Analyzing {platform} account: {username}")
        # Collect public posts, tags, check-ins
        # Compare activity level and type before/after incident date
        # Look for physical activity indicators post-injury

    return audit_results
```

### 5. Missing Persons Investigations

Missing persons investigations combine urgency with particular care not to endanger the subject if they have fled a dangerous situation.

**Critical ethical consideration**: A missing person may be missing because they are escaping abuse, domestic violence, or stalking. Providing location information to the person seeking them — without knowing their relationship to the missing person and the circumstances of disappearance — could endanger the missing person. Requests for missing persons investigations require careful authorization vetting.

**OSINT approach for legitimate cases**:

1. **Last known digital trace**: Most recent social media activity, device location indicators, last communications
2. **Association network**: Who did they know? Who might they be with?
3. **Financial activity**: Bank/card transaction trail (requires law enforcement access for private accounts; public records for business accounts)
4. **Transportation records**: Flight manifests (restricted), vehicle records
5. **Geographic area**: Is there a geographic direction suggested by contacts or activity?

---

## 18.4 AI Integration for PI Practice

AI tools provide significant leverage in PI work:

```python
import anthropic
import json

def ai_assisted_timeline_analysis(social_media_posts: list, claim_date: str, claim_description: str) -> str:
    """
    Use AI to identify inconsistencies between social media activity and insurance claim
    """
    client = anthropic.Anthropic()

    posts_text = "\n".join([
        f"[{p.get('date', 'Unknown')} - {p.get('platform', 'Unknown')}]: {p.get('content', '')}"
        for p in sorted(social_media_posts, key=lambda x: x.get('date', ''))
    ])

    prompt = f"""You are a licensed private investigator analyzing social media activity for an insurance fraud investigation.

CLAIM DETAILS:
Claim Date: {claim_date}
Claim Description: {claim_description}

SOCIAL MEDIA ACTIVITY (PUBLIC POSTS):
{posts_text}

Analyze the social media activity for:

1. PHYSICAL ACTIVITY INDICATORS
- Any posts suggesting physical activity inconsistent with the claimed limitation
- Location check-ins or geotagged photos
- Photos showing subject in physical activities
- References to sports, exercise, or demanding physical activities

2. EMPLOYMENT INDICATORS
- References to work activities during claimed disability period
- Business-related posts (meeting clients, work activities)
- Job-seeking activity

3. TIMELINE ANOMALIES
- Activity level changes around claim date
- Social media behavior patterns that warrant investigation

4. DOCUMENTATION NOTES
- Specific posts that should be archived as evidence
- Any content that appears to have been deleted or modified

Present findings objectively. Note dates, platform, and exact content for all relevant posts. Distinguish between what posts directly show vs. what they suggest. Do not overinterpret.

This analysis is for a licensed investigator conducting a lawful insurance fraud investigation."""

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{"role": "user", "content": prompt}]
    )

    return response.content[0].text
```

---

## 18.5 Documentation Standards for PI Work

PI work that will be used in legal proceedings requires forensic-grade documentation:

**Chain of custody**: Every piece of digital evidence must have a documented chain of custody from collection through presentation. Who collected it, when, from what source, using what method, and who has had access since.

**Contemporaneous documentation**: Create documentation at the time of collection, not after. Notes written from memory after the fact have diminished legal value.

**Screenshot best practices**:
- Include URL, timestamp, and browser metadata in screenshots
- Archive pages to the Wayback Machine immediately after collection
- Use tools like Hunchly that create automatic, timestamped collections
- Save original files (HTML, images) alongside screenshots

**Report standards**:
- Every factual claim must be cited to a specific documented source
- Confidence levels must be explicit
- Limitations on methodology must be disclosed
- Expert credentials must be established if report may be presented in court

---

## Summary

Modern private investigation is primarily a digital discipline. Physical surveillance has been largely supplemented by social media monitoring, public records research, commercial database access, and geospatial analysis.

Licensed PIs must navigate their state licensing requirements, data access restrictions that differ from law enforcement powers, FCRA obligations for certain use cases, and ethical constraints that prevent harm to subjects and third parties.

AI tools — particularly LLMs for document analysis and timeline pattern recognition — provide significant efficiency gains for the document-heavy aspects of PI work, from background investigations to insurance fraud analysis.

Documentation discipline, always important in PI work, is elevated by the potential for findings to be used in legal proceedings. Forensic-grade documentation from the moment of collection is the professional standard.

---

## Common Mistakes and Pitfalls

- **Unlicensed practice**: Conducting PI activities in states where you are not licensed
- **DMV overreach**: Attempting to access motor vehicle records without the specific statutory authorization required in most states
- **Domestic investigation boundary crossing**: Following subjects or accessing communications without appropriate authorization
- **FCRA neglect**: Providing background investigation reports that are used for employment/housing without FCRA compliance
- **Authorization documentation failures**: Not documenting client authorization chain before beginning investigation
- **Evidence contamination**: Interacting with evidence (liking posts, following accounts) that alters the subject's behavior or creates evidentiary complications

---

## Further Reading

- National Association of Legal Investigators (NALI) professional standards
- ACFE (Association of Certified Fraud Examiners) fraud investigation guidelines
- Each state's Department of Public Safety / PI licensing board website
- Michael Bazzell's *OSINT Techniques* — practitioner-focused personal investigation methodology
- PI Magazine — professional PI trade publication
