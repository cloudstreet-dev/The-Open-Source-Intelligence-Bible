# Appendix A: Tool Reference Guide

This appendix provides a structured reference to the tools discussed throughout the book, organized by function category. For each tool, the reference includes: primary function, access model (free/freemium/paid), API availability, and the chapter(s) where it is discussed.

---

## A.1 Social Media Intelligence Tools

| Tool | Primary Function | Access | API | Chapter |
|---|---|---|---|---|
| Sherlock | Username search across 300+ platforms | Free/OSS | N/A (CLI) | 5, 15 |
| Maigret | Username OSINT with detailed profile reports | Free/OSS | N/A (CLI) | 5, 15 |
| Holehe | Check email against platforms | Free/OSS | N/A (CLI) | 5 |
| Instaloader | Instagram downloader (public profiles) | Free/OSS | N/A (CLI) | 5, 15 |
| Twint | Twitter scraping without API | Free/OSS | N/A | 5 |
| PRAW | Python Reddit API wrapper | Free | Yes (Reddit API) | 5, 14 |
| Meltwater | Media monitoring and social listening | Paid | Yes | 16 |
| Brandwatch | Social analytics and listening | Paid | Yes | 16 |

---

## A.2 Domain and Network Intelligence

| Tool | Primary Function | Access | API | Chapter |
|---|---|---|---|---|
| Shodan | Internet-connected device search engine | Free tier + Paid | Yes | 6, 15, 19 |
| Censys | Internet scan and certificate data | Free tier + Paid | Yes | 6, 15 |
| SecurityTrails | Historical DNS and domain intelligence | Free tier + Paid | Yes | 6, 15 |
| DomainTools | WHOIS intelligence and domain history | Paid | Yes | 6, 15 |
| crt.sh | Certificate Transparency log search | Free | JSON endpoint | 6, 15, 19 |
| ViewDNS | DNS and IP tools | Free + Paid | Yes | 6 |
| Subfinder | Passive subdomain discovery | Free/OSS | N/A (CLI) | 15, 19 |
| Amass | Attack surface mapping | Free/OSS | N/A (CLI) | 6, 15 |
| dnspython | DNS queries in Python | Free/OSS | Python library | 6 |
| ipwhois | IP WHOIS in Python | Free/OSS | Python library | 6 |

---

## A.3 Search and Historical Data

| Tool | Primary Function | Access | API | Chapter |
|---|---|---|---|---|
| Google Advanced Search | Operator-based web search | Free | Limited | 9 |
| Wayback Machine (CDXAPI) | Historical web archive access | Free | Yes | 9, 15 |
| archive.today | On-demand web archival | Free | N/A | 9 |
| Intelligence X | Deep web and breach data search | Free tier + Paid | Yes | 9 |
| PasteBin / GhostBin | Paste site search | Free | Limited | 9 |
| ExifTool | Metadata extraction from files | Free/OSS | CLI / library | 9, 15 |
| Maltego | Link analysis and data aggregation | Free tier + Paid | Transform API | 15, 16 |

---

## A.4 Public Records and Corporate Data

| Tool | Primary Function | Access | API | Chapter |
|---|---|---|---|---|
| OpenCorporates | Global company registry search | Free + Paid | Yes | 7, 18, 21, 22 |
| SEC EDGAR | US public company filings | Free | Yes (EFTS) | 7, 22 |
| UK Companies House | UK corporate registry | Free | Yes | 7, 15 |
| PACER | US federal court records | Paid (per page) | Limited | 7, 18 |
| CourtListener | Federal case law and PACER mirror | Free | Yes | 7, 18, 22 |
| USASpending.gov | US federal contracts and grants | Free | Yes | 7 |
| FINRA BrokerCheck | Financial services registration | Free | Unofficial | 7, 18, 22 |
| FEC API | US campaign finance data | Free | Yes | 7 |
| ICIJ Offshore Leaks | Offshore corporate data from leaks | Free | JSON | 21, 22 |
| OpenSanctions | Global sanctions and watchlists | Free + Paid API | Yes | 21, 22, 24 |

---

## A.5 Geospatial and Satellite Intelligence

| Tool | Primary Function | Access | API | Chapter |
|---|---|---|---|---|
| Google Earth Pro | Satellite imagery (historical) | Free | N/A | 8, 15 |
| Sentinel Hub | Copernicus satellite imagery | Free + Paid | Yes | 8, 15 |
| Planet Labs | Commercial daily satellite imagery | Paid | Yes | 8, 15 |
| Maxar | Sub-meter commercial imagery | Paid | Yes | 8 |
| MarineTraffic | AIS vessel tracking | Free + Paid | Yes | 8, 15 |
| FlightAware | Flight tracking | Free + Paid | Yes | 8, 15 |
| ADS-B Exchange | Unfiltered ADS-B flight data | Free | Yes | 8 |
| OpenSky Network | Research-grade ADS-B | Free (research) | Yes | 8 |
| QGIS | GIS desktop application | Free/OSS | Python API | 8, 15 |
| astral | Solar position calculations | Free/OSS | Python library | 8 |
| Folium | Python map visualization | Free/OSS | Python library | 8, 15 |

---

## A.6 AI and NLP Tools

| Tool | Primary Function | Access | API | Chapter |
|---|---|---|---|---|
| Anthropic API (Claude) | LLM for analysis, synthesis, coding | Paid | Yes | 10-12, throughout |
| OpenAI API (GPT-4o) | LLM for analysis | Paid | Yes | 10, 25 |
| HuggingFace Transformers | NLP models (NER, classification, etc.) | Free/OSS | Python library | 11 |
| spaCy | Production NLP pipeline | Free/OSS | Python library | 11, 14 |
| Ollama | Local LLM inference | Free/OSS | Local API | 27 |
| Whisper | Audio transcription | Free/OSS | Python library | 25 |
| sentence-transformers | Text embeddings | Free/OSS | Python library | 11 |

---

## A.7 Data Storage and Processing

| Tool | Primary Function | Access | API | Chapter |
|---|---|---|---|---|
| Elasticsearch | Full-text search and analytics | Free/OSS + Paid | Yes | 23, 27 |
| PostgreSQL | Relational database | Free/OSS | SQL | 23, 27 |
| Neo4j | Graph database | Free/OSS (Community) + Paid | Cypher / REST | 13, 27 |
| Redis | Cache, session storage, pub/sub | Free/OSS + Paid | Yes | 23, 27 |
| Celery | Distributed task queue | Free/OSS | Python library | 14, 23, 27 |
| Apache Kafka | Distributed event streaming | Free/OSS | Yes | 23, 27 |
| Apache Airflow | Workflow orchestration | Free/OSS | Yes | 23, 27 |
| SQLite | Embedded relational database | Free/OSS | Python built-in | 14, 27 |

---

## A.8 Visualization and Reporting

| Tool | Primary Function | Access | API | Chapter |
|---|---|---|---|---|
| Plotly | Interactive charts and graphs | Free/OSS | Python library | 17, 27 |
| NetworkX | Graph analysis (Python) | Free/OSS | Python library | 5, 13, 17 |
| Gephi | Graph visualization (desktop) | Free/OSS | N/A | 13, 15 |
| Maltego | Link analysis and visualization | Free tier + Paid | Transform API | 15, 16 |
| Hunchly | Web investigation capture | Paid | N/A | 4, 15, 18 |
| TimelineJS | Browser-based timeline | Free/OSS | N/A | 17 |

---

## A.9 Security and OPSEC Tools

| Tool | Primary Function | Access | API | Chapter |
|---|---|---|---|---|
| VirusTotal | File and URL threat intelligence | Free + Paid | Yes | 20, 27 |
| AbuseIPDB | IP reputation database | Free + Paid | Yes | 20, 23 |
| URLhaus | Malware URL database | Free | Yes | 20 |
| MISP | Threat intelligence platform | Free/OSS | Yes | 20 |
| Tor Browser | Anonymizing browser | Free/OSS | N/A | 15, 26 |
| VeraCrypt | Full-disk and volume encryption | Free/OSS | N/A | 26, 27 |
| Signal | End-to-end encrypted messaging | Free/OSS | N/A | 26 |
| Tails OS | Amnesic operating system | Free/OSS | N/A | 26 |
| Canarytokens | Honeypot/canary tokens | Free | N/A | 24 |

---

## A.10 API Cost Reference (Approximate, 2024-2025)

| Service | Free Tier | Paid Tier |
|---|---|---|
| Shodan | 100 results/month | $49-149/month |
| SecurityTrails | 50 queries/month | $199+/month |
| Hunter.io | 25 searches/month | $49+/month |
| VirusTotal | 4 req/min | $200+/month (enterprise) |
| NewsAPI | 100 req/day | $449/month |
| Anthropic API | None (pay per use) | ~$3-15/MTok (model dependent) |
| OpenAI API | None (pay per use) | ~$2.50-10/MTok |
| OpenCorporates | 100 req/month | Contact for pricing |
| Sentinel Hub | 30,000 processing units | $22+/month |

*Costs change frequently. Verify current pricing before budgeting.*

---

## A.11 Tool Evaluation Checklist

Before adding a tool to your stack, evaluate:

- [ ] **Authorization**: Does using this tool comply with relevant ToS, licensing, and applicable law?
- [ ] **Data quality**: How fresh is the data? What is the coverage and accuracy rate?
- [ ] **Rate limits**: What are the limits and how do they affect workflow design?
- [ ] **API stability**: Does the provider have a stable API with versioning and deprecation notice?
- [ ] **Privacy handling**: Where does collected data go? Does the tool log your queries?
- [ ] **Cost predictability**: Can you bound the cost at expected usage volumes?
- [ ] **Alternatives**: Is there a free or open-source alternative that meets the requirement?
- [ ] **Longevity**: Is the provider financially stable? What's the shutdown risk?
