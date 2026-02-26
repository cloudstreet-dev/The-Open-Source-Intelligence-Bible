# Chapter 28: Real-World Case Studies

## Learning Objectives

By the end of this chapter, you will be able to:
- Apply the methodologies from earlier chapters to realistic investigation scenarios
- Recognize how investigation techniques combine in practice across different domains
- Understand how investigations evolve as new evidence emerges
- Learn from case patterns that appear across multiple investigation types
- Appreciate how legal, ethical, and technical considerations interact in real investigations

---

## 28.1 Case Study Methodology

The case studies in this chapter are composite illustrations — they are not accounts of specific real investigations, but are constructed from realistic patterns observed across documented open-source investigations, published case studies, and the professional experience that informs this book. Each case demonstrates how multiple techniques integrate into a coherent investigative workflow.

Real investigations are messy in ways that textbook examples aren't. Leads go cold. Sources recant. Subjects move faster than investigators. Evidence turns out to be ambiguous. These case studies attempt to capture some of that messiness alongside the methodology.

---

## 28.2 Case Study 1: Shell Company Network Investigation

**Scenario**: A nonprofit organization has been approached by an investor offering significant funding. Due diligence is requested to assess whether the investor and their associated entities are legitimate.

### Initial Brief

```
Subject: Westbridge Capital Partners LLC
Primary contact: Listed as "Alexander Voronov" (CEO)
Jurisdiction: Delaware LLC, registered 2021
Investment amount: $2,500,000
Funding purpose: "Strategic partnership and market expansion"
```

### Phase 1: Entity Registration Research

```python
# Investigation begins with corporate registry research

investigation_notes = {
    "step_1": {
        "action": "OpenCorporates search for Westbridge Capital Partners",
        "result": {
            "found": True,
            "registered": "Delaware, 2021-03-15",
            "registered_agent": "National Registered Agents Inc. (generic agent)",
            "officers_listed": "None disclosed (Delaware does not require)",
            "status": "Active"
        },
        "assessment": "Delaware LLC with no disclosure requirements — need other avenues"
    },
    "step_2": {
        "action": "Search for Westbridge Capital Partners in SEC EDGAR",
        "result": {
            "found": False,
            "interpretation": "Not a registered investment advisor or broker-dealer"
        },
        "assessment": "No SEC registration — for a $2.5M investment fund, this warrants explanation"
    },
    "step_3": {
        "action": "Search FINRA BrokerCheck for Alexander Voronov",
        "result": {
            "found": False,
            "interpretation": "Not a registered broker or investment advisor"
        },
        "assessment": "Combination of no SEC/FINRA registration with $2.5M investment offer is a red flag"
    }
}
```

### Phase 2: Digital Footprint Research

```python
digital_research = {
    "website": {
        "url": "westbridgecapital.com",
        "whois": {
            "registered": "2021-02-28",  # Two weeks before LLC formation
            "registrar": "Namecheap",
            "privacy": "WhoisGuard protected",
            "hosting": "Cloudflare (masks origin IP)"
        },
        "content_assessment": "Professional-looking website, generic investment language",
        "reverse_image_search": {
            "action": "Reverse image search on team member photos",
            "result": "Alexander Voronov photo matches profile photo of different person on multiple sites"
        }
    },
    "linkedin": {
        "profile_found": True,
        "created": "2021-01",  # Shortly before corporate registration
        "connections": 47,
        "employment_history": "Claims 15 years in private equity, no verifiable companies",
        "activity": "Minimal — 3 posts since account creation",
        "synthetic_indicators": [
            "Low connection count for claimed experience level",
            "Profile photo suspected stolen (reverse image search)",
            "Employment history unverifiable through any public source"
        ]
    },
    "email_analysis": {
        "email_used": "avoronov@westbridgecapital.com",
        "domain_age": "2021-02-28",
        "spf_record": "Present",
        "dmarc": "Not configured — allows spoofing"
    }
}
```

### Phase 3: Adverse Media and Related Entity Research

```python
adverse_research = {
    "news_search": {
        "queries": [
            '"Westbridge Capital" fraud',
            '"Alexander Voronov" investor',
            '"Westbridge Capital Partners" SEC'
        ],
        "results": {
            "Westbridge Capital": "No results",
            "Alexander Voronov": "Results for different Alexander Voronov in different jurisdiction — not same person",
            "SEC": "No enforcement actions found"
        }
    },
    "related_entity_search": {
        "action": "Search for common registered agent address used by similar LLCs",
        "finding": "National Registered Agents Inc. hosts thousands of LLCs — not distinctive",
        "alternative": "Search for other companies with same contact email domain",
        "result": "Domain registered same month as two other 'capital partners' LLCs with similar naming patterns"
    },
    "icij_search": {
        "entities_checked": ["Westbridge Capital Partners", "Alexander Voronov"],
        "result": "No hits in Offshore Leaks Database"
    },
    "opensanctions": {
        "entities_checked": ["Westbridge Capital Partners LLC", "Alexander Voronov"],
        "result": "No hits on Alexander Voronov — the name is common enough that absence is not confirmation of legitimacy"
    }
}
```

### Phase 4: Synthesis and Assessment

```python
import anthropic

def synthesize_due_diligence(investigation_data: dict) -> str:
    client = anthropic.Anthropic()

    prompt = f"""You are a due diligence analyst reviewing investigation findings about a potential investor.

INVESTIGATION FINDINGS:
{str(investigation_data)}

Provide a structured assessment covering:

1. RED FLAGS: What concerns have been identified?
2. LEGITIMATE INDICATORS: What suggests legitimacy?
3. UNRESOLVED QUESTIONS: What could not be verified?
4. RISK ASSESSMENT: Overall risk rating and reasoning
5. RECOMMENDED NEXT STEPS: What additional verification should be done before accepting funding?

Be direct and evidence-based. Do not soften concerns that warrant serious attention."""

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1500,
        messages=[{"role": "user", "content": prompt}]
    )
    return response.content[0].text

# Synthesized findings in this case:
# - Stolen profile photo is a near-disqualifying red flag
# - No regulatory registration for claimed investment activity
# - Coordinated LLC creation pattern with similar entities
# - All digital presence dates from a single 6-week window in early 2021
# Recommendation: Do not accept funding; refer to law enforcement if investor persists
```

**Investigation outcome**: The reverse image search finding — the profile photo belonging to a different person — was the decisive evidence. Combined with no regulatory registration and a synthetic-looking digital footprint created in a compressed timeframe, the recommendation was to decline the investment and document the interaction for potential fraud report filing.

---

## 28.3 Case Study 2: Geolocation Verification

**Scenario**: A video clip is circulating on social media purporting to show military activity at a location claimed to be in a conflict zone. Editors need verification before publication.

### Initial Assessment

The video shows:
- Military vehicles on a road
- Distinctive road markings
- A partial building facade visible in background
- Vegetation consistent with temperate climate
- Time visible on dashboard: 14:37
- Sun appears to be ahead and to the right of vehicle direction

### Geolocation Methodology

```python
geolocation_analysis = {
    "sun_analysis": {
        "visible_sun_position": "Ahead and right of vehicle",
        "time_on_dashboard": "14:37",
        "inference": {
            "if_heading_north": "Sun would be ahead-right in northern hemisphere late afternoon",
            "hemisphere_indicator": "Northern hemisphere, late afternoon",
            "approximate_latitude": "40-55 degrees N based on sun elevation angle"
        }
    },

    "vegetation_analysis": {
        "trees_visible": "Deciduous, full foliage",
        "inference": {
            "climate": "Temperate, continental",
            "season": "Late spring to early fall",
            "rules_out": "Tropical, desert, or polar regions"
        }
    },

    "road_markings": {
        "center_line": "Dashed white (not yellow)",
        "edge_line": "White",
        "markings_style": "Consistent with Eastern European road standards",
        "notes": "Yellow center lines common in US/Canada; white in Europe and much of Asia"
    },

    "building_facade": {
        "visible_features": [
            "Stucco exterior finish",
            "Distinctive window pattern — rectangular with central divider",
            "Partial Cyrillic text visible on sign"
        ],
        "inference": "Cyrillic text confirms Eastern European/Eurasian location"
    },

    "vehicle_identification": {
        "vehicles_visible": "Military trucks with visible chassis type",
        "wheel_configuration": "6x6 all-terrain truck",
        "potential_matches": "Multiple former-Soviet truck models",
        "distinctive_marking": "Unit insignia partially visible — requires specialist identification"
    },

    "shadow_analysis": {
        "shadow_direction": "Slightly toward the left of frame",
        "sun_azimuth_estimate": "Roughly 210-230 degrees (SW to WSW)",
        "combined_with_time": {
            "sun_azimuth_sw_at_14:37": "Consistent with latitude ~50-55N in summer"
        }
    }
}

# Reverse image search and geolocation
search_queries = [
    "site:google.com/maps [distinctive building features]",
    "[road markings] [vegetation type] [region claimed]",
    "Yandex reverse image search with still frames"
]

# Cross-reference with claimed location
verification_status = {
    "consistent_with_claimed_location": True,
    "confidence": "MEDIUM",
    "reasons": [
        "Cyrillic text confirmed (consistent with claimed region)",
        "Sun position consistent with claimed date and time at claimed coordinates",
        "Road markings consistent with country of claimed location",
        "Vegetation consistent with season and latitude"
    ],
    "remaining_uncertainty": [
        "Could be anywhere with Cyrillic text and similar terrain",
        "Vehicle type not definitively identified",
        "Date cannot be independently confirmed from video metadata"
    ]
}
```

**Investigation outcome**: The geolocation analysis supported the claimed location with medium confidence. The Cyrillic text narrowed the region significantly; the sun analysis was consistent with the claimed date and coordinates. Recommendation was to publish with caveats describing verification methodology and confidence level.

---

## 28.4 Case Study 3: Social Media Account Network Analysis

**Scenario**: An investigation team suspects a coordinated social media campaign is amplifying specific political messaging. They have collected 50 accounts that regularly engage with each other and share similar content.

```python
import networkx as nx
from datetime import datetime
from collections import Counter
from typing import List, Dict

def analyze_coordination_network(accounts: List[Dict], posts: List[Dict]) -> Dict:
    """
    Analyze accounts for coordinated inauthentic behavior
    """

    G = nx.DiGraph()
    findings = {
        'creation_clustering': {},
        'content_similarity': {},
        'behavioral_patterns': {},
        'network_structure': {},
        'overall_assessment': {}
    }

    # 1. Account creation date analysis
    creation_dates = []
    for account in accounts:
        if account.get('created_at'):
            try:
                date = datetime.fromisoformat(account['created_at'].replace('Z', '+00:00'))
                creation_dates.append(date)
                G.add_node(account['username'], created=date.isoformat())
            except ValueError:
                pass

    if creation_dates:
        creation_dates.sort()
        # Find clusters of accounts created within 7-day windows
        clusters = []
        window = []
        for date in creation_dates:
            if not window or (date - window[0]).days <= 7:
                window.append(date)
            else:
                if len(window) > 2:
                    clusters.append(window)
                window = [date]
        if len(window) > 2:
            clusters.append(window)

        findings['creation_clustering'] = {
            'total_accounts': len(accounts),
            'creation_clusters': len(clusters),
            'clustered_accounts': sum(len(c) for c in clusters),
            'largest_cluster_size': max(len(c) for c in clusters) if clusters else 0,
            'assessment': 'HIGH RISK' if (sum(len(c) for c in clusters) / len(accounts)) > 0.5 else 'MEDIUM'
        }

    # 2. Retweet/amplification network
    for post in posts:
        if post.get('retweeted_from'):
            source = post.get('retweeted_from')
            amplifier = post.get('username')
            if G.has_node(source) and G.has_node(amplifier):
                if G.has_edge(amplifier, source):
                    G[amplifier][source]['weight'] += 1
                else:
                    G.add_edge(amplifier, source, weight=1)

    if len(G.edges()) > 0:
        # Analyze network structure
        density = nx.density(G)
        avg_degree = sum(d for n, d in G.degree()) / len(G)

        # High density in small network = coordinated
        findings['network_structure'] = {
            'nodes': len(G.nodes()),
            'edges': len(G.edges()),
            'density': density,
            'avg_degree': avg_degree,
            'density_assessment': 'HIGH (consistent with coordination)' if density > 0.3 else 'MEDIUM'
        }

        # Find central accounts
        try:
            betweenness = nx.betweenness_centrality(G)
            top_central = sorted(betweenness.items(), key=lambda x: x[1], reverse=True)[:5]
            findings['network_structure']['central_accounts'] = [
                {'account': k, 'centrality': v} for k, v in top_central
            ]
        except Exception:
            pass

    # 3. Posting time analysis — look for human-inconsistent posting patterns
    post_hours = Counter()
    account_post_hours = {}

    for post in posts:
        username = post.get('username')
        posted_at = post.get('posted_at')
        if posted_at:
            try:
                dt = datetime.fromisoformat(posted_at.replace('Z', '+00:00'))
                hour = dt.hour
                post_hours[hour] += 1
                if username not in account_post_hours:
                    account_post_hours[username] = Counter()
                account_post_hours[username][hour] += 1
            except ValueError:
                pass

    # Check for accounts posting at 3-5am (common for bot accounts running on UTC)
    overnight_posters = []
    for username, hours in account_post_hours.items():
        late_night_posts = sum(hours[h] for h in range(2, 6))
        total_posts = sum(hours.values())
        if total_posts > 5 and late_night_posts / total_posts > 0.3:
            overnight_posters.append({
                'account': username,
                'late_night_ratio': late_night_posts / total_posts
            })

    findings['behavioral_patterns'] = {
        'overnight_posting_accounts': overnight_posters,
        'total_accounts_analyzed': len(account_post_hours),
        'assessment': 'SUSPICIOUS' if len(overnight_posters) > 5 else 'NORMAL'
    }

    # 4. Content similarity — find accounts sharing identical or near-identical content
    post_texts = {}
    for post in posts:
        text = post.get('text', '')
        if len(text) > 50:  # Skip very short posts
            username = post.get('username')
            if username not in post_texts:
                post_texts[username] = []
            post_texts[username].append(text[:200])  # First 200 chars

    # Find exact duplicate posts across accounts
    all_texts = []
    for username, texts in post_texts.items():
        for text in texts:
            all_texts.append((username, text))

    text_counter = Counter(text for _, text in all_texts)
    coordinated_posts = [(text, count) for text, count in text_counter.items() if count > 3]

    findings['content_similarity'] = {
        'identical_posts_shared_by_3_plus': len(coordinated_posts),
        'examples': coordinated_posts[:5],
        'assessment': 'HIGH RISK' if len(coordinated_posts) > 5 else 'MEDIUM'
    }

    # Overall assessment
    risk_factors = sum([
        findings['creation_clustering'].get('assessment', '') == 'HIGH RISK',
        findings['behavioral_patterns'].get('assessment', '') == 'SUSPICIOUS',
        findings['content_similarity'].get('assessment', '') == 'HIGH RISK',
        findings.get('network_structure', {}).get('density_assessment', '').startswith('HIGH')
    ])

    findings['overall_assessment'] = {
        'risk_factors_count': risk_factors,
        'coordination_likelihood': 'HIGH' if risk_factors >= 3 else ('MEDIUM' if risk_factors >= 2 else 'LOW'),
        'recommendation': 'Report to platform trust & safety team and document findings' if risk_factors >= 2 else 'Continue monitoring'
    }

    return findings
```

**Investigation outcome**: Analysis revealed 73% of accounts created within a 14-day window two months before a major election, posting density of 0.47 (very high for an organic network), and 12 accounts posting between 3:00-5:00 AM UTC at rates inconsistent with human behavior. Findings were reported to the platform's trust and safety team with full documentation.

---

## 28.5 Case Study 4: Financial Fraud Investigation

**Scenario**: A company's finance team has flagged unusual wire transfers to unfamiliar vendors. The OSINT component of the fraud investigation involves researching the receiving entities.

```python
fraud_investigation = {
    "initial_flags": {
        "transfer_1": {
            "amount": 47500,
            "recipient": "Pacific Rim Consulting Ltd",
            "bank": "SWIFT transfer to Hong Kong correspondent",
            "invoice_description": "Advisory services Q3",
            "matching_po": False
        },
        "transfer_2": {
            "amount": 49800,
            "recipient": "Pacific Rim Consulting Ltd",
            "bank": "Same correspondent bank",
            "invoice_description": "Advisory services Q4",
            "timing": "Two transfers just below $50,000 threshold — structuring indicator"
        }
    },

    "entity_research": {
        "Pacific Rim Consulting Ltd": {
            "hong_kong_registry": {
                "found": True,
                "incorporated": "2022-01-15",
                "address": "Virtual office, shared with 200+ other companies",
                "directors": ["Zhang Wei", "Li Mingzhi"],
                "status": "Active"
            },
            "uk_companies_house": {
                "found": False
            },
            "opencorporates_global": {
                "found": True,
                "notes": "Single entity in HK; no global parent company"
            },
            "web_presence": {
                "website": "Not found",
                "linkedin": "Not found",
                "news": "No results"
            },
            "assessment": "Entity appears to exist solely on paper; no operating presence"
        }
    },

    "transaction_pattern_analysis": {
        "structuring_indicators": [
            "Both transfers just below $50,000 BSA reporting threshold",
            "Same recipient across multiple transfers",
            "No matching purchase orders in ERP system",
            "Invoice dates fall on weekend"
        ],
        "internal_access_indicators": [
            "Transfers authorized by single employee without secondary approval (process violation)",
            "Employee has access to both AP system and banking portal"
        ]
    },

    "director_research": {
        "Zhang Wei": {
            "note": "Extremely common name — difficult to research without additional identifiers",
            "action": "Request from HK registry for director details including ID number"
        },
        "Li Mingzhi": {
            "opencorporates_officers": "Appears as director in 14 other HK companies, 12 incorporated in 2021-2022",
            "assessment": "Serial director profile — common in nominee director services"
        }
    },

    "recommendation": {
        "immediate_actions": [
            "Preserve all financial records and communications",
            "Do not alert the authorizing employee",
            "Engage forensic accounting firm",
            "Refer to law enforcement for potential SAR filing"
        ],
        "osint_findings_summary": "Recipient entity shows classic characteristics of a shell company used for fraudulent billing: virtual office address, nominee directors, no public operating presence, recently incorporated, no web footprint, and transaction structuring pattern consistent with BSA avoidance"
    }
}
```

---

## 28.6 Lessons Across Case Studies

Reviewing these cases reveals recurring patterns:

**Compressed timelines are suspicious**: In Case Study 1, all of the investor's digital presence was created within a 6-week window. In Case Study 4, the recipient company was incorporated 6 months before the fraud. Legitimate businesses develop over years; fraudulent entities often appear fully-formed just before they're needed.

**Cross-source verification is essential**: Every case required more than one source. The geolocation case combined sun analysis, vegetation, road markings, and Cyrillic text — any one indicator alone was insufficient. The coordination case required creation dates, network analysis, posting patterns, AND content similarity.

**Absence of evidence is informative**: The investor in Case Study 1 had no regulatory registration for a $2.5M offering. The fraud recipient in Case Study 4 had no web presence, no LinkedIn, no news coverage. Legitimate large-scale financial actors leave tracks.

**Reverse image search remains underused**: The decisive finding in Case Study 1 — the stolen profile photo — took two minutes with reverse image search. This is among the most high-yield techniques in persona verification.

**Document continuously**: In every case, the documentation of methodology is as important as the findings. Investigators who cannot explain how they reached a conclusion have findings that are easily impeached.

---

## Summary

Real investigations integrate techniques from throughout this book into coherent workflows adapted to specific evidence. The methodology is consistent — pivot from known entities, cross-verify across multiple sources, document everything, calibrate confidence — while the specific tools and sources vary by case type.

Case studies are the best learning vehicle in OSINT because they show technique in the context of real investigative pressure: incomplete evidence, false leads, time constraints, and the requirement to reach conclusions with appropriate confidence rather than certainty.

---

## Common Mistakes and Pitfalls

- **Single-source conclusions**: No single finding, no matter how compelling, should drive a conclusion without corroboration
- **Availability bias**: Researching only the sources you're familiar with, rather than systematically covering all relevant sources
- **Overfitting to the hypothesis**: Interpreting ambiguous evidence as confirmation of what you already believe
- **Neglecting timestamp verification**: Images and videos can be repurposed; always verify temporal claims
- **Treating absence of evidence as evidence of absence**: Not finding something in a database is not the same as it not existing

---

## Further Reading

- Bellingcat — open source investigations with published methodology
- Global Investigative Journalism Network (GIJN) — investigative case studies and methodology
- OCCRP — organized crime and corruption investigation case studies
- ProPublica and The Intercept — published investigative methodology notes
- ICIJ offshore leak investigations — case studies in financial structure research
