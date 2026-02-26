# Chapter 2: The Data Landscape and Sources

## Learning Objectives

By the end of this chapter, you will be able to:
- Map the modern data landscape into structured source categories
- Evaluate the characteristics, strengths, and limitations of each major source type
- Understand how data flows from generation to public availability
- Identify the right data sources for common investigative requirements
- Assess data quality, freshness, and reliability across source types
- Understand the role of data brokers and commercial aggregators in the OSINT ecosystem

---

## 2.1 The Architecture of Open Data

To investigate effectively, you must first understand where data lives — how it is generated, stored, indexed, and made accessible. The instinct of many new OSINT practitioners is to reach immediately for a search engine. But Google indexes a small fraction of the publicly available information space. Understanding the full architecture of open data is what separates systematic investigation from surface-level searching.

Think of the data landscape as a three-dimensional space. The first dimension is **source type**: where the data comes from originally. The second dimension is **access method**: how you retrieve it. The third dimension is **data quality**: how reliable, complete, and current it is.

Most OSINT methodology frameworks address source type. Few address access method and data quality with equal rigor. All three dimensions matter for building effective investigative workflows.

---

## 2.2 Surface Web: The Indexed Internet

The surface web — content indexed by search engines like Google, Bing, and DuckDuckGo — is the most accessible tier of open data. It is also the most competitive: everyone with an internet connection can search it.

### What Lives on the Surface Web

**News and media**: Journalism from thousands of outlets worldwide, in dozens of languages, going back years in most archives. News coverage is a primary source for biographical information, organizational histories, and event documentation.

**Government websites**: Federal, state, and local government websites publish an enormous volume of useful data — regulations, agency reports, contractor award databases, meeting minutes, published investigations, and official statistics.

**Corporate web presence**: Company websites, press releases, investor relations pages, annual reports, and SEC or equivalent regulatory filings.

**Academic and research publications**: Journal articles, conference papers, thesis repositories, and preprint servers (arXiv, SSRN, bioRxiv). These are often overlooked as OSINT sources but are invaluable for technical intelligence and background research.

**Social media public profiles**: Publicly indexed social media content — LinkedIn profiles, public Twitter/X posts, public Facebook pages, YouTube channels, Instagram public accounts.

**Professional directories**: Bar association member directories, medical licensing databases, professional certification directories, contractor licensing registries.

### Surface Web Limitations

**Index incompleteness**: Even Google's index, the world's largest, covers an estimated 4-16% of the total web. Content that is not linked to from other indexed pages, content requiring JavaScript rendering, and content on platforms that restrict crawling may not be indexed.

**Temporal lag**: Search engine indexes are not real-time. New content may take days or weeks to appear. Deleted content may persist in the index for longer than it persists on its original server.

**Personalization and filter bubbles**: Search results are personalized and vary by user, location, search history, and other factors. Investigators using personalized accounts for OSINT searches may receive systematically different results than a neutral observer.

**SEO distortion**: Search ranking algorithms can be gamed. Well-resourced individuals and organizations can optimize results to suppress negative content and promote favorable content. Search result ordering is not a proxy for information quality.

### Effective Surface Web Strategy

- Use multiple search engines, not just Google
- Use site-specific searches (site:example.com) to index deep within specific domains
- Use operator queries to refine results (see Chapter 9 on advanced search)
- Disable personalization where possible — use private browsing, logged-out sessions, or dedicated research profiles
- Use search engine caches and the Wayback Machine to access deleted content
- Search in multiple languages for subjects with international dimensions

---

## 2.3 The Deep Web: Accessible But Not Indexed

The "deep web" describes web content that exists on the internet but is not indexed by search engines. The deep web is enormous — estimates suggest it is hundreds of times larger than the indexed web. Most of it is mundane: online banking portals, corporate intranets, email servers, database query results.

The investigatively relevant portion of the deep web includes:

### Public Records Systems

Government agencies at every level maintain online public records access systems that are not search-engine indexed:

**Court records**: PACER (U.S. federal courts), state court e-filing systems, international court records. These contain litigation histories, bankruptcy filings, criminal cases, and civil disputes.

**Property records**: County assessor databases, deed registries, mortgage records. Most U.S. counties have online property search systems accessible without a search engine.

**Business registrations**: Secretary of State databases in all 50 U.S. states, Companies House in the UK, national business registries in most countries. These reveal entity structures, registered agents, officer names, and filing histories.

**UCC filings**: Uniform Commercial Code filing databases record secured transactions — when someone pledges assets as collateral for a loan. These reveal financial relationships between businesses and individuals.

**Voter registration records**: In many U.S. states, voter registration data is public and includes home addresses, party affiliation, and voting history.

**Professional licensing**: State licensing boards for physicians, attorneys, engineers, contractors, financial advisors, and other regulated professions maintain searchable databases.

**Regulatory filings**: The SEC EDGAR database, FINRA BrokerCheck, the FEC's campaign finance database, and equivalent regulatory bodies publish detailed filings not indexed by search engines.

### Database Query Interfaces

Many organizations publish data through query interfaces rather than static pages. Because the results are dynamically generated, they are not indexed:

- Aviation registry databases (FAA, international equivalents)
- Vessel registration and AIS databases
- Patent and trademark databases (USPTO, EPO, WIPO)
- Domain registration WHOIS (before GDPR restrictions)
- Academic journal databases accessible via institutional login or open access portals

### Accessing the Deep Web Effectively

The deep web requires knowing what to look for and where to look. There is no general index. Key strategies:

- **Know the sources**: Build and maintain a reference library of deep web sources relevant to your investigation domains (see Appendix A)
- **Use meta-databases**: Sites like PublicRecordsNow, BRB Publications, or Black Book Online aggregate links to public records databases
- **Work backwards from entities**: Start with a known entity (a person, company, or address) and systematically query relevant databases
- **Use API access where available**: Many government databases offer API access that enables systematic, programmatic querying

---

## 2.4 Social Media Data: The Primary Modern OSINT Source

Social media has become the most fertile source of OSINT data for most investigations. People voluntarily share personal information, social connections, location data, behavioral patterns, and opinions in ways they often do not consider from an investigative standpoint.

### Platform Characteristics

Each major social platform has distinct characteristics that affect its investigative utility:

**Facebook/Meta**: Despite declining active users among younger demographics, Facebook has the largest total user base globally and the longest history of user data. Family networks, life events, check-ins, and community memberships make it particularly useful for building relationship maps and establishing biographical details. API access has been severely restricted since Cambridge Analytica.

**Instagram**: Visually rich, with geotagged content, story archives, and highly personal documentation of activities, locations, and relationships. Particularly valuable for subjects who are active visual communicators. Public profiles accessible without authentication; private profiles require following approval.

**Twitter/X**: Historically the richest source of real-time and historical text data for OSINT. The platform's API changes under Elon Musk's ownership dramatically curtailed free research access. But public tweet content, follower graphs, and engagement patterns remain valuable. The platform is particularly important for political research, journalist investigation, and tracking public discourse.

**LinkedIn**: Professional network with high-quality biographical data — employment history, education, professional connections, skills, endorsements. Particularly valuable for corporate investigations, due diligence, and background research. Resists automated scraping aggressively. Data tends to be more accurate than other platforms because professional reputation depends on accuracy.

**TikTok**: Primarily video-based, with younger demographics. Valuable for investigations involving youth communities, trend analysis, and video content analysis. Metadata is limited; geographic data is often coarse.

**Reddit**: Pseudonymous discussion platform with searchable post and comment history. Valuable for research into specific communities, tracking narratives, and finding discussion of specific topics. Username-based identity enables longitudinal analysis of user behavior.

**Telegram**: Messaging platform with large public group channels. Significant volumes of extremist, criminal, and politically sensitive content. Public channels can be accessed without authentication. A critical source for threat intelligence and extremism research.

**Discord**: Gaming and community communication platform with public servers. Public server content can be partially accessed; some servers require invitation. Important for youth communities, gaming investigations, and specific interest group research.

**Snapchat**: Ephemeral content by design, but location features (Snap Map) and public Stories provide geospatial intelligence opportunities. Content does not persist, limiting historical investigation.

### Social Media Collection Approaches

**Direct platform access**: Manual review of public profiles, posts, and connections. Time-consuming but controllable and clearly within platform terms.

**Official APIs**: Where still available, the most reliable access method. Rate-limited and increasingly restricted. Requires developer account registration.

**Third-party tools**: Tools like Social Mapper, Sherlock, Maltego social modules, and commercial platforms provide aggregated social media intelligence but are subject to platform API terms.

**Web scraping**: Automated extraction of public page content. Subject to platform terms of service (typically prohibited) and technical countermeasures. Not appropriate for evidence that must be legally defensible.

**Cached and archived content**: The Wayback Machine, Google cache, and social media archiving services preserve content that has been deleted from live platforms. Critical for historical investigation.

### The Social Media Data Quality Problem

Social media data quality is variable in ways that matter for investigation:

**Identity verification is weak**: Most platforms do not verify user identity. Profile information may be fabricated, belong to a different person, or represent a coordinated inauthentic account.

**Engagement can be artificial**: Follower counts, likes, and reshares can be purchased or artificially generated. Metrics that appear to indicate reach or influence may be manipulated.

**Content can be fabricated**: Text, images, and video can be manipulated. Deep fakes, edited screenshots, and contextually misleading media are common.

**Temporal accuracy is unreliable**: Timestamps may not accurately reflect when content was created. Reuploaded content may carry original timestamps or no timestamps.

**Platforms actively resist investigation**: Rate limiting, authentication requirements, and anti-automation measures are deliberately designed to slow investigative access.

---

## 2.5 Commercial Data Brokers and Aggregators

One of the most significant but least understood components of the OSINT ecosystem is the commercial data broker industry — companies that collect, aggregate, enhance, and resell personal information.

### How Data Brokers Work

Data brokers aggregate information from dozens to hundreds of sources:

- Public records (property records, court records, business filings, voter registration)
- Social media profiles
- Retail loyalty program data
- Online behavioral tracking data
- Credit header data (non-credit-score portions of credit files)
- Telephone directories
- Magazine subscription lists
- Warranty card registrations

This data is enhanced with inferred attributes (estimated income, political affiliation, purchasing propensity) and linked across records to build comprehensive profiles. The result is a product that individual OSINT investigators could not replicate through direct public records research.

### Key Commercial Data Broker Categories

**People search databases**: TLO, LexisNexis, IRB Search, TransUnion TLOxp, Accurint (LexisNexis). These provide the richest linked personal profiles with historical data, address history, associated persons, vehicle records, and much more. Access typically requires professional licensing (PI, attorney, law enforcement) or enterprise subscription.

**Business intelligence databases**: Dun & Bradstreet, Experian Business, Bureau van Dijk/Orbis. These provide company financials, officer information, subsidiary structures, and credit information. Available via subscription.

**Consumer people-search sites**: Spokeo, BeenVerified, Intelius, Whitepages, FastPeopleSearch. These provide partial views of the same underlying data available through professional tools, with less accuracy and fewer restrictions. Available to anyone.

**Social media data aggregators**: Various companies aggregate and sell social media data — often in gray legal territory given platform terms of service. Data quality and access methods vary significantly.

**Real estate data**: Zillow, Redfin, ATTOM, CoStar. Property transaction data, estimated values, owner information, and listing history. Some free, some subscription-based.

### Data Broker Quality and Accuracy

Commercial people-data is less accurate than practitioners often assume. Studies have found error rates of 10-40% in address history, relationship mapping, and biographical details in major commercial databases. The errors compound as data is aggregated from multiple sources, each with its own error rates.

This does not make commercial data less useful — it makes verification essential. Commercial data is best used to generate leads and hypothesis that are then verified against primary sources.

### Legal and Ethical Considerations

The use of commercial data broker data carries legal restrictions:

**FCRA compliance**: Data used for employment screening, credit decisions, insurance underwriting, or tenant screening must comply with the Fair Credit Reporting Act. This requires specific procedures, adverse action notices, and consumer rights compliance.

**GDPR**: EU residents have rights under GDPR that limit how their data can be collected and used. Using data broker data about EU persons for investigative purposes may implicate GDPR.

**Platform terms**: Consumer people-search sites typically prohibit use for stalking, harassment, or other harmful purposes. Professional data broker subscriptions require representation of lawful purpose.

---

## 2.6 Government and Public Records

Government records represent one of the most reliable and legally unambiguous OSINT sources. Governments generate and publish enormous quantities of information as a matter of public policy. The challenge is navigating the diverse systems through which this information is accessible.

### U.S. Federal Government Data

**SEC EDGAR**: Every public company's filings — 10-Ks, 10-Qs, 8-Ks, proxy statements, insider trading reports. Contains financial data, officer and director information, risk disclosures, and significant event disclosures. Free and comprehensive.

**FEC Campaign Finance**: Contributions and expenditures for federal political campaigns. Searchable by contributor name, employer, zip code, and amount. Particularly valuable for mapping political relationships.

**PACER**: Public Access to Court Electronic Records — U.S. federal court case documents. Small per-page fee. Contains litigation history, bankruptcy filings, criminal charges, and court orders.

**SAM.gov / USASpending.gov**: Federal contracting and grant data. Who received government contracts, for what, and how much. Invaluable for investigating government vendors and contractors.

**FDA databases**: Drug approvals, adverse event reports, warning letters, recall notices, clinical trial registrations (ClinicalTrials.gov).

**FAA databases**: Aircraft registration, airman certification, flight school approvals. The N-number registry enables ownership research on any registered aircraft.

**FCC licenses**: Radio station licenses, cellular tower permits, amateur radio operator licenses. The ULS database is searchable.

**Patent and trademark databases (USPTO)**: Every patent and trademark registration, with inventor names, assignee organizations, prosecution histories, and cited prior art.

**Export control records**: Bureau of Industry and Security enforcement records, Office of Foreign Assets Control sanctions lists.

### State and Local Government Records

State-level records are particularly valuable because state law governs most business entities, professional licenses, and many criminal matters:

**Secretary of State business filings**: Business entity formation documents, annual reports, officer/director names, registered agent information. Each state maintains its own system with varying depth and search capability.

**Property records**: County assessor and recorder offices maintain ownership history, sale prices, mortgage records, and tax assessment data. Digitization varies by county, but most major metropolitan areas have online access.

**Court records**: State criminal and civil court records vary significantly by state in their online availability. Some states have centralized statewide systems; others require county-by-county access.

**Regulatory licensing**: Each state licensing board maintains records for regulated professions — medical boards, bar associations, contractor licensing boards, insurance licensing, financial services licensing.

**Vital records**: Birth, death, marriage, and divorce records. Accessibility varies significantly by state. Historical vital records are generally more accessible than recent ones.

### International Government Records

International public records present significant variation in quality, accessibility, and language:

**UK Companies House**: One of the most open business registries in the world. Free access to all company filings, director information, officer appointments and resignations, financial accounts, and beneficial ownership data.

**EU-wide systems**: The EU has moved toward harmonized beneficial ownership registers as part of AML Directive implementation, though implementation varies by member state.

**OpenCorporates**: Aggregates company data from 140+ jurisdictions — a single interface for international corporate research, though coverage depth varies by country.

**World Bank, IMF, UN data**: Macroeconomic data, development statistics, governance indicators. Essential for country-level analysis.

---

## 2.7 Technical Data Sources

Technical data sources are often overlooked by investigators who lack engineering backgrounds, but they represent an extraordinarily rich OSINT layer.

### DNS and Domain Intelligence

The Domain Name System is a massive public database. Every registered domain has associated records that reveal:

- **Registrant information**: WHOIS records historically included registrant name, organization, address, and contact information. GDPR has obscured this for privacy-protected registrations, but non-EU registrations, older records, and privacy-service data often remain accessible.
- **DNS records**: A, AAAA, MX, TXT, NS, and CNAME records reveal hosting infrastructure, mail servers, SPF/DKIM configurations, and service provider relationships.
- **Historical WHOIS data**: PassiveDNS databases and WHOIS history tools capture data from before privacy protection was applied.
- **Certificate transparency logs**: Every TLS certificate issued for a domain is logged in public certificate transparency logs. These reveal subdomains, service providers, and certificate history.
- **Domain registration history**: Tools like DomainTools and SecurityTrails track domain ownership changes over time.

### IP Address and Network Intelligence

**BGP routing data**: The Border Gateway Protocol routing tables for the internet are publicly announced and collected. They reveal which organizations own which IP address blocks.

**WHOIS for IP addresses**: Regional Internet Registries (ARIN, RIPE, APNIC, AFRINIC, LACNIC) publish allocation data for IP address ranges.

**Passive DNS**: Commercial passive DNS databases record which domains have resolved to which IP addresses over time. This enables reconstruction of historical hosting relationships.

**Shodan, Censys, FOFA**: Internet-wide scanning databases that index all IP addresses for open services, banners, certificates, and vulnerabilities. These databases provide extraordinary visibility into internet-connected infrastructure.

### Certificate and Cryptographic Data

**SSL/TLS certificates**: Certificate transparency logs capture all publicly issued certificates. CrtSh, Facebook's Certificate Transparency Monitor, and similar tools enable subdomain discovery and infrastructure research.

**PGP key servers**: The public PGP key server network contains millions of public keys with associated email addresses and names.

**Blockchain records**: Cryptocurrency transactions are public by design. Bitcoin, Ethereum, and most other public blockchain transactions can be traced through block explorers. Blockchain analytics companies (Chainalysis, Elliptic, TRM Labs) provide specialized analysis.

### Leaked and Breach Data

A ethically and legally complex but practically significant OSINT source is data that has been leaked from organizations and made publicly available:

**Have I Been Pwned (HIBP)**: Troy Hunt's service allows individuals to check whether their email appears in breach databases. The underlying breach data is not publicly searchable, but the service's notification capability is investigatively useful.

**DeHashed, Snusbase, LeakCheck**: Commercial services providing searchable access to breach data. Legal status varies by jurisdiction. Use requires careful consideration of purpose and applicable law.

**Paste sites**: Pastebin, Ghostbin, and similar text-sharing services are frequently used to publish leaked data. Specialized search tools monitor paste sites for new publications.

The use of breached data is legally ambiguous in many jurisdictions and ethically complex. It is addressed in this book primarily from the perspective of defending against its use and understanding what adversaries can learn about your organization.

---

## 2.8 Geospatial and Satellite Data

Geospatial intelligence — understanding what is happening where — has been transformed by commercial availability of satellite imagery and location data.

### Commercial Satellite Imagery

**Planet Labs**: Daily global coverage at 3-5 meter resolution. Near-real-time monitoring of any location on Earth. Available via subscription.

**Maxar (WorldView)**: Sub-meter resolution imagery of specific locations, with an extensive archive. The premium tier of commercial imagery.

**Sentinel (ESA)**: Free, open satellite imagery from the European Space Agency. Lower resolution (10-60m) but globally available and free.

**Google Earth/Maps**: Historical imagery back to the early satellite era at many locations. The most accessible interface for non-specialist investigators.

**Umbra, Capella**: Synthetic Aperture Radar (SAR) imagery, which sees through cloud cover and can image at night. Increasingly commercially available.

### Location and Mapping Data

**OpenStreetMap**: Community-maintained global mapping database. Often more detailed than commercial maps for specific locations. Machine-readable and API-accessible.

**Bing Maps**: Alternative to Google with independent satellite imagery archive — different dates, different coverage gaps.

**Apple Maps**: Another independent imagery source with its own coverage.

**Historical mapping services**: David Rumsey Map Collection, National Library of Congress, Ordnance Survey archives provide historical cartographic data.

### AIS and ADS-B: Maritime and Aviation Tracking

**AIS (Automatic Identification System)**: International maritime tracking standard. Commercial vessels are required to transmit AIS signals, which are received by coastal stations and satellites. MarineTraffic, VesselFinder, and similar services provide real-time and historical vessel tracking.

**ADS-B (Automatic Dependent Surveillance-Broadcast)**: Aviation equivalent of AIS. Commercial aircraft transponder data collected by ground stations and uploaded to FlightAware, Flightradar24, and similar services. Military and some private aircraft can disable or spoof transponders.

**Strava, AllTrails heat maps**: Aggregated GPS tracking from fitness apps has accidentally revealed sensitive information — including the locations of classified military installations whose personnel used fitness trackers.

---

## 2.9 Data Flow and Lifecycle

Understanding how data moves from generation to public availability helps investigators anticipate where information will appear and plan collection timing.

### The Data Generation Layer

Information originates from human activity: a business transaction creates a receipt and potentially a regulatory record; a social media post creates platform data and potentially news coverage; a property purchase creates county records, title insurance documents, and mortgage records; a legal action creates court records, potentially press coverage, and regulatory action records.

### The Storage and Indexing Layer

Raw data is stored in the systems of the organizations that collected it. From there, it flows to:

- **Primary online access**: The organization's own website or database interface
- **Third-party aggregation**: Data brokers, news archives, court document services
- **Web indexing**: Search engine crawls that create indexed copies
- **Web archiving**: Wayback Machine and other archiving services capture snapshots

### The Retention Layer

Information persists differently across sources:

- **Government records**: Often permanent; rarely deleted without legal process
- **Social media**: Subject to platform deletion policies and user deletion; partially preserved by archives
- **News articles**: Generally persistent but subject to deletion, paywalling, or link rot
- **Commercial databases**: Retained based on provider policy; may lag actual record deletion
- **Web archives**: Snapshots preserved at capture time; can retrieve content deleted from primary source
- **Breach databases**: Persist indefinitely once public, regardless of original source

### Implications for Investigative Timing

Data freshness and retention patterns have direct implications for investigative strategy:

- Real-time social media content should be preserved immediately — before deletion
- Government records may take weeks or months to update after an event
- Commercial databases may lag primary sources by days to years
- Web archives can recover content deleted from primary sources

---

## 2.10 Building a Source Matrix

Professional OSINT practitioners develop source matrices — structured mappings of investigative questions to relevant sources. A source matrix forces systematic thinking about where information lives before the investigation begins.

### Sample Source Matrix for Individual Investigation

| Information Need | Primary Sources | Secondary Sources | Notes |
|---|---|---|---|
| Current address | Property records, voter registration | Commercial databases, social media | Verify against multiple sources |
| Employment | LinkedIn, company filings, professional licensing | News archives, social media | LinkedIn self-reported; verify |
| Criminal history | State court systems, PACER | Commercial databases | Varies significantly by jurisdiction |
| Vehicle ownership | DMV (restricted), public accident reports | Commercial databases (PI-licensed) | Direct DMV access requires authorization |
| Business affiliations | Secretary of State, SEC | OpenCorporates, LinkedIn | UCC filings for financial relationships |
| Social network | Social media analysis | Phone records (restricted) | Cross-platform linkage |
| Financial activity | SEC filings, bankruptcy records, property | Commercial databases | Direct financial records restricted |
| Online presence | Google, social platforms, domain records | Web archives, breach data | Pseudonym linkage is key challenge |

### Sample Source Matrix for Corporate Investigation

| Information Need | Primary Sources | Secondary Sources | Notes |
|---|---|---|---|
| Entity structure | Secretary of State, foreign registries | OpenCorporates, news | Beneficial ownership hard to trace |
| Financial condition | SEC filings, credit agencies | News, industry reports | Public companies only for SEC |
| Litigation history | PACER, state courts | News archives | Material litigation disclosed in SEC filings |
| Key personnel | SEC filings, LinkedIn, company website | News, professional registrations | Cross-reference for accuracy |
| IP portfolio | USPTO, EPO, Google Patents | Academic publications | Reveals R&D focus |
| Contracts and relationships | SAM.gov, news, LinkedIn | SEC disclosed relationships | Government contracts are public |
| Reputation | News archives, court records | Social media, review sites | Qualitative assessment |

---

## Summary

The modern OSINT data landscape encompasses five primary tiers: the indexed surface web, the deep web of accessible-but-unindexed records, social media platforms with their rich but access-restricted data, commercial data aggregators who have assembled comprehensive personal profiles, and technical infrastructure data sources that reveal network and system relationships.

Each source type has distinct characteristics in terms of coverage, accuracy, accessibility, and cost. Effective OSINT practice requires knowing which sources are most likely to yield relevant information for a given investigative question — and understanding the limitations and quality problems of each source before drawing conclusions.

Government and public records provide the most legally unambiguous access to authoritative data. Commercial databases provide rich aggregated profiles at the cost of accuracy. Social media provides direct human intelligence but with access restrictions and verification challenges. Technical sources provide infrastructure and relationship data that is invisible to other methods.

Building source matrices — structured mappings of investigative requirements to data sources — before investigation begins is a foundational discipline that separates systematic investigation from ad hoc searching.

---

## Common Mistakes and Pitfalls

- **Google-first bias**: Treating search engine results as representative of available data when 85%+ of the information space is not indexed
- **Conflating database coverage with data accuracy**: Comprehensive profiles in commercial databases are not necessarily accurate profiles
- **Ignoring temporal dynamics**: Failing to account for data freshness, lag times, and the difference between current and historical records
- **Missing the technical layer**: Overlooking DNS, certificate, and network data sources that reveal infrastructure relationships invisible in content sources
- **Platform monoculture**: Relying on one social media platform when subjects may be more active on others
- **Neglecting international sources**: For subjects with international dimensions, failing to consult non-English and non-U.S. sources dramatically reduces coverage
- **Source conflation**: Losing track of where specific information came from, making verification impossible

---

## Further Reading

- BRB Publications *Public Records Online* — comprehensive guide to U.S. public records databases
- OSINT Framework (osintframework.com) — categorized source directory
- Bellingcat guides to specific source categories — updated practitioner-focused overviews
- Trace Labs OSINT resources — particularly strong on missing persons case source methodology
- Michael Bazzell's OSINT Techniques books — practitioner-focused source coverage with regular updates
