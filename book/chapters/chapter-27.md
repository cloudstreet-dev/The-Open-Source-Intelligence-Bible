# Chapter 27: Designing and Building Your OSINT Stack

## Learning Objectives

By the end of this chapter, you will be able to:
- Architect an OSINT technical stack appropriate for your scale and use case
- Select and integrate tools based on functional requirements and operational constraints
- Build custom collection and processing components where commercial tools fall short
- Manage API keys, credentials, and rate limits across a multi-tool ecosystem
- Design a data model that supports pivot-based investigation
- Make build-vs-buy decisions for OSINT infrastructure components

---

## 27.1 Stack Design Philosophy

Building an OSINT stack is an architectural problem: matching capabilities to requirements, tools to workflows, and infrastructure to operational constraints. The right stack for a solo journalist bears no resemblance to the right stack for a financial crime team at a global bank.

Before selecting tools, define requirements:

**Collection scope**: Which data types do you need? Social media, public records, network data, documents, imagery? Each requires different tooling.

**Operational tempo**: Real-time monitoring needs stream processing; episodic investigations can use batch workflows. The architecture is fundamentally different.

**Team size and technical capability**: A Python-fluent developer can build and maintain custom collectors; a non-technical investigator needs GUI tools and managed services.

**Budget constraints**: The difference between a free-tier stack and a production enterprise stack can be $50,000+/year in commercial API costs alone.

**Legal operating environment**: GDPR, CCPA, and sector-specific regulations may prohibit certain data types or require specific handling — these are architectural constraints, not just policies.

---

## 27.2 Core Stack Components

```python
"""
OSINT Stack Architecture Reference
"""

OSINT_STACK_ARCHITECTURE = {
    "collection_layer": {
        "description": "Tools and systems that gather raw data from sources",
        "components": {
            "web_scrapers": {
                "options": ["BeautifulSoup + requests", "Playwright (JS rendering)", "Scrapy (crawler)"],
                "use_when": "Source has no API; static or dynamic web content"
            },
            "api_clients": {
                "options": ["Custom requests wrappers", "Official SDKs", "Third-party clients"],
                "use_when": "Source provides official API — always prefer API over scraping"
            },
            "rss_feeds": {
                "options": ["feedparser library", "FreshRSS (self-hosted reader)", "RSS-to-Webhook"],
                "use_when": "News sources, blogs, government data feeds"
            },
            "platform_monitors": {
                "options": ["Twitter API v2", "Reddit API (PRAW)", "Telegram tdlib"],
                "use_when": "Social media monitoring requirements"
            },
            "document_fetchers": {
                "options": ["PACER API", "SEC EDGAR bulk download", "Custom PDF fetchers"],
                "use_when": "Regulatory filings, court records, government documents"
            }
        }
    },

    "processing_layer": {
        "description": "Transform raw data into structured intelligence",
        "components": {
            "nlp_pipeline": {
                "options": ["spaCy", "HuggingFace transformers", "Claude API"],
                "use_when": "Entity extraction, classification, summarization"
            },
            "ocr": {
                "options": ["pytesseract + OpenCV", "Google Cloud Vision", "AWS Textract"],
                "use_when": "Scanned documents, image-embedded text"
            },
            "translation": {
                "options": ["Claude API", "DeepL API", "Google Translate API"],
                "use_when": "Multi-language sources"
            },
            "deduplication": {
                "options": ["SimHash (custom)", "MinHash + LSH", "Elasticsearch near-duplicate detection"],
                "use_when": "High-volume collection with significant overlap"
            }
        }
    },

    "storage_layer": {
        "description": "Persistent storage for collected and processed data",
        "components": {
            "document_store": {
                "options": ["Elasticsearch", "OpenSearch (self-hosted ES)", "Typesense (simpler)"],
                "use_when": "Full-text search across large document collections"
            },
            "relational": {
                "options": ["PostgreSQL", "SQLite (small scale)"],
                "use_when": "Structured entity data, relationships, metadata"
            },
            "graph_database": {
                "options": ["Neo4j", "ArangoDB", "NetworkX (in-memory)"],
                "use_when": "Relationship networks, pivot-based investigation"
            },
            "object_storage": {
                "options": ["MinIO (self-hosted)", "AWS S3", "Cloudflare R2"],
                "use_when": "Raw document storage, images, large files"
            },
            "cache": {
                "options": ["Redis", "Memcached"],
                "use_when": "API response caching, rate limit management, session data"
            },
            "vector_store": {
                "options": ["pgvector (PostgreSQL extension)", "Chroma", "Pinecone"],
                "use_when": "Semantic similarity search, RAG implementations"
            }
        }
    },

    "analysis_layer": {
        "description": "Tools for making sense of collected data",
        "components": {
            "llm_integration": {
                "options": ["Anthropic API (Claude)", "OpenAI API", "Ollama (local models)"],
                "use_when": "Analysis, summarization, report generation, entity disambiguation"
            },
            "graph_analysis": {
                "options": ["NetworkX (Python)", "Gephi (desktop)", "Neo4j GDS"],
                "use_when": "Network centrality, community detection, path finding"
            },
            "geospatial": {
                "options": ["QGIS", "Folium (Python)", "Kepler.gl"],
                "use_when": "Geographic visualization, satellite imagery overlay"
            },
            "timeline": {
                "options": ["Plotly (Python)", "TimelineJS (browser)", "Knight Lab Timeline"],
                "use_when": "Temporal analysis, event sequencing"
            }
        }
    },

    "orchestration_layer": {
        "description": "Managing collection schedules, workflows, and pipeline execution",
        "components": {
            "task_queue": {
                "options": ["Celery + Redis", "RQ (simpler)", "Apache Airflow (complex DAGs)"],
                "use_when": "Distributed task processing, scheduled collection"
            },
            "streaming": {
                "options": ["Apache Kafka", "Redis Streams (simpler)", "RabbitMQ"],
                "use_when": "Real-time data pipelines, event-driven processing"
            },
            "workflow_orchestration": {
                "options": ["Apache Airflow", "Prefect", "Dagster"],
                "use_when": "Complex multi-step workflows with dependencies"
            }
        }
    },

    "presentation_layer": {
        "description": "User interfaces and reporting for investigation findings",
        "components": {
            "investigation_platform": {
                "options": ["Maltego (commercial)", "Obsidian (notes-based)", "Custom web app"],
                "use_when": "Investigator-facing UI for exploration and link analysis"
            },
            "dashboards": {
                "options": ["Grafana", "Kibana (with Elasticsearch)", "Superset"],
                "use_when": "Monitoring dashboards, operational metrics"
            },
            "reporting": {
                "options": ["Jupyter notebooks", "Pandoc (markdown to PDF)", "Custom templates"],
                "use_when": "Deliverable reports, briefings, evidence packages"
            }
        }
    }
}
```

---

## 27.3 Reference Stack Configurations

### Stack A: Solo Investigator (Low Budget)

```python
SOLO_INVESTIGATOR_STACK = {
    "total_cost_monthly": "$20-50",
    "setup_complexity": "Medium",
    "target_user": "Individual journalist, researcher, PI",

    "components": {
        "collection": {
            "web": "requests + BeautifulSoup (free)",
            "social": "Twitter Academic API (free tier), PRAW for Reddit (free)",
            "documents": "pdfplumber, pytesseract (free, open source)",
            "news": "NewsAPI (free tier: 100 requests/day)"
        },
        "processing": {
            "nlp": "spaCy en_core_web_sm (free)",
            "ai_analysis": "Claude API (pay per use, ~$10-30/month for typical investigation use)",
            "ocr": "pytesseract + OpenCV (free)"
        },
        "storage": {
            "database": "SQLite (free, no server required)",
            "search": "SQLite FTS5 (built-in full-text search)",
            "files": "Local filesystem with encrypted folder (VeraCrypt)"
        },
        "analysis": {
            "graph": "NetworkX (free) + matplotlib visualization",
            "geo": "Folium (free, browser-based maps)",
            "timeline": "Plotly (free tier)"
        },
        "presentation": {
            "notes": "Obsidian (free for local use)",
            "reporting": "Markdown + Pandoc (free)"
        },
        "opsec": {
            "browser": "Firefox with uBlock Origin + container tabs",
            "vpn": "Mullvad VPN ($5/month)",
            "storage_encryption": "VeraCrypt (free)"
        }
    },

    "limitations": [
        "No real-time streaming capability",
        "Limited concurrent collection",
        "Manual triggering of most workflows",
        "Single-machine scale limits"
    ]
}
```

### Stack B: Small Team (Medium Investment)

```python
SMALL_TEAM_STACK = {
    "total_cost_monthly": "$200-800",
    "setup_complexity": "High (requires developer)",
    "target_user": "News organization OSINT team, corporate security team, forensics firm",

    "components": {
        "infrastructure": "Docker Compose on a $40-100/month VPS or dedicated server",
        "collection": {
            "orchestration": "Celery + Redis for scheduled collection",
            "web": "Playwright for dynamic sites, requests for static",
            "social": "Official APIs with managed rate limits",
            "documents": "Apache Tika (enterprise document parsing)"
        },
        "processing": {
            "nlp": "spaCy large model + custom NER models",
            "ai": "Claude API with prompt caching for efficiency",
            "dedup": "Elasticsearch near-duplicate detection"
        },
        "storage": {
            "search": "Elasticsearch (self-hosted, ~$30-50/month VPS)",
            "relational": "PostgreSQL",
            "graph": "Neo4j Community (free, self-hosted)",
            "cache": "Redis"
        },
        "analysis": {
            "graph": "Neo4j Browser + Gephi",
            "geo": "QGIS + Kepler.gl",
            "timeline": "Custom Plotly dashboard"
        },
        "presentation": {
            "investigation_ui": "Maltego (~$1,500/year per seat) or custom web app",
            "dashboards": "Grafana (free) connected to PostgreSQL",
            "reporting": "Jupyter notebooks + nbconvert"
        }
    }
}
```

### Stack C: Enterprise (Full Investment)

```python
ENTERPRISE_STACK = {
    "total_cost_monthly": "$5,000-50,000+",
    "setup_complexity": "Very high (dedicated engineering team)",
    "target_user": "Large financial institution, government agency, enterprise security team",

    "components": {
        "infrastructure": "Kubernetes cluster (AWS EKS / GCP GKE), multi-region",
        "collection": {
            "orchestration": "Apache Airflow for DAG-based workflows",
            "streaming": "Apache Kafka for real-time feeds",
            "scale": "Distributed Scrapy cluster",
            "commercial_data": "Palantir Data Fusion, or TLO, LexisNexis Accurint"
        },
        "processing": {
            "nlp": "Custom fine-tuned models on enterprise GPU cluster",
            "ai": "Claude API (enterprise tier with higher rate limits)",
            "translation": "DeepL API Pro or Google Cloud Translation",
            "scale": "Apache Spark for batch processing"
        },
        "storage": {
            "search": "Elasticsearch cluster (AWS OpenSearch or managed ES)",
            "relational": "PostgreSQL with read replicas",
            "graph": "Neo4j Enterprise or Amazon Neptune",
            "warehouse": "Snowflake or BigQuery for analytics",
            "archive": "AWS Glacier for long-term retention"
        },
        "analysis": {
            "platform": "Palantir Gotham / i2 Analyst's Notebook",
            "custom": "Custom React frontend with Neo4j visualization",
            "ml": "SageMaker or Vertex AI for custom model deployment"
        }
    }
}
```

---

## 27.4 API Key and Credential Management

OSINT stacks accumulate API keys from dozens of services. Poor credential management creates security risk and operational fragility.

```python
import os
from typing import Optional, Dict
from functools import wraps
import time

class APICredentialManager:
    """
    Centralized API credential management with rate limit tracking
    """

    def __init__(self):
        self._credentials: Dict[str, str] = {}
        self._rate_limits: Dict[str, Dict] = {}
        self._request_counts: Dict[str, list] = {}

    def load_from_environment(self) -> None:
        """Load all API credentials from environment variables"""
        api_key_env_vars = {
            'SHODAN_API_KEY': 'shodan',
            'HUNTER_API_KEY': 'hunter',
            'VIRUSTOTAL_API_KEY': 'virustotal',
            'NEWS_API_KEY': 'newsapi',
            'GITHUB_TOKEN': 'github',
            'WHOISXML_KEY': 'whoisxml',
            'ABUSEIPDB_KEY': 'abuseipdb',
            'SECURITYTRAILS_KEY': 'securitytrails',
            'ANTHROPIC_API_KEY': 'anthropic',
            'OPENAI_API_KEY': 'openai',
            'DEEPL_API_KEY': 'deepl',
        }

        for env_var, service_name in api_key_env_vars.items():
            value = os.getenv(env_var)
            if value:
                self._credentials[service_name] = value

    def get_key(self, service: str) -> Optional[str]:
        """Get API key for a service"""
        return self._credentials.get(service)

    def set_rate_limit(self, service: str, requests_per_minute: int, requests_per_day: int = None) -> None:
        """Configure rate limits for a service"""
        self._rate_limits[service] = {
            'per_minute': requests_per_minute,
            'per_day': requests_per_day
        }
        self._request_counts[service] = []

    def can_make_request(self, service: str) -> bool:
        """Check if request is within rate limits"""
        if service not in self._rate_limits:
            return True

        now = time.time()
        limits = self._rate_limits[service]
        counts = self._request_counts.get(service, [])

        # Clean old entries
        counts = [t for t in counts if now - t < 86400]  # Keep last 24h
        self._request_counts[service] = counts

        # Check per-minute limit
        minute_count = sum(1 for t in counts if now - t < 60)
        if minute_count >= limits.get('per_minute', float('inf')):
            return False

        # Check per-day limit
        if limits.get('per_day'):
            day_count = len(counts)
            if day_count >= limits['per_day']:
                return False

        return True

    def record_request(self, service: str) -> None:
        """Record that a request was made"""
        if service not in self._request_counts:
            self._request_counts[service] = []
        self._request_counts[service].append(time.time())

    def wait_if_needed(self, service: str) -> None:
        """Wait until we can make a request within rate limits"""
        while not self.can_make_request(service):
            print(f"Rate limit reached for {service}. Waiting...")
            time.sleep(10)

    def rate_limited(self, service: str):
        """Decorator for rate-limited API calls"""
        def decorator(func):
            @wraps(func)
            def wrapper(*args, **kwargs):
                self.wait_if_needed(service)
                result = func(*args, **kwargs)
                self.record_request(service)
                return result
            return wrapper
        return decorator

    def get_status_report(self) -> Dict:
        """Current rate limit status for all services"""
        now = time.time()
        report = {}

        for service in self._request_counts:
            counts = self._request_counts[service]
            recent_minute = sum(1 for t in counts if now - t < 60)
            recent_day = sum(1 for t in counts if now - t < 86400)
            limits = self._rate_limits.get(service, {})

            report[service] = {
                'requests_last_minute': recent_minute,
                'requests_last_day': recent_day,
                'minute_limit': limits.get('per_minute', 'unlimited'),
                'day_limit': limits.get('per_day', 'unlimited'),
                'key_loaded': service in self._credentials
            }

        return report


# Global credential manager instance
creds = APICredentialManager()
creds.load_from_environment()

# Configure rate limits for common services
creds.set_rate_limit('shodan', requests_per_minute=1, requests_per_day=100)
creds.set_rate_limit('virustotal', requests_per_minute=4, requests_per_day=500)
creds.set_rate_limit('hunter', requests_per_minute=60, requests_per_day=25)
creds.set_rate_limit('newsapi', requests_per_minute=100, requests_per_day=100)
creds.set_rate_limit('securitytrails', requests_per_minute=2, requests_per_day=50)
```

---

## 27.5 Data Model for Pivot-Based Investigation

The pivot-based investigation model (introduced in Chapter 4) requires a data model that supports traversal: given an entity, find everything connected to it, and from those connections, find further connections.

```python
from dataclasses import dataclass, field
from typing import List, Dict, Optional, Set
from datetime import datetime
import uuid

# Core entity types in the OSINT data model
ENTITY_TYPES = {
    # People
    'PERSON': 'person',
    'ORG': 'organization',

    # Digital
    'DOMAIN': 'domain',
    'IP': 'ip_address',
    'EMAIL': 'email',
    'USERNAME': 'username',
    'URL': 'url',
    'PHONE': 'phone',

    # Geographic
    'LOCATION': 'location',
    'ADDRESS': 'address',

    # Financial
    'ACCOUNT': 'account',
    'TRANSACTION': 'transaction',
    'WALLET': 'crypto_wallet',

    # Documents
    'DOCUMENT': 'document',
    'FILING': 'filing',

    # Events
    'EVENT': 'event',
    'INCIDENT': 'incident'
}

RELATIONSHIP_TYPES = {
    'OWNS': 'owns',
    'CONTROLS': 'controls',
    'EMPLOYS': 'employs',
    'REGISTERED_AT': 'registered_at',
    'LOCATED_AT': 'located_at',
    'COMMUNICATED_WITH': 'communicated_with',
    'LINKED_TO': 'linked_to',
    'RESOLVED_TO': 'resolved_to',
    'ASSOCIATED_WITH': 'associated_with',
    'FILED_BY': 'filed_by',
    'PARTICIPATED_IN': 'participated_in',
    'TRANSFERRED_TO': 'transferred_to'
}

@dataclass
class OSINTEntity:
    """Core entity in the OSINT knowledge graph"""
    entity_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    entity_type: str = ''
    value: str = ''
    label: str = ''

    # Provenance
    sources: List[Dict] = field(default_factory=list)
    first_seen: str = field(default_factory=lambda: datetime.now().isoformat())
    last_seen: str = field(default_factory=lambda: datetime.now().isoformat())

    # Confidence
    confidence: float = 0.5  # 0.0 to 1.0
    verified: bool = False

    # Attributes — flexible key-value store for type-specific data
    attributes: Dict = field(default_factory=dict)

    # Tags for categorization
    tags: Set[str] = field(default_factory=set)

    def add_source(self, source_name: str, source_url: str, collected_at: str = None) -> None:
        """Add a source attribution"""
        self.sources.append({
            'source': source_name,
            'url': source_url,
            'collected_at': collected_at or datetime.now().isoformat()
        })
        self.last_seen = datetime.now().isoformat()

    def to_dict(self) -> Dict:
        return {
            'entity_id': self.entity_id,
            'entity_type': self.entity_type,
            'value': self.value,
            'label': self.label,
            'sources': self.sources,
            'first_seen': self.first_seen,
            'last_seen': self.last_seen,
            'confidence': self.confidence,
            'verified': self.verified,
            'attributes': self.attributes,
            'tags': list(self.tags)
        }


@dataclass
class OSINTRelationship:
    """Relationship between two OSINT entities"""
    relationship_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    source_entity_id: str = ''
    target_entity_id: str = ''
    relationship_type: str = ''

    # Evidence
    evidence: List[Dict] = field(default_factory=list)
    sources: List[Dict] = field(default_factory=list)
    confidence: float = 0.5

    # Temporal
    start_date: Optional[str] = None
    end_date: Optional[str] = None
    active: bool = True

    def to_dict(self) -> Dict:
        return {
            'relationship_id': self.relationship_id,
            'source_entity_id': self.source_entity_id,
            'target_entity_id': self.target_entity_id,
            'relationship_type': self.relationship_type,
            'evidence': self.evidence,
            'confidence': self.confidence,
            'start_date': self.start_date,
            'end_date': self.end_date,
            'active': self.active
        }


class OSINTKnowledgeGraph:
    """
    In-memory knowledge graph for OSINT investigation
    For persistence: export to Neo4j or Gephi format
    """

    def __init__(self, investigation_id: str):
        self.investigation_id = investigation_id
        self.entities: Dict[str, OSINTEntity] = {}
        self.relationships: Dict[str, OSINTRelationship] = {}
        self._value_index: Dict[str, str] = {}  # value -> entity_id

    def add_entity(self, entity: OSINTEntity) -> OSINTEntity:
        """Add entity, deduplicating by value"""
        key = f"{entity.entity_type}:{entity.value.lower()}"

        if key in self._value_index:
            # Merge with existing entity
            existing_id = self._value_index[key]
            existing = self.entities[existing_id]
            existing.sources.extend(entity.sources)
            existing.last_seen = datetime.now().isoformat()
            existing.confidence = max(existing.confidence, entity.confidence)
            existing.attributes.update(entity.attributes)
            existing.tags.update(entity.tags)
            return existing

        self.entities[entity.entity_id] = entity
        self._value_index[key] = entity.entity_id
        return entity

    def add_relationship(self, relationship: OSINTRelationship) -> OSINTRelationship:
        """Add relationship between entities"""
        # Verify entities exist
        if relationship.source_entity_id not in self.entities:
            raise ValueError(f"Source entity {relationship.source_entity_id} not found")
        if relationship.target_entity_id not in self.entities:
            raise ValueError(f"Target entity {relationship.target_entity_id} not found")

        self.relationships[relationship.relationship_id] = relationship
        return relationship

    def get_neighbors(self, entity_id: str, relationship_types: List[str] = None) -> List[OSINTEntity]:
        """Get all entities connected to the given entity"""
        neighbor_ids = set()

        for rel in self.relationships.values():
            if relationship_types and rel.relationship_type not in relationship_types:
                continue
            if rel.source_entity_id == entity_id:
                neighbor_ids.add(rel.target_entity_id)
            elif rel.target_entity_id == entity_id:
                neighbor_ids.add(rel.source_entity_id)

        return [self.entities[eid] for eid in neighbor_ids if eid in self.entities]

    def find_by_value(self, value: str, entity_type: str = None) -> Optional[OSINTEntity]:
        """Find entity by value"""
        if entity_type:
            key = f"{entity_type}:{value.lower()}"
            entity_id = self._value_index.get(key)
            return self.entities.get(entity_id) if entity_id else None

        # Search all types
        for entity in self.entities.values():
            if entity.value.lower() == value.lower():
                return entity
        return None

    def pivot_investigation(self, start_value: str, max_hops: int = 3) -> Dict:
        """
        Perform a pivot investigation from a starting entity
        Returns all connected entities within max_hops
        """
        start_entity = self.find_by_value(start_value)
        if not start_entity:
            return {'error': f'Entity not found: {start_value}'}

        visited = {start_entity.entity_id}
        result = {
            'start': start_entity.to_dict(),
            'hops': []
        }

        current_layer = [start_entity.entity_id]

        for hop in range(max_hops):
            next_layer = []
            hop_entities = []

            for entity_id in current_layer:
                neighbors = self.get_neighbors(entity_id)
                for neighbor in neighbors:
                    if neighbor.entity_id not in visited:
                        visited.add(neighbor.entity_id)
                        next_layer.append(neighbor.entity_id)
                        hop_entities.append(neighbor.to_dict())

            if not hop_entities:
                break

            result['hops'].append({
                'hop': hop + 1,
                'entities_count': len(hop_entities),
                'entities': hop_entities
            })

            current_layer = next_layer

        result['total_entities_found'] = len(visited) - 1
        return result

    def export_to_gephi(self, output_path: str) -> None:
        """Export graph to Gephi GEXF format"""
        import xml.etree.ElementTree as ET

        gexf = ET.Element('gexf', {'xmlns': 'http://gexf.net/1.3', 'version': '1.3'})
        graph = ET.SubElement(gexf, 'graph', {'defaultedgetype': 'directed'})

        nodes = ET.SubElement(graph, 'nodes')
        for entity in self.entities.values():
            node = ET.SubElement(nodes, 'node', {
                'id': entity.entity_id,
                'label': entity.label or entity.value[:50]
            })

        edges = ET.SubElement(graph, 'edges')
        for rel in self.relationships.values():
            edge = ET.SubElement(edges, 'edge', {
                'id': rel.relationship_id,
                'source': rel.source_entity_id,
                'target': rel.target_entity_id,
                'label': rel.relationship_type
            })

        tree = ET.ElementTree(gexf)
        tree.write(output_path, encoding='unicode', xml_declaration=True)
        print(f"Exported {len(self.entities)} entities and {len(self.relationships)} relationships to {output_path}")
```

---

## Summary

Building an OSINT stack is an ongoing architectural project, not a one-time implementation. The right architecture for your requirements today may not be the right architecture after a year of scaling, tool maturation, and shifting investigative focus.

Successful OSINT stacks share common characteristics regardless of scale:

**API-first**: Always prefer official APIs over scraping; they're more stable, rate-limited responsibly, and legally clearer.

**Credential hygiene**: Centralize credential management, rotate keys regularly, monitor usage, and never commit credentials to version control.

**Data model clarity**: A well-designed entity and relationship model is the foundation. Retrofitting a data model onto an existing collection is orders of magnitude harder than getting it right initially.

**Modular design**: Each component (collector, processor, enricher, storer) should be independently replaceable. Tool landscapes change; good interfaces make migration possible.

---

## Common Mistakes and Pitfalls

- **Over-engineering early**: Building Kafka + Kubernetes for a two-person team adds months of operational overhead for marginal benefit
- **Hardcoding credentials**: API keys in code or config files committed to git is a common, serious error
- **No rate limit management**: Unmanaged API calls trigger bans and waste budget; build rate limiting from day one
- **Ignoring costs**: Commercial API costs scale with usage; a successful investigation pipeline can generate unexpected bills
- **Single-source dependencies**: Building critical workflows around a single commercial API creates fragility when that API changes pricing or terms

---

## Further Reading

- Twelve-Factor App methodology (12factor.net) — principles for maintainable application architecture
- Elasticsearch documentation — index design and query optimization
- Neo4j graph data modeling guide — designing effective property graphs
- AWS Well-Architected Framework — cloud infrastructure design principles
- Docker and Kubernetes documentation — container orchestration for OSINT infrastructure
