# Chapter 16: AI-Enhanced Investigative Platforms

## Learning Objectives

By the end of this chapter, you will be able to:
- Evaluate AI-enhanced OSINT platforms against investigative requirements
- Integrate LLM APIs into existing investigative workflows
- Build custom AI-enhanced tools for specific investigative domains
- Assess the accuracy and reliability of AI platform outputs
- Manage the operational and cost considerations of AI platform use
- Combine multiple AI platforms for comprehensive investigative coverage

---

## 16.1 The AI Platform Landscape

Dedicated OSINT platforms with AI enhancement are emerging rapidly. Some are established companies adding AI features to existing platforms; others are AI-first tools built specifically to leverage LLM and ML capabilities for investigative work.

This chapter covers both the commercial landscape and the approach to building custom AI-enhanced tooling — because the commercial landscape will continue to evolve, and investigators who understand the underlying architecture can evaluate new entrants and adapt their workflows.

---

## 16.2 Commercial AI-Enhanced Platforms

### Palantir Technologies

Palantir's Gotham and Foundry platforms are the enterprise standard for large-scale analytical environments that integrate OSINT, internal data, and AI analysis. They are primarily used by government agencies, large financial institutions, and major corporations.

*What they do well*: Massive-scale data ingestion, advanced graph analysis, temporal pattern analysis, integration with classified and proprietary data sources.

*Limitations*: Extremely expensive, complex to deploy, primarily appropriate for large organizational use.

*AI integration*: Palantir has integrated LLM capabilities for natural language querying, automated hypothesis generation, and analytical report drafting.

### i2 Analyst's Notebook (IBM)

Long the standard for law enforcement and intelligence analyst workstations. IBM has integrated AI assistance into the platform including pattern detection, entity resolution, and analytical assistance.

*What it does well*: Link analysis visualization, timeline analysis, evidence management, case documentation.

*Limitations*: Windows-based, expensive, high learning curve.

### Skopenow

Commercial OSINT platform aimed at insurance investigations, legal, and HR use cases. Aggregates social media, court records, news, and other sources with AI-powered relevance scoring.

*What it does well*: Automated multi-source aggregation with risk scoring, report generation, FCRA-compliant searches.

*Limitations*: Less depth than specialist tools in specific domains; coverage gaps in some jurisdictions.

### Babel Street

Commercial AI platform for analyzing text in 200+ languages at scale. Designed for government and intelligence agency use.

*What it does well*: Multilingual text analysis at scale, social media monitoring in non-English sources, translation with cultural context preservation.

### OSINT Industries

Automated OSINT aggregation platform aimed at investigators and corporate users. Queries multiple data sources and aggregates results.

### Lampyre

AI-powered OSINT platform with emphasis on corporate and financial investigations.

---

## 16.3 Building Custom AI-Enhanced Investigative Tools

For investigators who need capabilities not available from commercial platforms, or who want to integrate AI into specific workflows, building custom tools using LLM APIs is increasingly practical.

### The Investigation Assistant Architecture

A custom AI investigative assistant integrates:
1. **Data access layer**: APIs to relevant data sources
2. **Processing layer**: NLP, entity extraction, vector storage
3. **AI reasoning layer**: LLM for analysis and synthesis
4. **Interface layer**: Command-line, web UI, or API

```python
import anthropic
import json
from datetime import datetime
from typing import Optional
import sqlite3

class InvestigativeAssistant:
    """
    AI-powered investigative assistant with tool integration
    Combines LLM reasoning with data source access
    """

    def __init__(self, anthropic_api_key: str, database_path: str = 'investigation.db'):
        self.client = anthropic.Anthropic(api_key=anthropic_api_key)
        self.db_path = database_path
        self.conversation_history = []
        self.current_investigation = None
        self._init_database()

    def _init_database(self):
        """Initialize investigation database"""
        with sqlite3.connect(self.db_path) as conn:
            conn.execute('''
                CREATE TABLE IF NOT EXISTS investigations (
                    id TEXT PRIMARY KEY,
                    name TEXT,
                    created_at TEXT,
                    subjects TEXT,
                    status TEXT DEFAULT 'active'
                )
            ''')
            conn.execute('''
                CREATE TABLE IF NOT EXISTS findings (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    investigation_id TEXT,
                    finding_type TEXT,
                    content TEXT,
                    source TEXT,
                    confidence TEXT,
                    added_at TEXT,
                    ai_analysis TEXT
                )
            ''')
            conn.execute('''
                CREATE TABLE IF NOT EXISTS messages (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    investigation_id TEXT,
                    role TEXT,
                    content TEXT,
                    timestamp TEXT
                )
            ''')
            conn.commit()

    def start_investigation(self, name: str, subjects: list) -> str:
        """Start a new investigation"""
        import uuid
        investigation_id = str(uuid.uuid4())[:8]

        with sqlite3.connect(self.db_path) as conn:
            conn.execute(
                'INSERT INTO investigations VALUES (?, ?, ?, ?, ?)',
                (investigation_id, name, datetime.now().isoformat(),
                 json.dumps(subjects), 'active')
            )

        self.current_investigation = investigation_id
        self.conversation_history = []

        print(f"✓ Investigation '{name}' started (ID: {investigation_id})")
        print(f"  Subjects: {', '.join(subjects)}")
        return investigation_id

    def add_finding(self, finding_type: str, content: str, source: str,
                   confidence: str = 'medium') -> int:
        """Add a finding to the current investigation"""
        if not self.current_investigation:
            raise ValueError("No active investigation. Call start_investigation() first.")

        # Get AI analysis of the finding
        ai_analysis = self._analyze_finding(finding_type, content)

        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.execute(
                'INSERT INTO findings (investigation_id, finding_type, content, source, confidence, added_at, ai_analysis) VALUES (?, ?, ?, ?, ?, ?, ?)',
                (self.current_investigation, finding_type, content, source,
                 confidence, datetime.now().isoformat(), ai_analysis)
            )
            finding_id = cursor.lastrowid

        return finding_id

    def _analyze_finding(self, finding_type: str, content: str) -> str:
        """Use AI to analyze a single finding in context"""
        # Get existing findings for context
        existing_findings = self._get_investigation_findings()

        if not existing_findings:
            context = "This is the first finding in the investigation."
        else:
            context = f"Existing findings in this investigation:\n" + \
                      "\n".join(f"- [{f['type']}] {f['content'][:100]}..."
                                for f in existing_findings[-5:])

        prompt = f"""As an OSINT analyst, briefly analyze this new finding in context of the investigation:

Finding type: {finding_type}
Finding: {content}

{context}

Provide:
1. Key significance of this finding
2. How it connects to existing findings (if any)
3. Suggested follow-up queries or investigations
4. Confidence notes (reliability considerations)

Be concise — 3-5 sentences maximum."""

        response = self.client.messages.create(
            model="claude-haiku-4-5-20251001",  # Use Haiku for quick analysis
            max_tokens=512,
            messages=[{"role": "user", "content": prompt}]
        )
        return response.content[0].text

    def _get_investigation_findings(self) -> list:
        """Get all findings for the current investigation"""
        with sqlite3.connect(self.db_path) as conn:
            conn.row_factory = sqlite3.Row
            cursor = conn.execute(
                'SELECT * FROM findings WHERE investigation_id = ? ORDER BY added_at',
                (self.current_investigation,)
            )
            return [dict(row) for row in cursor.fetchall()]

    def analyze_all_findings(self) -> str:
        """Generate comprehensive analysis of all findings"""
        findings = self._get_investigation_findings()

        if not findings:
            return "No findings to analyze."

        findings_text = "\n".join([
            f"[{f['finding_type'].upper()}] Source: {f['source']}\n{f['content']}"
            for f in findings
        ])

        prompt = f"""You are an experienced OSINT analyst. Provide a comprehensive analysis of the following investigation findings.

INVESTIGATION ID: {self.current_investigation}
TOTAL FINDINGS: {len(findings)}

ALL FINDINGS:
{findings_text}

Provide:

## Executive Summary
[2-3 sentences summarizing the key findings]

## Key Relationships and Patterns
[Identify the most significant connections between findings]

## Timeline (if applicable)
[Chronological narrative of key events]

## Confidence Assessment
[Overall confidence level and key uncertainties]

## Information Gaps
[Critical information not established by current findings]

## Recommended Next Steps
[3-5 specific follow-up investigations]

## Risk Indicators
[Any red flags or concerning patterns]

Base analysis strictly on provided findings. Clearly distinguish confirmed facts from inferences."""

        response = self.client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=3000,
            messages=[{"role": "user", "content": prompt}]
        )
        return response.content[0].text

    def chat(self, user_message: str) -> str:
        """Conversational interface for investigation queries"""
        if not self.current_investigation:
            return "No active investigation. Please start one first."

        # Add investigation context to conversation
        findings = self._get_investigation_findings()
        findings_summary = "\n".join([
            f"[{f['finding_type']}] {f['content'][:200]}"
            for f in findings[-10:]  # Last 10 findings
        ])

        system_prompt = f"""You are an experienced OSINT analyst assisting with an active investigation.

Current investigation ID: {self.current_investigation}

Recent findings:
{findings_summary if findings_summary else "No findings yet."}

You can:
- Analyze findings and suggest patterns
- Suggest sources and investigative approaches
- Help formulate search queries
- Assess confidence levels
- Identify information gaps

Always base analysis on the findings provided. Note when making inferences vs. citing established facts."""

        # Add user message to history
        self.conversation_history.append({"role": "user", "content": user_message})

        # Get AI response
        response = self.client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=2048,
            system=system_prompt,
            messages=self.conversation_history
        )

        assistant_message = response.content[0].text

        # Add to conversation history
        self.conversation_history.append({
            "role": "assistant",
            "content": assistant_message
        })

        # Save to database
        with sqlite3.connect(self.db_path) as conn:
            conn.execute(
                'INSERT INTO messages (investigation_id, role, content, timestamp) VALUES (?, ?, ?, ?)',
                (self.current_investigation, 'user', user_message, datetime.now().isoformat())
            )
            conn.execute(
                'INSERT INTO messages (investigation_id, role, content, timestamp) VALUES (?, ?, ?, ?)',
                (self.current_investigation, 'assistant', assistant_message, datetime.now().isoformat())
            )

        return assistant_message

    def generate_report(self, format: str = 'markdown') -> str:
        """Generate a formatted investigation report"""
        findings = self._get_investigation_findings()

        with sqlite3.connect(self.db_path) as conn:
            conn.row_factory = sqlite3.Row
            investigation = conn.execute(
                'SELECT * FROM investigations WHERE id = ?',
                (self.current_investigation,)
            ).fetchone()

        if not investigation:
            return "Investigation not found."

        # Get comprehensive analysis
        analysis = self.analyze_all_findings()

        report_sections = [
            f"# Investigation Report",
            f"**Investigation**: {investigation['name']}",
            f"**ID**: {investigation['id']}",
            f"**Date**: {datetime.now().strftime('%Y-%m-%d')}",
            f"**Subjects**: {investigation['subjects']}",
            "",
            "---",
            "",
            analysis,
            "",
            "---",
            "",
            f"## Source Documentation ({len(findings)} findings)",
            "",
        ]

        # Add individual findings as appendix
        for i, finding in enumerate(findings, 1):
            report_sections.extend([
                f"### Finding {i}: {finding['finding_type'].title()}",
                f"**Source**: {finding['source']}",
                f"**Date Added**: {finding['added_at']}",
                f"**Confidence**: {finding['confidence']}",
                "",
                finding['content'],
                "",
                f"*AI Analysis*: {finding['ai_analysis']}",
                "",
            ])

        return '\n'.join(report_sections)
```

### Usage Example

```python
# Example investigation workflow
assistant = InvestigativeAssistant(api_key="your_key", database_path="my_investigation.db")

# Start new investigation
assistant.start_investigation(
    name="AcmeCorp Due Diligence",
    subjects=["AcmeCorp Inc", "John Smith CEO"]
)

# Add findings as they are collected
assistant.add_finding(
    finding_type="corporate_record",
    content="AcmeCorp Inc incorporated in Delaware on 2018-03-15. CIK: 1234567. Officers: John Smith (CEO), Jane Doe (CFO)",
    source="SEC EDGAR",
    confidence="high"
)

assistant.add_finding(
    finding_type="litigation",
    content="AcmeCorp Inc was named as defendant in case 22-cv-01234 SDNY. Filed 2022-06-15. Plaintiff alleges breach of contract. Settled 2023-01-10.",
    source="PACER",
    confidence="high"
)

assistant.add_finding(
    finding_type="news",
    content="CEO John Smith previously led TechStartup Inc (2015-2018) which shut down after fraud investigation by SEC.",
    source="TechCrunch article, 2018-11-15",
    confidence="medium"
)

# Interactive chat
response = assistant.chat("What are the key risk indicators for this investment?")
print(response)

response = assistant.chat("What additional information do we need to assess the litigation risk?")
print(response)

# Generate full report
report = assistant.generate_report()
with open("acmecorp_report.md", "w") as f:
    f.write(report)
```

---

## 16.4 AI-Enhanced Monitoring Platforms

For ongoing monitoring of subjects, entities, or topics:

```python
import anthropic
from dataclasses import dataclass
from typing import List, Dict
import json

@dataclass
class MonitoringAlert:
    """Alert generated by AI monitoring"""
    alert_id: str
    severity: str  # 'critical', 'high', 'medium', 'low'
    subject: str
    trigger_description: str
    evidence: List[str]
    recommended_actions: List[str]
    generated_at: str

class AIMonitoringEngine:
    """AI-powered monitoring engine for continuous OSINT surveillance"""

    def __init__(self, client: anthropic.Anthropic):
        self.client = client
        self.monitoring_profiles = {}

    def add_monitoring_profile(self, subject_id: str, subject_info: dict, alert_conditions: list):
        """Add a subject to monitor"""
        self.monitoring_profiles[subject_id] = {
            'info': subject_info,
            'conditions': alert_conditions,
            'baseline': {},
            'history': []
        }

    def evaluate_new_content(self, subject_id: str, new_content: List[dict]) -> List[MonitoringAlert]:
        """
        Evaluate new content against monitoring profile
        Returns any triggered alerts
        """
        if subject_id not in self.monitoring_profiles:
            return []

        profile = self.monitoring_profiles[subject_id]
        alerts = []

        # Format new content for AI evaluation
        content_text = "\n\n".join([
            f"SOURCE: {c.get('source', 'Unknown')}\nDATE: {c.get('date', 'Unknown')}\nCONTENT: {c.get('text', '')[:500]}"
            for c in new_content
        ])

        conditions_text = "\n".join([f"- {c}" for c in profile['conditions']])

        prompt = f"""You are monitoring an entity for relevant changes and alert conditions.

MONITORED SUBJECT:
{json.dumps(profile['info'], indent=2)}

ALERT CONDITIONS TO CHECK:
{conditions_text}

NEW CONTENT TO EVALUATE:
{content_text}

Evaluate whether any alert conditions are triggered by the new content.

For each triggered alert, provide:
1. Alert severity (critical/high/medium/low)
2. Which condition was triggered
3. Specific evidence from the content
4. Recommended actions

If no alert conditions are triggered, say "NO ALERTS TRIGGERED".

Format your response as JSON:
{{
  "alerts": [
    {{
      "severity": "high",
      "condition": "condition that was triggered",
      "evidence": ["specific quote or fact from content"],
      "recommended_actions": ["specific action 1", "action 2"]
    }}
  ]
}}"""

        response = self.client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}]
        )

        try:
            # Parse JSON response
            response_text = response.content[0].text

            # Extract JSON from response
            import re
            json_match = re.search(r'\{.*\}', response_text, re.DOTALL)
            if json_match:
                alert_data = json.loads(json_match.group())

                for alert_dict in alert_data.get('alerts', []):
                    if alert_dict.get('severity') != 'none':
                        import uuid
                        alert = MonitoringAlert(
                            alert_id=str(uuid.uuid4())[:8],
                            severity=alert_dict.get('severity', 'medium'),
                            subject=subject_id,
                            trigger_description=alert_dict.get('condition', ''),
                            evidence=alert_dict.get('evidence', []),
                            recommended_actions=alert_dict.get('recommended_actions', []),
                            generated_at=datetime.now().isoformat()
                        )
                        alerts.append(alert)

        except json.JSONDecodeError:
            # If AI didn't return proper JSON, check for text indicators
            if "NO ALERTS TRIGGERED" not in response_text.upper():
                # There may be alerts but format was wrong
                pass

        return alerts
```

---

## 16.5 Evaluating AI Platform Accuracy

AI platforms make errors. Systematic evaluation before production use prevents deploying inaccurate tools on live investigations.

### Accuracy Evaluation Framework

```python
class PlatformEvaluator:
    """Framework for evaluating AI OSINT platform accuracy"""

    def __init__(self, platform_name: str):
        self.platform_name = platform_name
        self.test_cases = []
        self.results = []

    def add_test_case(self, input_data: dict, expected_output: dict, test_type: str):
        """Add a test case with known correct answer"""
        self.test_cases.append({
            'input': input_data,
            'expected': expected_output,
            'type': test_type
        })

    def evaluate_entity_extraction(self, platform_function, test_text: str, known_entities: dict) -> dict:
        """
        Evaluate how well a platform extracts entities
        known_entities: {'PERSON': ['John Smith'], 'ORG': ['AcmeCorp']}
        """
        extracted = platform_function(test_text)

        results = {}
        for entity_type, expected in known_entities.items():
            found = [e.lower() for e in extracted.get(entity_type, [])]

            # Calculate precision and recall
            true_positives = sum(1 for e in expected if e.lower() in found)
            false_positives = sum(1 for e in found if e not in [x.lower() for x in expected])
            false_negatives = len(expected) - true_positives

            precision = true_positives / (true_positives + false_positives) if (true_positives + false_positives) > 0 else 0
            recall = true_positives / (true_positives + false_negatives) if (true_positives + false_negatives) > 0 else 0
            f1 = 2 * (precision * recall) / (precision + recall) if (precision + recall) > 0 else 0

            results[entity_type] = {
                'precision': round(precision, 3),
                'recall': round(recall, 3),
                'f1': round(f1, 3),
                'missed': [e for e in expected if e.lower() not in found],
                'false_positives': [e for e in found if e not in [x.lower() for x in expected]]
            }

        return results

    def run_accuracy_report(self) -> dict:
        """Generate comprehensive accuracy report"""
        if not self.results:
            return {'error': 'No evaluation results available'}

        return {
            'platform': self.platform_name,
            'test_cases': len(self.test_cases),
            'overall_accuracy': sum(r.get('correct', False) for r in self.results) / len(self.results),
            'by_type': {}  # Aggregate by test type
        }
```

---

## Summary

AI-enhanced investigative platforms represent the frontier of modern OSINT capability. Commercial platforms like Palantir, i2, and specialized tools provide integrated environments for large-scale investigations. For practitioners who need custom solutions, LLM APIs enable building purpose-built investigative assistants and monitoring systems.

Building custom AI tools requires designing around the fundamental capabilities: document analysis, entity extraction, pattern recognition, hypothesis generation, and report synthesis. The key architectural elements are: a conversational interface for ad-hoc queries, a structured data store for findings and context, AI-powered analysis of individual findings and aggregate patterns, and automated report generation.

Accuracy evaluation is mandatory before production deployment. Platform errors in investigative contexts have real consequences.

---

## Common Mistakes and Pitfalls

- **AI platform over-trust**: Commercial AI platforms make errors; findings require verification
- **Context window neglect**: Long investigations need context management strategies; AI models can only process limited context
- **Conversation history management**: Storing too much conversation history in context degrades AI performance
- **API cost forecasting**: AI API costs at scale can be significant; model selection and request optimization matter
- **Privacy in cloud AI**: Investigation subjects' data fed to cloud AI APIs requires privacy analysis

---

## Further Reading

- Palantir Technologies documentation and case studies
- LangChain documentation for building AI-powered applications
- Anthropic model evaluation methodology
- NIST AI Risk Management Framework
