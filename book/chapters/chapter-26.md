# Chapter 26: Operational Security and Risk Management

## Learning Objectives

By the end of this chapter, you will be able to:
- Develop a personal OPSEC plan appropriate for your investigation context
- Assess and manage legal, physical, and reputational risks in investigative work
- Implement data handling and storage practices that protect sources and findings
- Establish secure communication channels for sensitive investigation coordination
- Design risk-aware workflows that reduce exposure without compromising effectiveness
- Understand when to escalate concerns and how to seek legal counsel appropriately

---

## 26.1 Risk in OSINT Practice

Every investigation carries risk. The nature and magnitude of that risk depends on the subject, the methodology, the jurisdiction, and the use to which findings will be put. An OPSEC-unaware investigator is one incident away from:

- Exposing sources who trusted them with information
- Creating legal liability through unauthorized data access or privacy violations
- Alerting the investigation subject, allowing evidence destruction or flight
- Compromising ongoing law enforcement investigations through premature disclosure
- Personal physical risk in investigations involving dangerous actors

Risk management in OSINT is not about avoiding all risk — it is about making deliberate, informed decisions about which risks to accept and which to mitigate.

---

## 26.2 Threat Modeling for Investigators

A threat model is a structured analysis of what you're protecting, from whom, and how effectively. The process:

```python
from dataclasses import dataclass, field
from typing import List, Dict
from enum import Enum

class ThreatLevel(Enum):
    NEGLIGIBLE = 1
    LOW = 2
    MEDIUM = 3
    HIGH = 4
    CRITICAL = 5

class MitigationStatus(Enum):
    NOT_STARTED = "Not started"
    IN_PROGRESS = "In progress"
    IMPLEMENTED = "Implemented"
    NOT_APPLICABLE = "N/A"

@dataclass
class Threat:
    """Individual threat in the threat model"""
    threat_actor: str
    attack_vector: str
    target_asset: str
    likelihood: ThreatLevel
    impact: ThreatLevel
    existing_controls: List[str] = field(default_factory=list)
    recommended_controls: List[str] = field(default_factory=list)
    mitigation_status: MitigationStatus = MitigationStatus.NOT_STARTED

    @property
    def risk_score(self) -> int:
        return self.likelihood.value * self.impact.value

    @property
    def risk_level(self) -> str:
        score = self.risk_score
        if score <= 4:
            return "LOW"
        elif score <= 9:
            return "MEDIUM"
        elif score <= 16:
            return "HIGH"
        else:
            return "CRITICAL"


@dataclass
class InvestigatorThreatModel:
    """
    Comprehensive threat model for an OSINT investigator
    """
    investigator_type: str  # e.g., "journalist", "corporate security", "PI", "researcher"
    investigation_subject_risk: str  # e.g., "organized crime", "corporate", "individual"
    operating_jurisdiction: str
    findings_use: str  # e.g., "publication", "litigation", "internal", "law enforcement"

    assets_to_protect: List[str] = field(default_factory=list)
    threats: List[Threat] = field(default_factory=list)

    def generate_standard_threats(self) -> None:
        """Generate standard threat inventory based on investigator profile"""
        standard_threats = [
            Threat(
                threat_actor="Investigation subject",
                attack_vector="Legal action (defamation, harassment claim)",
                target_asset="Investigation findings and investigator identity",
                likelihood=ThreatLevel.MEDIUM,
                impact=ThreatLevel.HIGH,
                existing_controls=[],
                recommended_controls=[
                    "Maintain meticulous documentation of sources",
                    "Obtain legal review before publication/disclosure",
                    "Avoid publication until evidence meets threshold",
                    "Ensure all investigative methods were lawful"
                ]
            ),
            Threat(
                threat_actor="Investigation subject",
                attack_vector="Counter-surveillance / OPSEC detection",
                target_asset="Investigation identity and methodology",
                likelihood=ThreatLevel.MEDIUM,
                impact=ThreatLevel.HIGH,
                existing_controls=[],
                recommended_controls=[
                    "Use dedicated investigation accounts",
                    "Access sources from isolated browsing environment",
                    "Do not access subject's social media from personal accounts",
                    "Compartmentalize investigation activities"
                ]
            ),
            Threat(
                threat_actor="Hostile state actor (if applicable)",
                attack_vector="Technical intrusion against investigator devices",
                target_asset="Investigation data, sources, communications",
                likelihood=ThreatLevel.LOW,
                impact=ThreatLevel.CRITICAL,
                existing_controls=[],
                recommended_controls=[
                    "Use end-to-end encrypted communications (Signal)",
                    "Full disk encryption on all devices",
                    "Enable lockdown mode on iOS for high-risk periods",
                    "Avoid SMS for sensitive communication",
                    "Consider Tails OS for highest-sensitivity work"
                ]
            ),
            Threat(
                threat_actor="Platform moderation",
                attack_vector="Account suspension for research accounts",
                target_asset="Investigation access to social media data",
                likelihood=ThreatLevel.MEDIUM,
                impact=ThreatLevel.MEDIUM,
                existing_controls=[],
                recommended_controls=[
                    "Maintain research accounts in good standing",
                    "Do not violate platform ToS in investigation activities",
                    "Archive all platform data immediately upon collection",
                    "Use platform APIs where available"
                ]
            ),
            Threat(
                threat_actor="Data breach",
                attack_vector="Investigation data stolen from investigator storage",
                target_asset="Source identities, investigation data",
                likelihood=ThreatLevel.LOW,
                impact=ThreatLevel.CRITICAL,
                existing_controls=[],
                recommended_controls=[
                    "Encrypt all investigation data at rest",
                    "Use password manager with strong unique passwords",
                    "Enable MFA on all accounts storing investigation data",
                    "Limit cloud storage of sensitive materials"
                ]
            ),
            Threat(
                threat_actor="Disgruntled source",
                attack_vector="Source recants, disputes, or reveals investigation",
                target_asset="Investigation credibility",
                likelihood=ThreatLevel.LOW,
                impact=ThreatLevel.HIGH,
                existing_controls=[],
                recommended_controls=[
                    "Document source communications contemporaneously",
                    "Verify source claims against independent evidence",
                    "Do not publish findings based solely on single-source claims",
                    "Understand source motivation and potential bias"
                ]
            )
        ]

        # Filter by relevance to investigation type
        if self.investigation_subject_risk == "organized crime":
            # Add physical safety threat
            standard_threats.append(Threat(
                threat_actor="Investigation subject (organized crime)",
                attack_vector="Physical intimidation or harm",
                target_asset="Investigator physical safety",
                likelihood=ThreatLevel.MEDIUM,
                impact=ThreatLevel.CRITICAL,
                existing_controls=[],
                recommended_controls=[
                    "Do not conduct surveillance of organized crime subjects alone",
                    "Vary routines if under physical surveillance",
                    "Maintain regular check-in protocols with trusted contacts",
                    "Notify law enforcement if directly threatened",
                    "Consider relocating during high-risk publication period"
                ]
            ))

        self.threats = standard_threats

    def generate_threat_model_report(self) -> str:
        """Generate structured threat model report"""
        if not self.threats:
            self.generate_standard_threats()

        report = [
            f"# Investigator Threat Model",
            f"**Investigator Type**: {self.investigator_type}",
            f"**Subject Risk Level**: {self.investigation_subject_risk}",
            f"**Jurisdiction**: {self.operating_jurisdiction}",
            f"**Findings Use**: {self.findings_use}",
            "",
            "## Assets to Protect",
            ""
        ]

        default_assets = [
            "Source identities and communications",
            "Investigation methodology and evidence",
            "Investigation subject information (pre-disclosure)",
            "Investigator personal identity and safety",
            "Organization/employer reputation"
        ]

        for asset in (self.assets_to_protect or default_assets):
            report.append(f"- {asset}")

        report.extend(["", "## Threat Assessment", ""])

        # Sort by risk score descending
        sorted_threats = sorted(self.threats, key=lambda t: t.risk_score, reverse=True)

        for i, threat in enumerate(sorted_threats, 1):
            report.extend([
                f"### Threat {i}: {threat.threat_actor} — {threat.attack_vector}",
                f"**Risk Level**: {threat.risk_level} (Likelihood: {threat.likelihood.name}, Impact: {threat.impact.name})",
                f"**Asset at Risk**: {threat.target_asset}",
                "",
                "**Recommended Controls**:"
            ])
            for control in threat.recommended_controls:
                report.append(f"- {control}")
            report.append("")

        return '\n'.join(report)
```

---

## 26.3 Secure Data Handling

Investigation data is among the most sensitive information an organization handles. Mishandled investigation data can expose sources, alert subjects, create legal liability, and undermine findings in litigation.

### Data Classification for Investigations

```python
from enum import Enum
from dataclasses import dataclass, field
from typing import List, Optional
from datetime import datetime
import os
import json

class DataClassification(Enum):
    PUBLIC = "Public"           # Can be shared openly
    INTERNAL = "Internal"       # Share within organization only
    SENSITIVE = "Sensitive"     # Need-to-know access; encrypt at rest
    RESTRICTED = "Restricted"   # Strict access control; encrypt in transit and at rest
    SOURCE = "Source"           # Source communications — maximum protection

@dataclass
class InvestigationDataItem:
    """An item of investigation data with classification metadata"""
    content: str
    classification: DataClassification
    source_description: str
    collection_date: str = field(default_factory=lambda: datetime.now().isoformat())
    contains_pii: bool = False
    access_log: List[Dict] = field(default_factory=list)
    destruction_date: Optional[str] = None

    def log_access(self, accessed_by: str, purpose: str) -> None:
        """Log access for chain of custody"""
        self.access_log.append({
            'accessed_by': accessed_by,
            'purpose': purpose,
            'timestamp': datetime.now().isoformat()
        })

    def to_sanitized_dict(self) -> Dict:
        """Return data without PII-containing fields"""
        return {
            'classification': self.classification.value,
            'source_description': self.source_description,
            'collection_date': self.collection_date,
            'contains_pii': self.contains_pii,
            'access_log_count': len(self.access_log)
        }


class SecureInvestigationVault:
    """
    Secure storage framework for investigation data
    In production: implement with proper encryption library (e.g., cryptography.fernet)
    """

    def __init__(self, vault_path: str, encryption_key: bytes = None):
        self.vault_path = vault_path
        self.encryption_key = encryption_key
        self.items: Dict[str, InvestigationDataItem] = {}

        os.makedirs(vault_path, exist_ok=True)

        # Access log
        self.access_log_path = os.path.join(vault_path, 'access_log.jsonl')

    def store(self, item_id: str, item: InvestigationDataItem) -> None:
        """Store an investigation data item"""
        self.items[item_id] = item

        # For RESTRICTED and SOURCE data, encrypt before writing
        if item.classification in (DataClassification.RESTRICTED, DataClassification.SOURCE):
            self._store_encrypted(item_id, item)
        else:
            self._store_plain(item_id, item)

    def retrieve(self, item_id: str, accessor: str, purpose: str) -> Optional[InvestigationDataItem]:
        """Retrieve an item and log access"""
        item = self.items.get(item_id)
        if item:
            item.log_access(accessor, purpose)
            self._log_access(item_id, accessor, purpose)
        return item

    def _store_encrypted(self, item_id: str, item: InvestigationDataItem) -> None:
        """Store item with encryption"""
        if not self.encryption_key:
            raise ValueError("Encryption key required for RESTRICTED/SOURCE data")

        try:
            from cryptography.fernet import Fernet
            f = Fernet(self.encryption_key)
            encrypted = f.encrypt(json.dumps({
                'content': item.content,
                'classification': item.classification.value,
                'source_description': item.source_description,
                'collection_date': item.collection_date,
                'contains_pii': item.contains_pii
            }).encode())

            path = os.path.join(self.vault_path, f"{item_id}.enc")
            with open(path, 'wb') as f_out:
                f_out.write(encrypted)

        except ImportError:
            # cryptography not installed — write with WARNING
            path = os.path.join(self.vault_path, f"{item_id}.UNENCRYPTED.json")
            with open(path, 'w') as f_out:
                json.dump({'WARNING': 'NOT ENCRYPTED', 'content': item.content}, f_out)

    def _store_plain(self, item_id: str, item: InvestigationDataItem) -> None:
        """Store non-sensitive item"""
        path = os.path.join(self.vault_path, f"{item_id}.json")
        with open(path, 'w') as f_out:
            json.dump(item.to_sanitized_dict(), f_out, indent=2)

    def _log_access(self, item_id: str, accessor: str, purpose: str) -> None:
        """Append to access log"""
        with open(self.access_log_path, 'a') as f:
            f.write(json.dumps({
                'timestamp': datetime.now().isoformat(),
                'item_id': item_id,
                'accessor': accessor,
                'purpose': purpose
            }) + '\n')

    def generate_data_inventory(self) -> str:
        """Generate data inventory for compliance and disclosure purposes"""
        inventory = [
            "# Investigation Data Inventory",
            f"**Generated**: {datetime.now().isoformat()[:19]}",
            f"**Vault**: {self.vault_path}",
            "",
            "| Item ID | Classification | PII | Collection Date | Destruction Date |",
            "|---|---|---|---|---|"
        ]

        for item_id, item in self.items.items():
            inventory.append(
                f"| {item_id} | {item.classification.value} | "
                f"{'Yes' if item.contains_pii else 'No'} | "
                f"{item.collection_date[:10]} | "
                f"{item.destruction_date or 'Not set'} |"
            )

        return '\n'.join(inventory)
```

---

## 26.4 Secure Communications

Investigators coordinating on sensitive matters need communications infrastructure that matches the threat model.

### Communication Security Tiers

| Threat Level | Appropriate Tools | Avoid |
|---|---|---|
| Low (public interest research) | Standard email, Slack | None significant |
| Medium (corporate investigation, PI work) | Signal for sensitive items, encrypted email (ProtonMail) | Standard SMS, unencrypted email for sensitive content |
| High (investigative journalism, organized crime) | Signal with disappearing messages, face-to-face for most sensitive | All cloud platforms, standard telephony |
| Critical (state actor threat) | Tails OS, air-gapped devices, SecureDrop, in-person | All networked communications for most sensitive |

### Signal Protocol Implementation Considerations

For teams that need to implement secure communication infrastructure rather than relying on consumer apps:

```python
"""
Secure communication checklist for investigation teams
This is a practices guide, not executable code
"""

SECURE_COMMUNICATION_PRACTICES = {
    "signal_usage": [
        "Enable disappearing messages on sensitive conversations (24h or less)",
        "Use Note to Self only — never copy Signal conversations to screenshots",
        "Verify safety numbers with key contacts in person or via a separate channel",
        "Enable registration lock (PIN) to prevent SIM swapping attacks",
        "Disable link previews for sensitive conversations",
        "Use Signal on a dedicated device if subject is sophisticated adversary"
    ],

    "email_security": [
        "Use ProtonMail or Tutanota for sensitive external communications",
        "Configure DKIM, SPF, and DMARC for organizational email",
        "Never send source-identifying information via standard email",
        "Use PGP encryption for sensitive email content to external parties",
        "Enable 2FA on all email accounts — authenticator app, not SMS"
    ],

    "document_sharing": [
        "Use end-to-end encrypted file sharing (Signal, Keybase) not Google Drive for sensitive docs",
        "Implement need-to-know access on all investigation documents",
        "Watermark investigation documents with recipient ID to detect leaks",
        "Use OnionShare for sending files to anonymous sources",
        "Document all document-sharing actions with timestamp and recipient"
    ],

    "source_protection": [
        "Offer SecureDrop contact to sensitive sources before they contact you",
        "Never store source identity information in cloud services",
        "Use anonymizing language in notes — describe source by role, not name",
        "Destroy source-identifying materials according to documented schedule",
        "Consult legal counsel before promising confidentiality you cannot guarantee"
    ]
}
```

---

## 26.5 Legal Risk Management

OSINT investigations carry specific legal risks that vary by jurisdiction, subject type, and investigation methodology.

### Key Legal Risks

**Computer Fraud and Abuse Act (CFAA) / Unauthorized Access**: Accessing systems without authorization, or exceeding authorized access, is a federal crime in the US. "Authorization" in OSINT contexts means:
- Data that is publicly available without login: generally lawful to access
- Data behind a login that you authenticated to see: lawful if ToS-compliant
- Data behind technical controls bypassed by the investigator: potentially CFAA-violating

**Electronic Communications Privacy Act (ECPA)**: Intercepting electronic communications (without consent) violates federal law. Scraping public posts is not interception; monitoring private messages is.

**State privacy laws**: California's CPRA, Illinois's BIPA (biometric data), and similar laws impose data handling requirements on personal data. Investigators who collect PII may have obligations under these frameworks.

**Defamation**: Publishing false statements of fact about identifiable individuals. The defamation risk in OSINT investigations is publishing findings that are incomplete, misattributed, or taken out of context.

```python
def legal_risk_assessment(
    investigation_methodology: List[str],
    subject_type: str,  # "individual", "corporation", "public_official"
    jurisdiction: str,
    intended_use: str   # "publication", "litigation", "internal", "law_enforcement"
) -> Dict:
    """
    Framework for assessing legal risk of investigation activities
    NOTE: This is not legal advice. Consult a qualified attorney.
    """
    risks = []
    mitigations = []

    # CFAA analysis
    computer_activities = [a for a in investigation_methodology
                          if any(kw in a.lower() for kw in ['login', 'access', 'scrape', 'crawl', 'api'])]

    if computer_activities:
        risks.append({
            'law': 'CFAA / Computer Fraud and Abuse Act',
            'applicability': 'HIGH' if 'login bypass' in str(computer_activities).lower() else 'MEDIUM',
            'relevant_activities': computer_activities,
            'analysis': 'Verify all data access is authorized — only access data behind login using legitimate credentials you have right to use'
        })
        mitigations.append("Document authorization basis for every data access method")
        mitigations.append("Use only official APIs where available")

    # Privacy law analysis
    pii_activities = [a for a in investigation_methodology
                     if any(kw in a.lower() for kw in ['personal', 'identity', 'address', 'phone', 'email'])]

    if pii_activities and jurisdiction in ['CA', 'Illinois', 'EU', 'UK']:
        risks.append({
            'law': f'Privacy law ({jurisdiction})',
            'applicability': 'MEDIUM',
            'relevant_activities': pii_activities,
            'analysis': f'Collection of personal data in {jurisdiction} may trigger data protection obligations'
        })
        mitigations.append("Collect only PII necessary for investigation purpose (data minimization)")
        mitigations.append("Establish legal basis for PII processing if subject to GDPR")

    # Publication risk
    if intended_use == 'publication':
        risks.append({
            'law': 'Defamation / libel',
            'applicability': 'HIGH' if subject_type == 'individual' else 'MEDIUM',
            'analysis': 'Publication of investigation findings creates defamation exposure if facts are wrong, misleading, or lack context',
            'relevant_activities': ['All publication activities']
        })
        mitigations.append("Verify all material facts across multiple independent sources")
        mitigations.append("Give subject opportunity to respond before publication (legal requirement in some jurisdictions)")
        mitigations.append("Obtain legal review of draft publication")

    # Litigation support risk
    if intended_use == 'litigation':
        risks.append({
            'law': 'Evidence rules / chain of custody',
            'applicability': 'HIGH',
            'analysis': 'Evidence collected without proper documentation may be inadmissible or impeachable',
            'relevant_activities': ['All evidence collection']
        })
        mitigations.append("Document all collection with timestamp, URL, and methodology")
        mitigations.append("Archive page snapshots with cryptographic hashing")
        mitigations.append("Engage attorney early to ensure evidence collection meets litigation standards")

    return {
        'risks': risks,
        'mitigations': list(set(mitigations)),
        'recommendation': 'Consult qualified legal counsel before proceeding with high-risk activities',
        'disclaimer': 'This assessment is a framework for discussion with legal counsel, not legal advice'
    }
```

---

## 26.6 Incident Response for Investigators

When OPSEC is breached — when a subject discovers the investigation, when a device is compromised, when a source's identity is endangered — a prepared response minimizes harm.

### Incident Taxonomy

1. **Source exposure**: Investigation subject or third party may have identified a source
2. **Investigation exposure**: Subject knows they are being investigated; may destroy evidence or flee
3. **Device compromise**: Investigator device or account may be compromised by adversary
4. **Premature publication**: Investigation findings published before proper verification

### Response Framework

```python
INCIDENT_RESPONSE_PLAYBOOKS = {
    "source_exposure": {
        "immediate_actions": [
            "Alert source immediately through most secure available channel",
            "Document when and how exposure may have occurred",
            "Assess what information adversary may have obtained",
            "Contact organization legal counsel",
            "Do not discuss exposure details via potentially compromised channels"
        ],
        "within_24_hours": [
            "Assess source's safety and need for protective action",
            "Rotate all credentials that may be compromised",
            "Review investigation methodology for how exposure occurred",
            "Evaluate whether to pause, accelerate, or abort investigation"
        ],
        "documentation": [
            "Create incident timeline",
            "Document what was exposed and what was protected",
            "Record all communications about incident"
        ]
    },

    "device_compromise": {
        "immediate_actions": [
            "Disconnect device from network (WiFi and cellular)",
            "Do NOT attempt to log out of accounts — may alert attacker",
            "Contact security team or IT immediately",
            "From a DIFFERENT device: change passwords for all critical accounts",
            "Enable MFA on any accounts not yet protected"
        ],
        "within_24_hours": [
            "Full forensic imaging of compromised device (do not wipe first)",
            "Audit access logs on all platforms for unauthorized activity",
            "Notify sources who may have communicated via compromised device",
            "Assess what investigation data was accessible on device"
        ],
        "recovery": [
            "Restore from known-clean backup",
            "Implement additional endpoint security",
            "Review and tighten all security settings"
        ]
    },

    "investigation_exposure": {
        "immediate_actions": [
            "Assess how subject became aware (via counter-surveillance, third party, leak)",
            "Preserve all currently-collected evidence immediately — subject may destroy",
            "Do not confront subject about knowing",
            "Evaluate investigative timeline — can collection be accelerated?"
        ],
        "within_24_hours": [
            "Brief legal counsel and client/editor",
            "Assess whether subject actions (evidence destruction, flight) have legal implications",
            "Determine whether to proceed, accelerate, or abort"
        ]
    }
}
```

---

## Summary

Operational security and risk management are not constraints on effective investigation — they are the infrastructure that makes sustained, credible investigation possible. An investigator who burns sources, faces legal action, or compromises findings through OPSEC failures is not just personally harmed; they undermine trust in the investigative methods and institutions they represent.

Risk management in OSINT follows the same structured approach as any risk discipline: identify assets, model threats, assess likelihood and impact, implement controls proportionate to risk, and maintain an incident response capability for when controls fail.

---

## Common Mistakes and Pitfalls

- **OPSEC theater**: Implementing visible security measures (VPN, Tor) while ignoring behavioral OPSEC (logging into personal accounts during investigation sessions)
- **No legal review budget**: Investigation organizations that don't budget for legal review often discover they need it after publication, when it's too late
- **Assuming all collection is legal**: The legality of data collection varies significantly by jurisdiction and method — assumptions based on US law may be wrong in EU contexts
- **Inadequate data retention policies**: Keeping investigation data longer than necessary creates unnecessary exposure
- **Source over-trust**: Sources have interests and limitations — building an investigation on unverified source claims is a documentation and credibility risk

---

## Further Reading

- EFF Surveillance Self-Defense — ssd.eff.org
- CPJ (Committee to Protect Journalists) — digital safety resources for journalists
- Freedom of the Press Foundation — digital security training
- Genie in a Bottle: Understanding Computer Fraud and Abuse Act Risk (legal analysis)
- GDPR and Journalism — Data Protection Network guidance
