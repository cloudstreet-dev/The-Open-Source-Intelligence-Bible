# Appendix C: Legal Resources

This appendix consolidates the legal frameworks, statutes, and regulatory resources referenced throughout the book. It is a reference guide, not legal advice. Consult qualified legal counsel in your jurisdiction before conducting investigations with significant legal exposure.

---

## C.1 United States Federal Law

### Computer Fraud and Abuse Act (CFAA)
**Statute**: 18 U.S.C. § 1030
**Key provisions**: Prohibits unauthorized access to computers and computer systems. "Unauthorized access" has been interpreted expansively to include ToS violations in some jurisdictions, though the Supreme Court's 2021 decision in *Van Buren v. United States* narrowed the scope.
**Relevance**: Scraping, credential sharing, bypassing access controls
**Resources**:
- EFF's CFAA overview: eff.org/issues/cfaa
- DOJ CFAA manual (public): justice.gov (search CFAA)

### Electronic Communications Privacy Act (ECPA)
**Statute**: 18 U.S.C. §§ 2510-2523 (Wiretap Act), 2701-2711 (Stored Communications Act)
**Key provisions**: Prohibits interception of electronic communications; governs government access to stored communications.
**Relevance**: Recording communications, accessing stored messages, intercepting transmissions
**Resources**:
- EFF's ECPA overview: eff.org/issues/ecpa

### Fair Credit Reporting Act (FCRA)
**Statute**: 15 U.S.C. § 1681 et seq.
**Key provisions**: Regulates consumer credit reporting; imposes obligations on companies that compile and use consumer reports.
**Relevance**: Background investigations used for employment, housing, or credit decisions
**Resources**:
- FTC FCRA guide: ftc.gov/legal-library/browse/statutes/fair-credit-reporting-act
- CFPB FCRA resources: consumerfinance.gov

### Wiretapping and Recording Laws
**Federal baseline**: 18 U.S.C. § 2511 — one-party consent at federal level
**State variation**: California, Connecticut, Florida, Illinois, Maryland, Massachusetts, Michigan, Montana, Nevada, New Hampshire, Oregon, Pennsylvania, Washington require all-party consent
**Relevance**: Recording interviews, monitoring communications
**Resources**: Reporters Committee for Freedom of the Press state-by-state guide: rcfp.org/resources/recording-phone-calls-and-conversations/

### Driver's Privacy Protection Act (DPPA)
**Statute**: 18 U.S.C. § 2721
**Key provisions**: Restricts access to DMV records; permissible uses are enumerated
**Relevance**: Vehicle registration research, PI work involving vehicles

---

## C.2 United States Privacy and Data Law

### California Consumer Privacy Act / CPRA
**Statute**: Cal. Civil Code § 1798.100 et seq.
**Key provisions**: Grants California residents rights to know about, delete, and opt out of sale of personal information.
**Relevance**: Collecting and processing personal data about California residents
**Resources**: California Privacy Protection Agency: cppa.ca.gov

### Illinois Biometric Information Privacy Act (BIPA)
**Statute**: 740 ILCS 14/
**Key provisions**: Prohibits collecting biometric identifiers (fingerprints, retina scans, facial geometry) without written consent.
**Relevance**: Facial recognition, biometric data collection
**Resources**: Illinois legislature website

### Video Privacy Protection Act (VPPA)
**Statute**: 18 U.S.C. § 2710
**Key provisions**: Prohibits disclosure of personally identifiable information about video rental/subscription services.
**Relevance**: Limited direct OSINT application; awareness for data broker research

### Children's Online Privacy Protection Act (COPPA)
**Statute**: 15 U.S.C. § 6501
**Key provisions**: Restricts collection of personal information from children under 13.
**Relevance**: Investigations involving minors; operating platforms accessible to children

---

## C.3 International Privacy Law

### European Union — GDPR
**Regulation**: EU 2016/679
**Key provisions**: Comprehensive data protection regulation; applies to processing of EU residents' personal data regardless of where processor is located.
**Key principles**: Lawfulness, fairness, transparency; purpose limitation; data minimization; accuracy; storage limitation; integrity and confidentiality; accountability.
**Lawful bases for processing**: Consent, contract, legal obligation, vital interests, public task, legitimate interests.
**Journalism exception**: Article 85 allows member states to provide exemptions for journalism and academic research.
**Resources**:
- Official text: gdpr-info.eu
- EDPB guidelines: edpb.europa.eu
- UK ICO guidance: ico.org.uk/for-organisations/guide-to-data-protection/guide-to-the-general-data-protection-regulation-gdpr/

### United Kingdom — UK GDPR and DPA 2018
**Post-Brexit**: UK GDPR largely mirrors EU GDPR with UK-specific adjustments.
**Special category**: Journalism processing exemptions in Schedule 2, Part 5 of DPA 2018.
**Resources**: ICO (Information Commissioner's Office): ico.org.uk

### Brazil — LGPD
**Law**: Lei Geral de Proteção de Dados (Law 13,709/2018)
**Applies to**: Processing of personal data in Brazil, or data of individuals located in Brazil.
**Supervisory authority**: ANPD (Autoridade Nacional de Proteção de Dados)

### Canada — PIPEDA and Provincial Laws
**Federal**: Personal Information Protection and Electronic Documents Act (PIPEDA)
**Quebec**: Law 25 (Loi 25) — significantly enhanced privacy requirements effective 2023
**Resources**: Office of the Privacy Commissioner of Canada: priv.gc.ca

---

## C.4 Sector-Specific Regulations

### Financial Services — Bank Secrecy Act (BSA)
**Statute**: 31 U.S.C. §§ 5311-5336
**Key provisions**: Anti-money laundering reporting requirements; suspicious activity reports (SARs); currency transaction reports (CTRs).
**Relevance**: Financial crime investigation context; AML due diligence requirements
**Resources**: FinCEN: fincen.gov

### Financial Services — Corporate Transparency Act (CTA)
**Statute**: 31 U.S.C. § 5336
**Key provisions**: Requires most US companies to report beneficial ownership to FinCEN. Database access is restricted (not public).
**Effective**: January 1, 2024 (existing companies had until January 1, 2025)
**Resources**: FinCEN BOI: fincen.gov/boi

### Healthcare — HIPAA
**Statute**: 45 CFR Parts 160 and 164
**Key provisions**: Protects health information; limits disclosure without patient authorization.
**Relevance**: Investigations involving medical records or healthcare providers

### Securities — SEC Disclosure Requirements
**Key forms**:
- 10-K: Annual report
- 10-Q: Quarterly report
- 8-K: Material event disclosure (includes cybersecurity incidents since December 2023)
- DEF14A: Proxy statement (executive compensation, governance)
- Form 4: Insider transactions
- Schedule 13D/13G: Large shareholder disclosures
**Resources**: SEC EDGAR: sec.gov/cgi-bin/browse-edgar

---

## C.5 Platform Terms of Service — Key Provisions

### General ToS Principles for OSINT

Most major platforms prohibit:
- Automated scraping without permission
- Creating accounts to circumvent restrictions
- Collecting data for surveillance or harassment
- Using platform data for training AI models without authorization
- Selling or commercially exploiting scraped data

CFAA exposure from ToS violations: Post-*Van Buren*, the Supreme Court narrowed CFAA scope to situations where access exceeds authorization defined by access controls, not merely ToS terms. However, some courts still apply ToS violations as grounds for CFAA claims.

### Twitter/X API Terms
Relevant policies: Developer Policy, Display Requirements, Automation Rules
**Note**: Twitter significantly changed API access and pricing in 2023; verify current terms before building integrations.

### LinkedIn Terms
LinkedIn has actively enforced against scraping with CFAA claims (*hiQ Labs v. LinkedIn*). The Ninth Circuit ruled that scraping publicly available LinkedIn profiles is not CFAA-violating (no authorization required for public data), but this remains litigated territory.

### Meta/Facebook
Graph API has significantly restricted data access since the Cambridge Analytica incident. Accessing non-public data through any means other than official API violates ToS and potentially CFAA.

---

## C.6 PI Licensing by State (US Summary)

*This is a brief reference. Requirements change; verify current requirements before practice.*

| State | License Required | Key Requirements |
|---|---|---|
| California | Yes (BSIS) | 6,000 hours experience or degree + 3,000 hours |
| Texas | Yes (DPS) | 3 years experience or degree + 1 year |
| New York | Yes (DCJS) | 3 years experience; background check |
| Florida | Yes (FDLE) | 2 years experience; licensure exam |
| Illinois | Yes | 3 years law enforcement or 5 years PI experience |
| Washington | Yes | No license required for PI work (one of few states) |
| Colorado | No state license | Local jurisdiction requirements vary |
| Idaho | No | No statewide PI license requirement |

**Resources**: National Association of Legal Investigators (NALI): nalionline.org

---

## C.7 Journalism Shield Laws

Shield laws protect journalists from being compelled to reveal confidential sources in legal proceedings. Coverage varies significantly by state; federal shield law protections are limited.

**States with shield laws**: 40+ states have some form of shield law; strength varies.
**Federal**: No comprehensive federal shield law; limited protections under common law.
**Key cases**: *Branzburg v. Hayes* (1972, Supreme Court) — limited First Amendment protection for source confidentiality.

**Resources**:
- Reporters Committee for Freedom of the Press: rcfp.org/resources/shields-sources-from-coast-to-coast/
- Student Press Law Center: splc.org

---

## C.8 Finding Legal Counsel for OSINT Matters

**Media law and First Amendment attorneys**: Reporters Committee for Freedom of the Press legal defense hotline (for journalists): 800-336-4243

**Privacy law specialists**: IAPP (International Association of Privacy Professionals) member directory: iapp.org

**Cybercrime and CFAA**: EFF legal clinic referrals; Federal Public Defender offices for criminal matters

**PI licensing**: State PI licensing boards typically have resources for licensure questions

**Disclaimer**: This appendix is a reference starting point. Laws change; this book may not reflect the most current state of law at time of reading. Always consult qualified legal counsel for specific situations.
