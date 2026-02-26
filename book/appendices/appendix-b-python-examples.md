# Appendix B: Python Code Examples Reference

This appendix provides a consolidated reference to the Python code patterns used throughout the book, organized by function. Each section includes the key imports, class/function signatures, and usage examples.

---

## B.1 HTTP Requests and Rate Limiting

```python
import requests
import time
from functools import wraps
from typing import Optional, Dict

class RateLimitedSession:
    """HTTP session with built-in rate limiting"""

    def __init__(self, requests_per_second: float = 1.0, user_agent: str = "OSINT/1.0"):
        self.session = requests.Session()
        self.session.headers.update({'User-Agent': user_agent})
        self.min_interval = 1.0 / requests_per_second
        self.last_request = 0

    def get(self, url: str, **kwargs) -> requests.Response:
        elapsed = time.time() - self.last_request
        if elapsed < self.min_interval:
            time.sleep(self.min_interval - elapsed)

        response = self.session.get(url, **kwargs)
        self.last_request = time.time()
        return response

    def get_json(self, url: str, params: Dict = None, **kwargs) -> Optional[Dict]:
        try:
            response = self.get(url, params=params, timeout=30, **kwargs)
            response.raise_for_status()
            return response.json()
        except (requests.RequestException, ValueError) as e:
            print(f"Request error for {url}: {e}")
            return None


# Usage
session = RateLimitedSession(requests_per_second=0.5)
data = session.get_json("https://api.example.com/endpoint", params={"q": "query"})
```

---

## B.2 Certificate Transparency Log Search

```python
import requests
from typing import Set

def ct_log_subdomains(domain: str) -> Set[str]:
    """Discover subdomains via crt.sh Certificate Transparency logs"""
    subdomains = set()

    try:
        response = requests.get(
            f"https://crt.sh/?q=%.{domain}&output=json",
            timeout=30
        )
        if response.status_code == 200:
            for cert in response.json():
                for name in cert.get('name_value', '').split('\n'):
                    name = name.strip().lstrip('*.')
                    if name.endswith(domain) and '.' in name:
                        subdomains.add(name.lower())
    except Exception as e:
        print(f"CT log error: {e}")

    return subdomains


# DNS resolution for discovered subdomains
import socket
from concurrent.futures import ThreadPoolExecutor

def resolve_domain(domain: str) -> dict:
    try:
        ip = socket.gethostbyname(domain)
        return {'domain': domain, 'ip': ip, 'resolved': True}
    except socket.gaierror:
        return {'domain': domain, 'ip': None, 'resolved': False}

def bulk_resolve(domains: list, max_workers: int = 20) -> list:
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        results = list(executor.map(resolve_domain, domains))
    return results
```

---

## B.3 spaCy NLP Pipeline

```python
import spacy
from typing import List, Dict

# Load model: python -m spacy download en_core_web_lg
nlp = spacy.load("en_core_web_lg")

OSINT_LABELS = {
    'PERSON': 'person',
    'ORG': 'organization',
    'GPE': 'location',
    'LOC': 'location',
    'MONEY': 'financial',
    'DATE': 'temporal',
    'FAC': 'facility',
    'PRODUCT': 'product'
}

def extract_entities(text: str) -> List[Dict]:
    """Extract named entities from text"""
    doc = nlp(text[:100000])  # spaCy has token limit
    entities = []

    for ent in doc.ents:
        if ent.label_ in OSINT_LABELS:
            entities.append({
                'text': ent.text,
                'label': OSINT_LABELS[ent.label_],
                'start': ent.start_char,
                'end': ent.end_char,
                'confidence': None  # spaCy doesn't expose confidence directly
            })

    return entities

def batch_extract(texts: List[str]) -> List[List[Dict]]:
    """Batch processing for efficiency"""
    results = []
    for doc in nlp.pipe(texts, batch_size=50):
        entities = [
            {'text': ent.text, 'label': OSINT_LABELS.get(ent.label_, ent.label_)}
            for ent in doc.ents if ent.label_ in OSINT_LABELS
        ]
        results.append(entities)
    return results
```

---

## B.4 Anthropic API Integration

```python
import anthropic
from typing import List, Dict, Optional

client = anthropic.Anthropic()  # Uses ANTHROPIC_API_KEY env var

def analyze_with_claude(
    text: str,
    task: str,
    model: str = "claude-sonnet-4-6",
    max_tokens: int = 1024
) -> str:
    """Basic Claude API call for analysis"""
    response = client.messages.create(
        model=model,
        max_tokens=max_tokens,
        messages=[{"role": "user", "content": f"{task}\n\nCONTENT:\n{text[:4000]}"}]
    )
    return response.content[0].text


def structured_extraction(text: str, schema: Dict) -> str:
    """Extract structured data matching a schema"""
    schema_str = str(schema)

    prompt = f"""Extract information from the following text and return a JSON object matching this schema:

SCHEMA:
{schema_str}

TEXT:
{text[:3000]}

Return only valid JSON, no other text."""

    response = client.messages.create(
        model="claude-haiku-4-5-20251001",  # Use Haiku for simple extraction
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}]
    )
    return response.content[0].text


def tool_use_agent(
    goal: str,
    tools: List[Dict],
    tool_executor: callable,
    max_turns: int = 10
) -> str:
    """Agentic tool-use loop"""
    messages = [{"role": "user", "content": goal}]

    for turn in range(max_turns):
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=4096,
            tools=tools,
            messages=messages
        )

        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason == "end_turn":
            # Extract final text
            for block in response.content:
                if hasattr(block, 'text'):
                    return block.text
            return "Agent completed without final text"

        if response.stop_reason == "tool_use":
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    result = tool_executor(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": str(result)
                    })
            messages.append({"role": "user", "content": tool_results})

    return "Max turns reached"
```

---

## B.5 NetworkX Graph Analysis

```python
import networkx as nx
import matplotlib.pyplot as plt
from typing import List, Dict, Tuple

def build_investigation_graph(entities: List[Dict], relationships: List[Dict]) -> nx.DiGraph:
    """Build a directed graph from entities and relationships"""
    G = nx.DiGraph()

    for entity in entities:
        G.add_node(
            entity['id'],
            label=entity.get('label', entity['id']),
            entity_type=entity.get('type', 'unknown'),
            **entity.get('attributes', {})
        )

    for rel in relationships:
        G.add_edge(
            rel['source'],
            rel['target'],
            relationship=rel.get('type', 'linked_to'),
            weight=rel.get('weight', 1),
            confidence=rel.get('confidence', 0.5)
        )

    return G

def analyze_graph(G: nx.DiGraph) -> Dict:
    """Calculate key graph metrics"""
    if len(G.nodes()) == 0:
        return {}

    metrics = {
        'node_count': len(G.nodes()),
        'edge_count': len(G.edges()),
        'density': nx.density(G),
    }

    # Centrality measures
    try:
        metrics['degree_centrality'] = dict(
            sorted(nx.degree_centrality(G).items(), key=lambda x: x[1], reverse=True)[:10]
        )
        metrics['betweenness_centrality'] = dict(
            sorted(nx.betweenness_centrality(G).items(), key=lambda x: x[1], reverse=True)[:10]
        )
    except Exception:
        pass

    # Community detection (requires undirected graph)
    try:
        import community as community_louvain
        G_undirected = G.to_undirected()
        partition = community_louvain.best_partition(G_undirected)
        metrics['communities'] = len(set(partition.values()))
    except ImportError:
        pass

    return metrics

def visualize_graph(G: nx.DiGraph, output_path: str = None, figsize: Tuple = (15, 10)):
    """Visualize investigation graph"""
    plt.figure(figsize=figsize)

    pos = nx.spring_layout(G, k=2, seed=42)

    # Color nodes by type
    color_map = {
        'person': '#FF6B6B',
        'organization': '#4ECDC4',
        'domain': '#45B7D1',
        'ip_address': '#96CEB4',
        'location': '#FFEAA7'
    }

    node_colors = [
        color_map.get(G.nodes[n].get('entity_type', 'unknown'), '#DDD')
        for n in G.nodes()
    ]

    nx.draw_networkx_nodes(G, pos, node_color=node_colors, node_size=500, alpha=0.9)
    nx.draw_networkx_edges(G, pos, edge_color='gray', arrows=True, alpha=0.6)
    nx.draw_networkx_labels(G, pos, {n: G.nodes[n].get('label', n)[:20] for n in G.nodes()},
                           font_size=8)

    plt.axis('off')
    plt.tight_layout()

    if output_path:
        plt.savefig(output_path, dpi=150, bbox_inches='tight')
    else:
        plt.show()

    plt.close()
```

---

## B.6 Document Processing Pipeline

```python
import pdfplumber
import pytesseract
from PIL import Image
import cv2
import numpy as np
from pathlib import Path
from typing import Dict, List

def extract_pdf_text(pdf_path: str) -> Dict:
    """Extract text and metadata from PDF"""
    result = {
        'pages': [],
        'metadata': {},
        'total_pages': 0,
        'errors': []
    }

    try:
        with pdfplumber.open(pdf_path) as pdf:
            result['total_pages'] = len(pdf.pages)
            result['metadata'] = pdf.metadata or {}

            for i, page in enumerate(pdf.pages):
                try:
                    text = page.extract_text() or ''
                    tables = page.extract_tables() or []
                    result['pages'].append({
                        'page_number': i + 1,
                        'text': text,
                        'table_count': len(tables),
                        'tables': tables
                    })
                except Exception as e:
                    result['errors'].append(f"Page {i+1}: {e}")

    except Exception as e:
        result['errors'].append(f"PDF open error: {e}")

    return result


def ocr_image(image_path: str, language: str = 'eng') -> str:
    """Extract text from image using Tesseract OCR"""
    # Preprocess image
    img = cv2.imread(image_path)
    if img is None:
        return ''

    # Grayscale
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)

    # Denoise
    denoised = cv2.fastNlMeansDenoising(gray)

    # Binarize with Otsu
    _, binary = cv2.threshold(denoised, 0, 255, cv2.THRESH_BINARY + cv2.THRESH_OTSU)

    # OCR
    config = f'--oem 3 --psm 6 -l {language}'
    text = pytesseract.image_to_string(binary, config=config)

    return text.strip()


def extract_exif(file_path: str) -> Dict:
    """Extract EXIF metadata using exiftool"""
    import subprocess, json

    try:
        result = subprocess.run(
            ['exiftool', '-json', '-a', '-G', file_path],
            capture_output=True, text=True, timeout=15
        )
        if result.returncode == 0:
            data = json.loads(result.stdout)
            return data[0] if data else {}
    except (FileNotFoundError, subprocess.TimeoutExpired, json.JSONDecodeError):
        pass
    return {}
```

---

## B.7 Data Deduplication

```python
import hashlib
import re
from typing import List, Dict

def content_fingerprint(text: str) -> str:
    """Exact content fingerprint via SHA-256"""
    normalized = re.sub(r'\s+', ' ', text.lower().strip())
    return hashlib.sha256(normalized.encode()).hexdigest()

def simhash(text: str, bits: int = 64) -> int:
    """SimHash for near-duplicate detection"""
    tokens = text.lower().split()
    vector = [0] * bits

    for token in tokens:
        token_hash = int(hashlib.md5(token.encode()).hexdigest(), 16)
        for i in range(bits):
            vector[i] += 1 if token_hash & (1 << i) else -1

    result = 0
    for i in range(bits):
        if vector[i] > 0:
            result |= (1 << i)
    return result

def hamming_distance(h1: int, h2: int) -> int:
    return bin(h1 ^ h2).count('1')

class DeduplicatingStore:
    """Store that deduplicates by both exact and near-duplicate"""

    def __init__(self, similarity_threshold: int = 5):
        self.exact_hashes = set()
        self.simhashes = []
        self.threshold = similarity_threshold
        self.items = []

    def add(self, text: str, metadata: Dict = None) -> bool:
        """Returns True if item was added (not duplicate)"""
        fp = content_fingerprint(text)

        # Exact duplicate check
        if fp in self.exact_hashes:
            return False

        # Near-duplicate check
        sh = simhash(text)
        for existing_sh in self.simhashes[-5000:]:
            if hamming_distance(sh, existing_sh) <= self.threshold:
                return False

        # Not a duplicate
        self.exact_hashes.add(fp)
        self.simhashes.append(sh)
        self.items.append({'text': text, 'fingerprint': fp, **(metadata or {})})
        return True
```

---

## B.8 Elasticsearch Integration

```python
from elasticsearch import Elasticsearch, helpers
from typing import List, Dict, Generator
from datetime import datetime

es = Elasticsearch(['http://localhost:9200'])

def bulk_index_documents(index_name: str, documents: List[Dict]) -> Dict:
    """Bulk index documents efficiently"""
    def generate_actions(docs: List[Dict]) -> Generator:
        for doc in docs:
            yield {
                "_index": index_name,
                "_id": doc.get('id') or doc.get('fingerprint'),
                "_source": {**doc, "@timestamp": datetime.utcnow().isoformat()}
            }

    success, failed = helpers.bulk(
        es,
        generate_actions(documents),
        raise_on_error=False,
        stats_only=True
    )
    return {'indexed': success, 'failed': failed}

def search_entities(index_pattern: str, entity_type: str,
                   entity_value: str, days_back: int = 30) -> Dict:
    """Search for entity occurrences"""
    return es.search(
        index=index_pattern,
        body={
            "query": {
                "bool": {
                    "must": [
                        {"nested": {
                            "path": "entities",
                            "query": {"bool": {"must": [
                                {"term": {"entities.type": entity_type}},
                                {"term": {"entities.value": entity_value.lower()}}
                            ]}}
                        }},
                        {"range": {"@timestamp": {"gte": f"now-{days_back}d"}}}
                    ]
                }
            },
            "sort": [{"@timestamp": {"order": "desc"}}],
            "size": 50
        }
    )
```

---

## B.9 Report Generation

```python
import anthropic
from datetime import datetime
from typing import List, Dict

def generate_investigation_report(
    subject: str,
    findings: List[Dict],
    investigation_type: str
) -> str:
    """Generate structured investigation report with AI assistance"""
    client = anthropic.Anthropic()

    findings_text = "\n\n".join([
        f"**{f.get('category', 'Finding').upper()}** [{f.get('confidence', 'Unknown confidence')}]\n"
        f"{f.get('finding', '')}\n"
        f"Source: {f.get('source', 'Unknown')}"
        for f in findings
    ])

    prompt = f"""Generate a professional investigation report based on these findings.

SUBJECT: {subject}
INVESTIGATION TYPE: {investigation_type}
DATE: {datetime.now().strftime('%Y-%m-%d')}

FINDINGS:
{findings_text}

Write a professional report with:
1. Executive Summary (2-3 sentences)
2. Methodology Note (sources consulted and methods used)
3. Findings by Category (organized and clear)
4. Risk Assessment
5. Recommendations
6. Limitations and Caveats

Maintain professional, objective tone. Cite source for each finding.
Do not overstate confidence beyond what evidence supports."""

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=3000,
        messages=[{"role": "user", "content": prompt}]
    )

    return response.content[0].text


def export_to_markdown(report_text: str, output_path: str) -> None:
    """Export report to markdown file"""
    with open(output_path, 'w', encoding='utf-8') as f:
        f.write(f"# Investigation Report\n\n")
        f.write(f"*Generated: {datetime.now().strftime('%Y-%m-%d %H:%M')}*\n\n")
        f.write("---\n\n")
        f.write(report_text)

    print(f"Report saved to {output_path}")
```

---

## B.10 Complete Requirements.txt

```
# Core HTTP and web
requests>=2.31.0
aiohttp>=3.9.0
httpx>=0.26.0
playwright>=1.40.0
beautifulsoup4>=4.12.0
lxml>=4.9.0
feedparser>=6.0.10

# NLP and ML
spacy>=3.7.0
transformers>=4.36.0
sentence-transformers>=2.3.0
torch>=2.1.0
pytesseract>=0.3.10
Pillow>=10.0.0
opencv-python>=4.8.0

# DNS and network
dnspython>=2.4.0
ipwhois>=1.2.0

# Data processing
pdfplumber>=0.10.0
pandas>=2.1.0
numpy>=1.26.0

# Database and storage
elasticsearch>=8.11.0
psycopg2-binary>=2.9.0
redis>=5.0.0
neo4j>=5.14.0
sqlalchemy>=2.0.0

# Task management
celery>=5.3.0
kombu>=5.3.0

# Visualization
plotly>=5.18.0
networkx>=3.2.0
matplotlib>=3.8.0
folium>=0.15.0

# AI APIs
anthropic>=0.21.0
openai>=1.10.0

# Security and crypto
cryptography>=41.0.0

# Utilities
python-dotenv>=1.0.0
click>=8.1.0
tqdm>=4.66.0
loguru>=0.7.0
```

---

## B.11 Environment Variable Reference

```bash
# API Keys
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
GITHUB_TOKEN=ghp_...
SHODAN_API_KEY=...
VIRUSTOTAL_API_KEY=...
HUNTER_API_KEY=...
WHOISXML_KEY=...
NEWS_API_KEY=...
SECURITYTRAILS_KEY=...
ABUSEIPDB_KEY=...
DEEPL_API_KEY=...
HIVE_API_KEY=...

# Infrastructure
ELASTICSEARCH_URL=http://localhost:9200
REDIS_URL=redis://localhost:6379/0
POSTGRES_DSN=postgresql://user:pass@localhost/osint_db
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=...

# OSINT Config
INVESTIGATION_DATA_PATH=/path/to/secure/vault
OSINT_LOG_LEVEL=INFO
RATE_LIMIT_REQUESTS_PER_SECOND=1.0

# Notification
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...
```
