# Chapter 21: Financial Crime and AML Investigations

## Learning Objectives

By the end of this chapter, you will be able to:
- Apply OSINT methodology to anti-money laundering (AML) and financial crime investigations
- Navigate the public records and data sources specific to financial intelligence
- Trace beneficial ownership through corporate structures using open sources
- Analyze cryptocurrency transactions using public blockchain data
- Conduct sanctions screening and politically exposed person (PEP) research
- Understand the regulatory context that mandates OSINT in financial compliance

---

## 21.1 OSINT in Financial Crime Investigation

Financial crime investigation — encompassing money laundering, fraud, sanctions evasion, and corruption — is one of the most data-intensive OSINT applications. Financial criminals actively construct complexity: shell companies, nominee directors, layered transactions, and opaque ownership structures are the tools of the trade.

OSINT penetrates this complexity by combining:
- Corporate registry data to map entity structures
- Property records to trace asset flows
- Beneficial ownership registers (where available)
- Cryptocurrency blockchain analysis
- Sanctions and PEP lists
- Court records, regulatory actions, and news coverage
- AIS and ADS-B data for sanctions evasion tracking

Regulatory mandates drive institutional OSINT demand. AML regulations — the Bank Secrecy Act in the U.S., the EU's Anti-Money Laundering Directives, FATF recommendations globally — require financial institutions to understand their customers and their customers' customers. This creates massive demand for scalable OSINT-based due diligence.

---

## 21.2 Regulatory Framework

### Key AML/Financial Crime Regulations

**Bank Secrecy Act (BSA) / FinCEN regulations**: U.S. financial institutions must file Suspicious Activity Reports (SARs), Currency Transaction Reports (CTRs), and maintain Customer Due Diligence (CDD) records including beneficial ownership information.

**EU Anti-Money Laundering Directives (AMLD)**: Progressive strengthening of AML requirements across EU member states, including beneficial ownership registers.

**FATF Recommendations**: The Financial Action Task Force provides the global standard for AML/CFT (counter-financing of terrorism) frameworks.

**OFAC Sanctions**: The Office of Foreign Assets Control administers U.S. economic sanctions programs. Sanctions screening is a compliance requirement for most financial institutions and many corporates.

### Beneficial Ownership Requirements

The Corporate Transparency Act (U.S., effective 2024) requires most U.S. corporations to report beneficial owners — individuals who own 25%+ or who exercise substantial control — to FinCEN. This creates a central registry of ultimate beneficial owners.

European AML directives have progressively required public beneficial ownership registers. EU5AMLD made these public; some member states have implemented fully searchable databases.

---

## 21.3 Corporate Beneficial Ownership Analysis

### Mapping Corporate Structures

```python
import requests
import json
from typing import Dict, List, Optional, Set
from dataclasses import dataclass, field
import networkx as nx

@dataclass
class CorporateEntity:
    """Corporate entity for ownership analysis"""
    name: str
    entity_type: str  # 'corporation', 'LLC', 'trust', 'individual', 'foundation'
    jurisdiction: str
    registration_number: Optional[str] = None
    registered_address: Optional[str] = None
    incorporation_date: Optional[str] = None
    status: str = 'unknown'
    officers: List[Dict] = field(default_factory=list)
    registered_agent: Optional[str] = None
    risk_flags: List[str] = field(default_factory=list)
    source: str = ""

class BeneficialOwnershipMapper:
    """Map beneficial ownership through corporate structures"""

    def __init__(self):
        self.entities = {}
        self.ownership_graph = nx.DiGraph()
        self.visited = set()

        # High-risk jurisdictions for shell companies
        self.high_risk_jurisdictions = {
            'British Virgin Islands', 'BVI', 'Cayman Islands', 'Panama',
            'Seychelles', 'Marshall Islands', 'Belize', 'Anguilla',
            'Cyprus', 'Malta', 'Vanuatu', 'Isle of Man', 'Jersey', 'Guernsey',
            'Liechtenstein', 'San Marino', 'Andorra'
        }

    def add_entity(self, entity: CorporateEntity, owned_by: str = None,
                   ownership_percentage: float = None):
        """Add an entity and optionally its owner"""
        entity_key = f"{entity.name}_{entity.jurisdiction}"
        self.entities[entity_key] = entity
        self.ownership_graph.add_node(entity_key, **{
            'name': entity.name,
            'type': entity.entity_type,
            'jurisdiction': entity.jurisdiction,
            'risk_flags': entity.risk_flags
        })

        if owned_by and owned_by in self.ownership_graph:
            self.ownership_graph.add_edge(
                owned_by, entity_key,
                weight=ownership_percentage or 0,
                relationship='owns'
            )

        # Flag high-risk jurisdictions
        if entity.jurisdiction in self.high_risk_jurisdictions:
            entity.risk_flags.append(f'HIGH_RISK_JURISDICTION: {entity.jurisdiction}')

        return entity_key

    def trace_ultimate_owners(self, entity_key: str, max_depth: int = 10) -> dict:
        """Trace the ownership chain to find ultimate beneficial owners"""
        result = {
            'starting_entity': entity_key,
            'ownership_chain': [],
            'ultimate_beneficial_owners': [],
            'shell_companies_detected': 0,
            'high_risk_jurisdictions': [],
            'total_depth': 0,
        }

        def traverse(current_entity, depth, path):
            if depth > max_depth or current_entity in self.visited:
                return
            self.visited.add(current_entity)

            entity_data = self.entities.get(current_entity, {})
            current_path = path + [current_entity]

            # Find who owns this entity
            predecessors = list(self.ownership_graph.predecessors(current_entity))

            if not predecessors:
                # No further owners — this may be an ultimate owner or a gap
                entity = self.entities.get(current_entity)
                if entity and entity.entity_type == 'individual':
                    result['ultimate_beneficial_owners'].append({
                        'entity': current_entity,
                        'chain_depth': depth,
                        'path': current_path,
                    })
                result['total_depth'] = max(result['total_depth'], depth)
                return

            for owner in predecessors:
                owner_entity = self.entities.get(owner)
                if owner_entity:
                    if owner_entity.jurisdiction in self.high_risk_jurisdictions:
                        result['high_risk_jurisdictions'].append(owner_entity.jurisdiction)

                    # Check for shell company characteristics
                    if self._is_shell_indicator(owner_entity):
                        result['shell_companies_detected'] += 1

                traverse(owner, depth + 1, current_path)

        traverse(entity_key, 0, [])
        self.visited.clear()
        return result

    def _is_shell_indicator(self, entity: CorporateEntity) -> bool:
        """Assess whether an entity has shell company characteristics"""
        indicators = [
            entity.jurisdiction in self.high_risk_jurisdictions,
            entity.entity_type in ('trust', 'foundation'),
            len(entity.officers) == 0,
            entity.registered_agent is not None and len(entity.officers) == 0,
            'nominee' in entity.name.lower(),
            entity.name.endswith('Holdings') or entity.name.endswith('Investments'),
        ]
        return sum(indicators) >= 2  # Two or more indicators suggest shell

    def search_opencorporates_officers(self, person_name: str) -> list:
        """Find all corporate positions held by a person across jurisdictions"""
        response = requests.get(
            'https://api.opencorporates.com/v0.4/officers/search',
            params={'q': person_name, 'format': 'json'}
        )

        positions = []
        if response.status_code == 200:
            data = response.json()
            officers = data.get('results', {}).get('officers', [])
            for officer_data in officers:
                officer = officer_data.get('officer', {})
                positions.append({
                    'person': officer.get('name', ''),
                    'position': officer.get('position', ''),
                    'company': officer.get('company', {}).get('name', ''),
                    'jurisdiction': officer.get('company', {}).get('jurisdiction_code', ''),
                    'start_date': officer.get('start_date', ''),
                    'end_date': officer.get('end_date', ''),
                    'is_current': not bool(officer.get('end_date')),
                })

        return positions

    def analyze_officer_network(self, person_name: str) -> dict:
        """Analyze the corporate network of a person through their officer positions"""
        positions = self.search_opencorporates_officers(person_name)

        companies = {}
        jurisdictions = {}
        co_officers = {}

        for position in positions:
            company = position['company']
            jurisdiction = position['jurisdiction']

            companies[company] = companies.get(company, 0) + 1
            jurisdictions[jurisdiction] = jurisdictions.get(jurisdiction, 0) + 1

        return {
            'person': person_name,
            'total_positions': len(positions),
            'current_positions': sum(1 for p in positions if p['is_current']),
            'companies': list(companies.keys())[:20],
            'jurisdictions': dict(sorted(jurisdictions.items(), key=lambda x: x[1], reverse=True)),
            'positions': positions[:20]
        }
```

---

## 21.4 Sanctions and PEP Screening

Sanctions compliance requires screening against multiple lists maintained by different regulatory bodies.

### Key Sanctions Lists

**U.S. OFAC SDN List**: Specially Designated Nationals and Blocked Persons List. The primary U.S. sanctions list.

**EU Sanctions**: European Union consolidated list of sanctioned persons and entities.

**UN Security Council Sanctions**: Global sanctions imposed by the UN Security Council.

**UK OFSI**: HM Treasury Office of Financial Sanctions Implementation.

**National sanctions lists**: Many countries maintain their own sanctions lists.

```python
import requests
from datetime import datetime

class SanctionsScreener:
    """Screen entities against major sanctions databases"""

    def __init__(self):
        self.ofac_data = None
        self.eu_data = None

    def load_ofac_sdn_list(self):
        """Load the OFAC SDN list (XML or JSON format)"""
        # OFAC provides the SDN list via their website
        # The XML version is downloadable
        ofac_url = "https://www.treasury.gov/ofac/downloads/sdn.xml"

        # In practice, use a commercial sanctions screening API
        # or download and parse the XML locally

        print("Loading OFAC SDN list...")
        print("Note: For production, use commercial APIs (Dow Jones, LexisNexis, Refinitiv)")
        print("SDN list available at: https://home.treasury.gov/policy-issues/financial-sanctions/sdn-list")
        return []

    def screen_entity(self, entity_name: str, entity_type: str = 'individual') -> dict:
        """
        Screen an entity name against sanctions lists
        In production, this would use commercial APIs with fuzzy matching
        """

        results = {
            'entity': entity_name,
            'entity_type': entity_type,
            'screened_at': datetime.now().isoformat(),
            'matches': [],
            'screening_lists': []
        }

        # Commercial sanctions screening APIs to use in production:
        # - Dow Jones Factiva/Risk & Compliance
        # - LexisNexis WorldCompliance
        # - Refinitiv World-Check
        # - Accuity Bankers Almanac
        # - ComplyAdvantage

        # Free/Open alternatives (lower quality):
        # OFAC XML download and local parsing
        # EU API: https://webgate.ec.europa.eu/fsd/fsf/public/files/xmlFullSanctionsList_1_1/content

        return results

    def get_ofac_sanctions_by_program(self, program: str) -> list:
        """Get all OFAC sanctioned entities in a specific sanctions program"""
        # OFAC organizes sanctions by program (e.g., IRAN, RUSSIA, NORTH_KOREA)
        # API: https://ofac-api.treasury.gov/documentation
        api_url = "https://ofac-api.treasury.gov/sanctions/v1/sdn"

        params = {
            'program': program,
            'type': 'Entity',
            'api_key': 'YOUR_OFAC_API_KEY'
        }

        try:
            response = requests.get(api_url, params=params, timeout=30)
            if response.status_code == 200:
                return response.json().get('sdnList', {}).get('sdnEntry', [])
        except Exception as e:
            print(f"OFAC API error: {e}")

        return []

class PEPScreener:
    """Screen individuals against Politically Exposed Person databases"""

    # OpenSanctions provides free PEP and sanctions data
    OPENSANCTIONS_API = "https://api.opensanctions.org/v3"

    def search_opensanctions(self, name: str, schema: str = 'Person') -> dict:
        """
        Search OpenSanctions database (free tier available)
        OpenSanctions aggregates PEP, sanctions, and crime data
        """
        try:
            response = requests.get(
                f"{self.OPENSANCTIONS_API}/search",
                params={
                    'q': name,
                    'schema': schema,
                    'limit': 20,
                },
                headers={'Authorization': 'ApiKey YOUR_OPENSANCTIONS_KEY'}
            )

            if response.status_code == 200:
                data = response.json()
                return {
                    'query': name,
                    'total': data.get('total', {}).get('value', 0),
                    'results': [
                        {
                            'id': entity.get('id'),
                            'caption': entity.get('caption'),
                            'schema': entity.get('schema'),
                            'datasets': entity.get('datasets', []),
                            'properties': entity.get('properties', {}),
                            'score': entity.get('score', 0),
                        }
                        for entity in data.get('results', [])
                    ]
                }
        except Exception as e:
            return {'error': str(e)}

        return {}

    def check_wikidata_for_pep(self, person_name: str) -> dict:
        """
        Use Wikidata SPARQL to check if a person holds political office
        Wikidata is a free, public knowledge graph
        """
        # Wikidata SPARQL query to find political positions
        query = f"""
        SELECT ?item ?itemLabel ?position ?positionLabel ?start ?end WHERE {{
          ?item rdfs:label "{person_name}"@en .
          ?item p:P39 ?positionStatement .
          ?positionStatement ps:P39 ?position .
          OPTIONAL {{ ?positionStatement pq:P580 ?start }}
          OPTIONAL {{ ?positionStatement pq:P582 ?end }}
          SERVICE wikibase:label {{ bd:serviceParam wikibase:language "en" }}
        }}
        """

        try:
            response = requests.get(
                'https://query.wikidata.org/sparql',
                params={'query': query, 'format': 'json'},
                headers={'User-Agent': 'OSINT Research Bot/1.0'},
                timeout=30
            )

            if response.status_code == 200:
                data = response.json()
                results = data.get('results', {}).get('bindings', [])

                positions = []
                for result in results:
                    positions.append({
                        'position': result.get('positionLabel', {}).get('value', ''),
                        'start': result.get('start', {}).get('value', ''),
                        'end': result.get('end', {}).get('value', ''),
                        'is_current': not bool(result.get('end', {}).get('value'))
                    })

                return {
                    'person': person_name,
                    'is_pep': len(positions) > 0,
                    'positions': positions
                }
        except Exception as e:
            return {'error': str(e)}

        return {}
```

---

## 21.5 Cryptocurrency Blockchain Analysis

Cryptocurrency transactions are public by design on most blockchain networks, providing a permanent, auditable record of fund flows.

```python
class BlockchainAnalyzer:
    """Analyze cryptocurrency transactions using public blockchain data"""

    def get_bitcoin_address_info(self, address: str) -> dict:
        """Get information about a Bitcoin address"""
        # Using blockchain.info API (free)
        try:
            response = requests.get(
                f"https://blockchain.info/rawaddr/{address}",
                timeout=30
            )

            if response.status_code == 200:
                data = response.json()
                return {
                    'address': address,
                    'total_received': data.get('total_received', 0) / 1e8,
                    'total_sent': data.get('total_sent', 0) / 1e8,
                    'final_balance': data.get('final_balance', 0) / 1e8,
                    'transaction_count': data.get('n_tx', 0),
                    'first_tx': data.get('txs', [{}])[-1].get('time', 0) if data.get('txs') else None,
                    'last_tx': data.get('txs', [{}])[0].get('time', 0) if data.get('txs') else None,
                }
        except Exception as e:
            return {'error': str(e)}
        return {}

    def get_ethereum_address_info(self, address: str, etherscan_api_key: str) -> dict:
        """Get information about an Ethereum address"""
        try:
            balance_response = requests.get(
                'https://api.etherscan.io/api',
                params={
                    'module': 'account',
                    'action': 'balance',
                    'address': address,
                    'tag': 'latest',
                    'apikey': etherscan_api_key
                }
            )

            tx_response = requests.get(
                'https://api.etherscan.io/api',
                params={
                    'module': 'account',
                    'action': 'txlist',
                    'address': address,
                    'sort': 'desc',
                    'apikey': etherscan_api_key
                }
            )

            balance_data = balance_response.json() if balance_response.status_code == 200 else {}
            tx_data = tx_response.json() if tx_response.status_code == 200 else {}

            return {
                'address': address,
                'balance_eth': int(balance_data.get('result', 0)) / 1e18,
                'transactions': len(tx_data.get('result', [])),
                'recent_transactions': [
                    {
                        'hash': tx.get('hash'),
                        'from': tx.get('from'),
                        'to': tx.get('to'),
                        'value_eth': int(tx.get('value', 0)) / 1e18,
                        'timestamp': tx.get('timeStamp'),
                    }
                    for tx in tx_data.get('result', [])[:10]
                ]
            }
        except Exception as e:
            return {'error': str(e)}

    def analyze_transaction_clusters(self, transactions: list) -> dict:
        """
        Basic clustering analysis for cryptocurrency transactions
        In production, use Chainalysis, Elliptic, or TRM Labs
        """
        # Count counterparties
        from collections import Counter
        senders = Counter()
        receivers = Counter()

        for tx in transactions:
            if tx.get('from'):
                senders[tx['from']] += 1
            if tx.get('to'):
                receivers[tx['to']] += 1

        # Identify significant counterparties
        significant = {
            'top_senders': senders.most_common(5),
            'top_receivers': receivers.most_common(5),
            'total_counterparties': len(set(list(senders.keys()) + list(receivers.keys())))
        }

        return significant

    def check_address_against_sanctions(self, crypto_address: str) -> dict:
        """
        Check a cryptocurrency address against OFAC crypto sanctions
        OFAC maintains a list of sanctioned cryptocurrency addresses
        """
        # OFAC publishes sanctioned crypto addresses at:
        # https://home.treasury.gov/policy-issues/financial-sanctions/recent-actions/20181128

        # Some platforms aggregate this:
        # Chainalysis (commercial), TRM Labs (commercial)
        # Free: search OFAC SDN list for virtual currency indicators

        return {
            'address': crypto_address,
            'note': 'Check OFAC SDN list for crypto addresses and commercial services like Chainalysis for comprehensive screening'
        }
```

---

## 21.6 Financial Crime Pattern Analysis with AI

```python
def analyze_financial_crime_indicators(entity_data: dict, corporate_structure: dict, financial_data: dict) -> str:
    """Use AI to analyze financial crime risk indicators"""

    import anthropic
    client = anthropic.Anthropic()

    prompt = f"""You are a certified anti-money laundering specialist conducting a risk assessment.

ENTITY INFORMATION:
{json.dumps(entity_data, indent=2)}

CORPORATE STRUCTURE:
{json.dumps(corporate_structure, indent=2)}

FINANCIAL INDICATORS:
{json.dumps(financial_data, indent=2)}

Analyze the above information for financial crime risk indicators including:

1. MONEY LAUNDERING RISK INDICATORS
   - Placement, layering, and integration patterns
   - Unusual corporate structure complexity
   - High-risk jurisdictions in ownership chain
   - Shell company indicators
   - Nominee director patterns

2. SANCTIONS RISK
   - Any indicators of sanctions nexus
   - Geographic risk factors

3. PEP EXPOSURE
   - Any indicators of politically exposed person involvement or association

4. BENEFICIAL OWNERSHIP CONCERNS
   - Opacity of ultimate beneficial ownership
   - Inconsistency between stated structure and operational reality

5. OVERALL RISK RATING
   [HIGH/MEDIUM/LOW] with justification

IMPORTANT:
- Base assessment strictly on provided information
- Clearly note what information was not available to complete the assessment
- Distinguish between red flags requiring investigation vs. confirmed risk
- Note that this is an OSINT assessment and not a conclusive finding of wrongdoing"""

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{"role": "user", "content": prompt}]
    )

    return response.content[0].text
```

---

## Summary

Financial crime and AML investigation is among the most systematically structured OSINT applications, driven by regulatory requirements that mandate specific due diligence processes. The core activities — beneficial ownership mapping, sanctions screening, PEP identification, and cryptocurrency transaction analysis — each have specialized data sources and methodologies.

Corporate structures used for money laundering have recognizable patterns: multi-jurisdictional layering through high-risk jurisdictions, nominee directors, shell companies without apparent business purpose, and circular ownership structures. Network analysis of corporate registries is particularly effective at revealing these patterns.

Cryptocurrency blockchain analysis leverages the inherent transparency of public blockchains. While sophisticated actors use mixing services and privacy coins to obscure trails, most criminal cryptocurrency activity leaves exploitable traces on public blockchain data.

Commercial tools (Chainalysis, Elliptic, World-Check) provide superior efficiency for production AML investigation, but open-source methods using free public databases provide significant capability for analysts without commercial tool access.

---

## Common Mistakes and Pitfalls

- **Jurisdiction ignorance**: Missing high-risk jurisdictions that are common money laundering channels
- **Nominal owner vs. beneficial owner confusion**: The registered owner of a corporate entity is often not the beneficial owner
- **Cryptocurrency address reuse assumption**: Cryptocurrency addresses are one-use; multiple transactions to the same address may not be related to the same counterparty
- **Screening list staleness**: Using outdated sanctions lists; lists are updated frequently and outdated screening creates compliance exposure
- **PEP scope too narrow**: PEPs include family members and close associates, not just the official themselves
- **Blockchain analysis tool over-reliance**: Without understanding blockchain analysis methodology, commercial tool outputs cannot be critically evaluated

---

## Further Reading

- FATF guidance documents on beneficial ownership and virtual assets
- FinCEN guidance on cryptocurrency AML obligations
- Basel AML Index — country risk rankings
- ICIJ Panama Papers and Pandora Papers methodology
- Chainalysis, TRM Labs, and Elliptic research publications on cryptocurrency crime
