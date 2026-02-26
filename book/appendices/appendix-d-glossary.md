# Appendix D: Glossary

This glossary defines key terms used throughout the book. Terms are organized alphabetically.

---

**Active reconnaissance**: Investigation techniques that involve direct interaction with target systems, which may leave detectable traces. Contrast with *passive reconnaissance*.

**ADS-B (Automatic Dependent Surveillance-Broadcast)**: A surveillance technology that aircraft use to broadcast their position, altitude, and identification to ground stations and other aircraft. ADS-B data is widely available through aggregators for flight tracking.

**Adverse media**: Negative news coverage, litigation records, or other derogatory public information about a subject — a standard category in due diligence screening.

**AIS (Automatic Identification System)**: A tracking system for ships that broadcasts vessel identity, position, course, and speed via VHF radio. Widely used in maritime intelligence.

**Analysis of Competing Hypotheses (ACH)**: A structured analytic technique that requires explicitly considering multiple hypotheses simultaneously, evaluating evidence diagnostically rather than seeking to confirm a pre-selected explanation.

**APT (Advanced Persistent Threat)**: A sophisticated, typically state-sponsored threat actor that conducts long-term targeted intrusion campaigns. OSINT is used both to research APT activity and to profile their infrastructure.

**Attack surface**: All the points of an organization's digital infrastructure that could be targeted by an adversary — domains, subdomains, IP addresses, exposed services, and employee accounts.

**Attribution**: Determining the responsible party for an action, statement, or content. In cybersecurity, attributing attacks to specific threat actors. In journalism, attributing statements to specific people.

**Blockchain**: A distributed, append-only ledger. In OSINT contexts, the Bitcoin and Ethereum blockchains are publicly readable and allow tracing of cryptocurrency transactions.

**Bug bounty program**: A program in which organizations offer monetary rewards to security researchers who responsibly discover and disclose vulnerabilities.

**Canary token**: A honeytoken — a file, URL, or other resource that generates an alert when accessed. Used to detect unauthorized access or investigation of sensitive materials.

**Certificate Transparency (CT)**: A public log of issued TLS/SSL certificates, enabling verification of certificate issuance and discovery of subdomains through historical certificate records.

**Chain of custody**: Documentation of who has collected, controlled, transferred, and accessed evidence from the moment of collection through its presentation in legal proceedings.

**CFAA (Computer Fraud and Abuse Act)**: US federal statute that prohibits unauthorized access to computers. Central to the legal framework governing OSINT collection methods.

**CISA (Cybersecurity and Infrastructure Security Agency)**: US federal agency responsible for protecting critical infrastructure from physical and cyber threats.

**Confirmation bias**: The tendency to search for, interpret, and recall information in a way that confirms pre-existing beliefs. One of the most consequential cognitive biases in OSINT investigation.

**Coordinated inauthentic behavior (CIB)**: Networks of accounts that act in concert to artificially amplify content or narratives, typically in violation of platform policies. Detection is a major focus of disinformation research.

**Cryptocurrency**: Digital currency that uses cryptography for security and typically operates on a blockchain. Bitcoin, Ethereum, and Monero are common targets of financial crime investigation.

**Dark web**: Portions of the internet accessible only through anonymizing networks such as Tor (.onion sites) or I2P. Often conflated with deep web; technically distinct.

**Data aggregation problem**: The principle that combining individually innocuous data points can create a picture that constitutes a serious privacy violation, even if each individual piece of data was lawfully collected.

**Deep learning**: A subset of machine learning that uses neural networks with many layers to learn representations of data. Underlies modern NLP, computer vision, and generative AI capabilities.

**Deep web**: Portions of the internet not indexed by standard search engines, including password-protected content, dynamic database-driven content, and private networks. Much larger than the indexed web.

**Deepfake**: Synthetic media — typically video or audio — in which a person's likeness, voice, or expression has been replaced or manipulated using deep learning techniques.

**DNS (Domain Name System)**: The internet's phonebook, translating human-readable domain names to IP addresses. DNS records (A, MX, NS, TXT, CNAME, etc.) are valuable OSINT data sources.

**DMARC (Domain-based Message Authentication, Reporting, and Conformance)**: An email authentication protocol that enables domain owners to control how unauthenticated email is handled. Misconfigured DMARC exposes domains to spoofing.

**EDGAR (Electronic Data Gathering, Analysis, and Retrieval)**: The SEC's public database of mandatory filings from public companies.

**Entity disambiguation**: The process of determining whether references to an entity (e.g., a person's name) in different contexts refer to the same individual or different individuals.

**EXIF (Exchangeable Image File Format)**: Metadata embedded in image files, which can include camera make/model, GPS coordinates, timestamps, and software information.

**FCRA (Fair Credit Reporting Act)**: US federal law governing consumer reporting agencies and the use of consumer reports for employment, housing, and credit decisions.

**FinCEN (Financial Crimes Enforcement Network)**: US Treasury bureau responsible for collecting and analyzing financial transaction data to combat money laundering, terrorism financing, and other financial crimes.

**GDPR (General Data Protection Regulation)**: EU data protection regulation applicable to processing of personal data of EU residents.

**Geolocation**: Determining the physical location where an image or video was captured, typically through analysis of visual, astronomical, and contextual clues.

**GEOINT (Geospatial Intelligence)**: Intelligence derived from the analysis of imagery and geospatial data. Includes satellite imagery, mapping, and location analytics.

**Graph database**: A database that stores data as nodes and edges, optimized for querying relationships. Neo4j is the most common graph database in OSINT applications.

**Hallucination**: In AI contexts, the generation of plausible-sounding but factually incorrect information by a language model. A critical limitation of LLMs in OSINT applications.

**HUMINT (Human Intelligence)**: Intelligence gathered through direct human-to-human interaction, as distinct from technical collection methods.

**IOC (Indicator of Compromise)**: Artifacts that suggest a computer system has been breached — IP addresses, domain names, file hashes, and URLs associated with known malicious activity.

**IMINT (Imagery Intelligence)**: Intelligence derived from imagery sources, including satellite, aerial, and ground-level photography.

**Intelligence cycle**: The structured process of intelligence production: planning, collection, processing, analysis, and dissemination.

**ITAR (International Traffic in Arms Regulations)**: US export control regulations governing defense articles and services. Relevant when OSINT research touches on weapons systems or controlled technologies.

**Kappa architecture**: A data pipeline architecture that processes all data as a stream, eliminating the batch processing layer of the lambda architecture.

**Lambda architecture**: A data pipeline architecture combining batch and stream processing layers, with results merged in a serving layer.

**LLM (Large Language Model)**: A neural network trained on large text corpora that can generate, classify, translate, and analyze text. Examples include Claude, GPT-4, and Gemini.

**Maltego**: A commercial link analysis and data aggregation platform widely used in OSINT and cybersecurity investigations.

**MISP (Malware Information Sharing Platform)**: An open-source threat intelligence platform for sharing, storing, and correlating indicators of compromise.

**MITRE ATT&CK**: A knowledge base of adversary tactics and techniques based on real-world observations, widely used in threat intelligence and red team assessments.

**Money mule**: An individual who transfers funds on behalf of others as part of a money laundering scheme, often unwittingly.

**Named Entity Recognition (NER)**: An NLP task that identifies and classifies proper nouns (people, organizations, locations, etc.) in text.

**OFAC (Office of Foreign Assets Control)**: US Treasury office that administers and enforces economic sanctions. The OFAC SDN (Specially Designated Nationals) list is a key sanctions screening resource.

**Open source**: In OSINT context, "open source" refers to publicly available information — not open-source software (though many OSINT tools are open-source software).

**OPSEC (Operations Security)**: Protecting sensitive information about investigation activities, methodology, and identity from adversaries.

**Passive DNS**: Historical records of which IP addresses a domain has resolved to over time. Valuable for tracking infrastructure changes and attributing malicious activity.

**Passive reconnaissance**: Investigation techniques that collect information without directly interacting with target systems. Contrast with *active reconnaissance*.

**PACER (Public Access to Court Electronic Records)**: The federal judiciary's online system for accessing US federal court records.

**PEP (Politically Exposed Person)**: An individual who holds or has held a prominent public function, and their close associates, subject to enhanced due diligence under AML regulations.

**Pivot**: In OSINT, using one identified data point (e.g., a phone number) to discover related data (e.g., other accounts using the same number). The fundamental investigative technique.

**Prompt injection**: An attack against LLM-powered systems in which malicious instructions embedded in input content cause the model to deviate from its intended behavior.

**RAG (Retrieval-Augmented Generation)**: An LLM architecture that retrieves relevant documents from a database and uses them as context when generating responses, improving accuracy and reducing hallucination.

**Responsible disclosure**: The practice of reporting discovered vulnerabilities to affected organizations and allowing remediation time before public disclosure.

**RLHF (Reinforcement Learning from Human Feedback)**: A training technique that fine-tunes language models based on human preference judgments, improving alignment with human values and intended use.

**SAR (Suspicious Activity Report)**: A report filed with FinCEN by financial institutions when they detect potentially suspicious financial activity that may indicate money laundering or other financial crimes.

**Sanctions**: Government-imposed restrictions on financial transactions and business dealings with specific individuals, organizations, or countries.

**SIGINT (Signals Intelligence)**: Intelligence gathered from intercepted signals, including communications and electronic emissions.

**SimHash**: A locality-sensitive hash algorithm that allows near-duplicate detection; documents with similar content produce similar hash values.

**SOCMINT (Social Media Intelligence)**: Intelligence gathered from social media platforms.

**STIX (Structured Threat Information eXpression)**: A standardized language for representing and sharing threat intelligence.

**Subdomain enumeration**: The process of discovering subdomains of a target domain using certificate transparency logs, DNS bruteforcing, or passive sources.

**Threat intelligence**: Information about current or potential attacks, including indicators of compromise, threat actor profiles, and attack methodologies.

**Tor (The Onion Router)**: An anonymizing network that routes internet traffic through multiple relay nodes to obscure origin. Used for anonymization in sensitive investigations and to access .onion dark web sites.

**TTP (Tactics, Techniques, and Procedures)**: The behavioral profile of a threat actor, describing how they operate. Defined in frameworks like MITRE ATT&CK.

**UCC (Uniform Commercial Code)**: A set of laws governing commercial transactions in the US. UCC filings (secured transaction records) are valuable in asset investigation.

**Vector database**: A database optimized for storing and searching high-dimensional vector embeddings, enabling semantic similarity search.

**WHOIS**: A query-response protocol for obtaining registration information about internet resources including domain names and IP address blocks. WHOIS privacy services increasingly obscure registrant data.

**Zero-day**: A previously unknown software vulnerability for which no patch exists. In OSINT, relevant to threat intelligence about unpatched vulnerabilities being actively exploited.
