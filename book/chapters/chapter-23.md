# Chapter 23: Enterprise-Scale Analysis and Distributed Data Processing

## Learning Objectives

By the end of this chapter, you will be able to:
- Design and implement distributed OSINT pipelines that scale beyond single-machine limits
- Apply stream processing and batch processing patterns to continuous intelligence collection
- Build production-grade data infrastructure for enterprise OSINT operations
- Implement data deduplication, normalization, and quality control at scale
- Design alerting and monitoring systems for large-scale intelligence feeds
- Understand cost, performance, and operational trade-offs in enterprise deployments

---

## 23.1 The Scale Problem in OSINT

Individual and small-team OSINT workflows operate within comfortable limits: a few hundred sources, response times measured in minutes, data volumes measured in gigabytes. Enterprise OSINT operations — monitoring global supply chains, tracking threat actors across thousands of indicators, processing SEC filings across the entire public markets universe — break these limits.

Scale introduces challenges that good single-machine code doesn't encounter:

**Volume**: Processing terabytes of news articles, social media content, or financial disclosures requires distributed compute.

**Velocity**: Real-time intelligence requirements — threat feeds, breaking news, social media monitoring — demand stream processing rather than batch jobs.

**Variety**: Enterprise OSINT integrates dozens of heterogeneous data sources with incompatible schemas, data formats, and update cadences.

**Veracity**: At scale, data quality problems that seem minor become operationally significant. Duplicate records, incorrect entity matching, stale data, and malformed inputs propagate through downstream analysis.

**Value**: Enterprise deployments justify infrastructure investment that individual practitioners cannot; the challenge is architecting that infrastructure correctly.

---

## 23.2 Distributed Pipeline Architecture

### Architecture Patterns

**Lambda architecture**: Separates processing into batch and speed layers. A batch layer (e.g., Apache Spark) reprocesses complete historical data on a regular schedule; a speed layer (e.g., Apache Kafka + Flink) processes recent data in real-time. Results are merged in a serving layer.

**Kappa architecture**: Processes all data as a stream, eliminating the batch layer. Simpler operationally; requires the stream processing framework to handle historical replay.

**Microservice architecture**: Decomposes the pipeline into independent services (collector, processor, enricher, storer, alerter) that communicate via message queues. Each service scales independently.

For most OSINT operations, a practical architecture combines:
- Apache Kafka for message streaming
- Celery or Apache Airflow for task orchestration
- Elasticsearch for search and aggregation
- PostgreSQL for structured entity data
- Redis for caching and rate limiting

```python
# Enterprise OSINT pipeline architecture
# Using Celery for distributed task processing

from celery import Celery
from celery.utils.log import get_task_logger
from kombu import Queue
import redis
import json
from datetime import datetime
from typing import Dict, List, Optional
import hashlib

# Initialize Celery application
# In production: broker and backend are Redis or RabbitMQ
app = Celery('osint_pipeline')
app.config_from_object({
    'broker_url': 'redis://localhost:6379/0',
    'result_backend': 'redis://localhost:6379/1',
    'task_queues': (
        Queue('high_priority', routing_key='high'),
        Queue('normal', routing_key='normal'),
        Queue('low_priority', routing_key='low'),
    ),
    'task_routes': {
        'pipeline.collect_threat_feed': {'queue': 'high_priority'},
        'pipeline.collect_news': {'queue': 'normal'},
        'pipeline.process_filings': {'queue': 'low_priority'},
    },
    'task_acks_late': True,
    'worker_prefetch_multiplier': 1,
    'task_serializer': 'json',
    'result_serializer': 'json',
    'accept_content': ['json'],
    'timezone': 'UTC',
    'enable_utc': True,
    'beat_schedule': {
        'collect-threat-feeds-hourly': {
            'task': 'pipeline.collect_threat_feed',
            'schedule': 3600.0,
        },
        'collect-news-every-15min': {
            'task': 'pipeline.collect_news',
            'schedule': 900.0,
        },
        'process-sec-filings-daily': {
            'task': 'pipeline.process_filings',
            'schedule': 86400.0,
        },
    },
})

logger = get_task_logger(__name__)

# Redis connection for deduplication and caching
redis_client = redis.Redis(host='localhost', port=6379, db=2, decode_responses=True)


def compute_content_hash(content: str) -> str:
    """Generate content fingerprint for deduplication"""
    return hashlib.sha256(content.encode('utf-8')).hexdigest()[:16]


@app.task(name='pipeline.collect_threat_feed', bind=True, max_retries=3)
def collect_threat_feed(self, feed_config: Dict) -> Dict:
    """
    Collect from a threat intelligence feed
    Idempotent: uses content hashing to skip duplicates
    """
    import requests

    results = {
        'feed': feed_config.get('name'),
        'collected': 0,
        'duplicates_skipped': 0,
        'errors': 0,
        'items': []
    }

    try:
        response = requests.get(
            feed_config['url'],
            headers=feed_config.get('headers', {}),
            timeout=30
        )
        response.raise_for_status()

        # Parse response based on format
        format_type = feed_config.get('format', 'json')

        if format_type == 'json':
            items = response.json() if isinstance(response.json(), list) else [response.json()]
        elif format_type == 'csv':
            import csv, io
            reader = csv.DictReader(io.StringIO(response.text))
            items = list(reader)
        elif format_type == 'text':
            # One IOC per line
            items = [{'value': line.strip()} for line in response.text.split('\n') if line.strip()]
        else:
            items = []

        for item in items:
            # Deduplication check
            item_str = json.dumps(item, sort_keys=True)
            content_hash = compute_content_hash(item_str)
            cache_key = f"seen:{feed_config['name']}:{content_hash}"

            if redis_client.exists(cache_key):
                results['duplicates_skipped'] += 1
                continue

            # Mark as seen (expire after 7 days)
            redis_client.setex(cache_key, 604800, '1')

            # Normalize and enqueue for processing
            normalized_item = {
                'source': feed_config['name'],
                'raw': item,
                'collected_at': datetime.utcnow().isoformat(),
                'content_hash': content_hash
            }

            # Send to processing queue
            process_intelligence_item.apply_async(
                args=[normalized_item],
                queue='normal'
            )

            results['collected'] += 1
            results['items'].append(content_hash)

    except requests.exceptions.RequestException as exc:
        logger.error(f"Feed collection error: {exc}")
        raise self.retry(exc=exc, countdown=60 * (self.request.retries + 1))

    except Exception as exc:
        logger.error(f"Unexpected error: {exc}")
        results['errors'] += 1

    logger.info(f"Feed {feed_config.get('name')}: {results['collected']} new, "
                f"{results['duplicates_skipped']} dupes")
    return results


@app.task(name='pipeline.process_intelligence_item', bind=True)
def process_intelligence_item(self, item: Dict) -> Dict:
    """
    Process a single intelligence item through the enrichment pipeline
    """
    processed = {
        'original': item,
        'entities': [],
        'enrichments': {},
        'stored': False
    }

    try:
        # Step 1: Entity extraction
        raw_content = item.get('raw', {})
        text_content = extract_text_from_item(raw_content)

        if text_content:
            entities = extract_entities(text_content)
            processed['entities'] = entities

        # Step 2: Enrichment based on entity types
        for entity in processed['entities'][:10]:  # Limit enrichment
            entity_type = entity.get('type')
            entity_value = entity.get('value')

            if entity_type == 'IP':
                enrichment = enrich_ip.apply_async(args=[entity_value], queue='normal')
                processed['enrichments'][entity_value] = enrichment.id

            elif entity_type == 'DOMAIN':
                enrichment = enrich_domain.apply_async(args=[entity_value], queue='normal')
                processed['enrichments'][entity_value] = enrichment.id

        # Step 3: Store processed item
        store_intelligence_item.apply_async(
            args=[processed],
            queue='low_priority'
        )
        processed['stored'] = True

    except Exception as exc:
        logger.error(f"Processing error: {exc}")
        processed['error'] = str(exc)

    return processed


def extract_text_from_item(raw: Dict) -> str:
    """Extract text content from various item formats"""
    text_fields = ['description', 'content', 'body', 'text', 'summary', 'title', 'value']
    parts = []
    for field in text_fields:
        if field in raw and raw[field]:
            parts.append(str(raw[field]))
    return ' '.join(parts)


def extract_entities(text: str) -> List[Dict]:
    """Extract OSINT-relevant entities from text"""
    import re
    entities = []

    # IP addresses
    ip_pattern = r'\b(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\b'
    for match in re.finditer(ip_pattern, text):
        ip = match.group()
        # Skip private ranges
        if not (ip.startswith('192.168.') or ip.startswith('10.') or ip.startswith('172.')):
            entities.append({'type': 'IP', 'value': ip, 'context': text[max(0, match.start()-50):match.end()+50]})

    # Domains
    domain_pattern = r'\b(?:[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?\.)+[a-zA-Z]{2,}\b'
    for match in re.finditer(domain_pattern, text):
        domain = match.group().lower()
        if len(domain) > 4 and '.' in domain:
            entities.append({'type': 'DOMAIN', 'value': domain, 'context': text[max(0, match.start()-50):match.end()+50]})

    # File hashes (MD5, SHA1, SHA256)
    md5_pattern = r'\b[a-f0-9]{32}\b'
    sha256_pattern = r'\b[a-f0-9]{64}\b'

    for match in re.finditer(md5_pattern, text, re.IGNORECASE):
        entities.append({'type': 'HASH_MD5', 'value': match.group().lower()})

    for match in re.finditer(sha256_pattern, text, re.IGNORECASE):
        entities.append({'type': 'HASH_SHA256', 'value': match.group().lower()})

    return entities


@app.task(name='pipeline.enrich_ip')
def enrich_ip(ip: str) -> Dict:
    """Enrich an IP address with threat intelligence data"""
    import requests

    enrichment = {'ip': ip, 'sources': {}}

    # Cache check
    cache_key = f"enrich:ip:{ip}"
    cached = redis_client.get(cache_key)
    if cached:
        return json.loads(cached)

    # AbuseIPDB
    try:
        import os
        api_key = os.getenv('ABUSEIPDB_KEY', '')
        if api_key:
            response = requests.get(
                'https://api.abuseipdb.com/api/v2/check',
                headers={'Key': api_key, 'Accept': 'application/json'},
                params={'ipAddress': ip, 'maxAgeInDays': 90},
                timeout=10
            )
            if response.status_code == 200:
                data = response.json().get('data', {})
                enrichment['sources']['abuseipdb'] = {
                    'abuse_confidence_score': data.get('abuseConfidenceScore'),
                    'total_reports': data.get('totalReports'),
                    'country': data.get('countryCode'),
                    'isp': data.get('isp'),
                    'usage_type': data.get('usageType')
                }
    except Exception:
        pass

    # Cache result for 6 hours
    redis_client.setex(cache_key, 21600, json.dumps(enrichment))

    return enrichment


@app.task(name='pipeline.enrich_domain')
def enrich_domain(domain: str) -> Dict:
    """Enrich a domain with WHOIS and DNS data"""
    import requests

    enrichment = {'domain': domain, 'sources': {}}

    cache_key = f"enrich:domain:{domain}"
    cached = redis_client.get(cache_key)
    if cached:
        return json.loads(cached)

    # WhoisXML API or similar
    try:
        import os
        api_key = os.getenv('WHOISXML_KEY', '')
        if api_key:
            response = requests.get(
                'https://www.whoisxmlapi.com/whoisserver/WhoisService',
                params={
                    'apiKey': api_key,
                    'domainName': domain,
                    'outputFormat': 'JSON'
                },
                timeout=10
            )
            if response.status_code == 200:
                data = response.json()
                whois_data = data.get('WhoisRecord', {})
                enrichment['sources']['whoisxml'] = {
                    'registrar': whois_data.get('registrarName'),
                    'created': whois_data.get('createdDate'),
                    'expires': whois_data.get('expiresDate'),
                    'registrant_country': whois_data.get('registrant', {}).get('country')
                }
    except Exception:
        pass

    # Cache result for 24 hours
    redis_client.setex(cache_key, 86400, json.dumps(enrichment))

    return enrichment


@app.task(name='pipeline.store_intelligence_item')
def store_intelligence_item(item: Dict) -> bool:
    """Store processed intelligence item to Elasticsearch"""
    try:
        from elasticsearch import Elasticsearch

        es = Elasticsearch(['http://localhost:9200'])

        # Index document
        index_name = f"osint-{datetime.utcnow().strftime('%Y-%m')}"
        es.index(
            index=index_name,
            document={
                **item,
                '@timestamp': datetime.utcnow().isoformat()
            }
        )
        return True

    except ImportError:
        # Elasticsearch not available — log to file as fallback
        with open('/tmp/osint_items.jsonl', 'a') as f:
            f.write(json.dumps(item) + '\n')
        return True

    except Exception as e:
        logger.error(f"Storage error: {e}")
        return False
```

---

## 23.3 Elasticsearch for OSINT Search

Elasticsearch provides the core search and aggregation capability for enterprise OSINT. Its inverted index, full-text search, and aggregation pipeline handle millions of documents with sub-second query performance.

```python
from elasticsearch import Elasticsearch
from elasticsearch.helpers import bulk
from datetime import datetime, timedelta
from typing import List, Dict, Generator
import json

class OSINTElasticsearch:
    """
    Elasticsearch interface for enterprise OSINT data store
    """

    INDEX_MAPPING = {
        "mappings": {
            "properties": {
                "@timestamp": {"type": "date"},
                "source": {"type": "keyword"},
                "content_hash": {"type": "keyword"},
                "text_content": {
                    "type": "text",
                    "analyzer": "english",
                    "fields": {
                        "keyword": {"type": "keyword", "ignore_above": 256}
                    }
                },
                "entities": {
                    "type": "nested",
                    "properties": {
                        "type": {"type": "keyword"},
                        "value": {"type": "keyword"},
                        "context": {"type": "text"}
                    }
                },
                "geo": {"type": "geo_point"},
                "risk_score": {"type": "float"},
                "tags": {"type": "keyword"},
                "enrichments": {"type": "object", "dynamic": True}
            }
        },
        "settings": {
            "number_of_shards": 3,
            "number_of_replicas": 1,
            "analysis": {
                "analyzer": {
                    "osint_analyzer": {
                        "type": "custom",
                        "tokenizer": "standard",
                        "filter": ["lowercase", "stop", "snowball"]
                    }
                }
            }
        }
    }

    def __init__(self, hosts: List[str] = None):
        self.es = Elasticsearch(hosts or ['http://localhost:9200'])
        self.base_index = 'osint'

    def ensure_index(self, index_name: str) -> None:
        """Create index with proper mapping if it doesn't exist"""
        if not self.es.indices.exists(index=index_name):
            self.es.indices.create(
                index=index_name,
                body=self.INDEX_MAPPING
            )

    def bulk_index(self, documents: List[Dict], index_suffix: str = None) -> Dict:
        """Bulk index documents for efficiency"""
        index_name = f"{self.base_index}-{index_suffix or datetime.utcnow().strftime('%Y-%m')}"
        self.ensure_index(index_name)

        def generate_actions(docs: List[Dict]) -> Generator:
            for doc in docs:
                yield {
                    "_index": index_name,
                    "_id": doc.get('content_hash'),  # Use hash as ID for idempotency
                    "_source": {
                        **doc,
                        "@timestamp": doc.get('collected_at', datetime.utcnow().isoformat())
                    }
                }

        success, failed = bulk(
            self.es,
            generate_actions(documents),
            raise_on_error=False,
            stats_only=True
        )

        return {'indexed': success, 'failed': failed}

    def search_entities(self, entity_type: str, entity_value: str,
                       days_back: int = 30) -> Dict:
        """Search for a specific entity across all indices"""
        query = {
            "query": {
                "bool": {
                    "must": [
                        {
                            "nested": {
                                "path": "entities",
                                "query": {
                                    "bool": {
                                        "must": [
                                            {"term": {"entities.type": entity_type}},
                                            {"term": {"entities.value": entity_value.lower()}}
                                        ]
                                    }
                                }
                            }
                        },
                        {
                            "range": {
                                "@timestamp": {
                                    "gte": f"now-{days_back}d"
                                }
                            }
                        }
                    ]
                }
            },
            "sort": [{"@timestamp": {"order": "desc"}}],
            "size": 100,
            "aggs": {
                "by_source": {
                    "terms": {"field": "source", "size": 20}
                },
                "timeline": {
                    "date_histogram": {
                        "field": "@timestamp",
                        "calendar_interval": "day"
                    }
                }
            }
        }

        return self.es.search(index=f"{self.base_index}-*", body=query)

    def threat_landscape_summary(self, days_back: int = 7) -> Dict:
        """
        Aggregate query providing threat landscape overview
        """
        query = {
            "query": {
                "range": {
                    "@timestamp": {"gte": f"now-{days_back}d"}
                }
            },
            "size": 0,
            "aggs": {
                "by_source": {
                    "terms": {"field": "source", "size": 20}
                },
                "entity_types": {
                    "nested": {"path": "entities"},
                    "aggs": {
                        "types": {
                            "terms": {"field": "entities.type", "size": 10}
                        }
                    }
                },
                "top_iocs": {
                    "nested": {"path": "entities"},
                    "aggs": {
                        "top_values": {
                            "terms": {
                                "field": "entities.value",
                                "size": 20,
                                "order": {"_count": "desc"}
                            }
                        }
                    }
                },
                "volume_over_time": {
                    "date_histogram": {
                        "field": "@timestamp",
                        "calendar_interval": "hour"
                    }
                }
            }
        }

        return self.es.search(index=f"{self.base_index}-*", body=query)

    def find_related_entities(self, entity_value: str, hops: int = 2) -> List[Dict]:
        """
        Find entities that co-appear with a given entity
        Implements a basic graph traversal using co-occurrence
        """
        related = []
        seen = {entity_value}

        current_layer = [entity_value]

        for hop in range(hops):
            next_layer = []

            for entity in current_layer:
                # Find documents containing this entity
                result = self.es.search(
                    index=f"{self.base_index}-*",
                    body={
                        "query": {
                            "nested": {
                                "path": "entities",
                                "query": {"term": {"entities.value": entity}}
                            }
                        },
                        "size": 20,
                        "_source": ["entities", "source", "@timestamp"]
                    }
                )

                for hit in result.get('hits', {}).get('hits', []):
                    for entity_obj in hit.get('_source', {}).get('entities', []):
                        related_value = entity_obj.get('value', '')
                        if related_value not in seen:
                            seen.add(related_value)
                            next_layer.append(related_value)
                            related.append({
                                'value': related_value,
                                'type': entity_obj.get('type'),
                                'hop': hop + 1,
                                'co_appeared_with': entity,
                                'source': hit.get('_source', {}).get('source')
                            })

            current_layer = next_layer[:20]  # Limit expansion

        return related
```

---

## 23.4 Stream Processing with Kafka

For near-real-time intelligence — social media monitoring, threat feeds, breaking news — batch processing is too slow. Kafka provides the message streaming backbone.

```python
# Kafka consumer for real-time OSINT stream processing
# Requires: pip install confluent-kafka

from confluent_kafka import Consumer, Producer, KafkaException
import json
import threading
from datetime import datetime
from typing import Callable, Dict, List

class OSINTStreamProcessor:
    """
    Real-time OSINT stream processor using Apache Kafka
    """

    def __init__(self, bootstrap_servers: str, group_id: str):
        self.consumer_config = {
            'bootstrap.servers': bootstrap_servers,
            'group.id': group_id,
            'auto.offset.reset': 'latest',
            'enable.auto.commit': True,
            'auto.commit.interval.ms': 5000,
            'session.timeout.ms': 30000,
            'max.poll.interval.ms': 300000,
        }

        self.producer_config = {
            'bootstrap.servers': bootstrap_servers,
            'acks': 'all',
            'retries': 3,
            'batch.size': 16384,
            'linger.ms': 10,
        }

        self.consumer = None
        self.producer = Producer(self.producer_config)
        self.running = False
        self.processors: Dict[str, List[Callable]] = {}

    def register_processor(self, topic: str, processor: Callable) -> None:
        """Register a processing function for a specific topic"""
        if topic not in self.processors:
            self.processors[topic] = []
        self.processors[topic].append(processor)

    def publish(self, topic: str, message: Dict, key: str = None) -> None:
        """Publish a message to a Kafka topic"""
        try:
            self.producer.produce(
                topic=topic,
                value=json.dumps(message).encode('utf-8'),
                key=key.encode('utf-8') if key else None,
                callback=self._delivery_callback
            )
            self.producer.poll(0)
        except Exception as e:
            print(f"Publish error: {e}")

    @staticmethod
    def _delivery_callback(err, msg):
        if err:
            print(f"Message delivery failed: {err}")

    def start_consuming(self, topics: List[str]) -> None:
        """Start consuming messages from specified topics"""
        self.consumer = Consumer(self.consumer_config)
        self.consumer.subscribe(topics)
        self.running = True

        print(f"Starting consumer for topics: {topics}")

        try:
            while self.running:
                msg = self.consumer.poll(timeout=1.0)

                if msg is None:
                    continue

                if msg.error():
                    if msg.error().code() == KafkaException._PARTITION_EOF:
                        continue
                    else:
                        print(f"Consumer error: {msg.error()}")
                        break

                # Process message
                try:
                    topic = msg.topic()
                    value = json.loads(msg.value().decode('utf-8'))

                    if topic in self.processors:
                        for processor in self.processors[topic]:
                            try:
                                result = processor(value)
                                if result:
                                    # Route result to output topic
                                    output_topic = f"{topic}-processed"
                                    self.publish(output_topic, result)
                            except Exception as e:
                                print(f"Processor error on {topic}: {e}")

                except json.JSONDecodeError as e:
                    print(f"JSON decode error: {e}")

        finally:
            self.consumer.close()

    def stop(self) -> None:
        """Gracefully stop the consumer"""
        self.running = False


# Example processing functions
def process_news_item(item: Dict) -> Dict:
    """Process a news article for entity extraction"""
    from pipeline import extract_entities, extract_text_from_item

    text = extract_text_from_item(item.get('raw', item))
    entities = extract_entities(text)

    return {
        'original': item,
        'entities': entities,
        'processed_at': datetime.utcnow().isoformat(),
        'entity_count': len(entities)
    }


def alert_on_high_risk(item: Dict) -> None:
    """Alert when high-risk content is detected"""
    risk_keywords = ['critical vulnerability', 'zero-day', 'active exploitation',
                     'ransomware campaign', 'nation-state', 'APT']

    content = json.dumps(item).lower()
    alerts = [kw for kw in risk_keywords if kw.lower() in content]

    if alerts:
        print(f"🚨 HIGH RISK ALERT: {alerts}")
        # In production: send to Slack, PagerDuty, email
```

---

## 23.5 Data Quality and Deduplication at Scale

Data quality failures at scale are expensive. A 1% duplicate rate across 1 million documents means 10,000 wasted processing cycles and polluted analysis.

```python
import hashlib
import re
from typing import Optional, Dict, List
from dataclasses import dataclass
from datetime import datetime

@dataclass
class QualityReport:
    """Data quality assessment report"""
    total_records: int = 0
    duplicates_detected: int = 0
    malformed_records: int = 0
    empty_records: int = 0
    validated_records: int = 0

    @property
    def quality_score(self) -> float:
        if self.total_records == 0:
            return 0.0
        return (self.validated_records / self.total_records) * 100


class OSINTDataQualityPipeline:
    """
    Data quality enforcement for enterprise OSINT pipelines
    """

    def __init__(self):
        self.seen_hashes: set = set()  # In production: use Redis or Bloom filter

    def compute_fingerprint(self, item: Dict) -> str:
        """
        Compute content fingerprint for deduplication
        Normalizes text before hashing to catch near-duplicates
        """
        text_fields = ['title', 'content', 'body', 'description', 'text']
        text_parts = []

        for field in text_fields:
            if field in item and item[field]:
                # Normalize: lowercase, collapse whitespace, strip punctuation
                text = str(item[field]).lower()
                text = re.sub(r'\s+', ' ', text)
                text = re.sub(r'[^\w\s]', '', text)
                text_parts.append(text[:500])  # First 500 chars of each field

        fingerprint_content = '|'.join(sorted(text_parts))
        return hashlib.sha256(fingerprint_content.encode()).hexdigest()

    def simhash_fingerprint(self, text: str, bit_size: int = 64) -> int:
        """
        SimHash for fuzzy deduplication — catches near-duplicates with minor edits
        Useful for news articles that are slight rewrites of each other
        """
        tokens = text.lower().split()
        vector = [0] * bit_size

        for token in tokens:
            token_hash = int(hashlib.md5(token.encode()).hexdigest(), 16)
            for i in range(bit_size):
                if token_hash & (1 << i):
                    vector[i] += 1
                else:
                    vector[i] -= 1

        fingerprint = 0
        for i in range(bit_size):
            if vector[i] > 0:
                fingerprint |= (1 << i)

        return fingerprint

    def hamming_distance(self, hash1: int, hash2: int) -> int:
        """Compute bit difference between two SimHash values"""
        xor = hash1 ^ hash2
        return bin(xor).count('1')

    def is_near_duplicate(self, text: str, threshold: int = 5) -> bool:
        """
        Check if text is a near-duplicate of something seen before
        threshold: maximum Hamming distance to consider duplicate
        """
        if not hasattr(self, 'simhashes'):
            self.simhashes = []

        current_hash = self.simhash_fingerprint(text)

        for seen_hash in self.simhashes[-10000:]:  # Check recent hashes
            if self.hamming_distance(current_hash, seen_hash) <= threshold:
                return True

        self.simhashes.append(current_hash)
        return False

    def validate_item(self, item: Dict) -> Dict:
        """
        Validate and normalize an OSINT item
        Returns item with quality flags added
        """
        quality = {
            'is_valid': True,
            'issues': [],
            'is_duplicate': False,
            'fingerprint': None
        }

        # Check required fields
        required_fields = ['source']
        for field in required_fields:
            if not item.get(field):
                quality['issues'].append(f"Missing required field: {field}")
                quality['is_valid'] = False

        # Check content
        text_content = self._extract_text(item)
        if not text_content or len(text_content.strip()) < 10:
            quality['issues'].append("Empty or minimal content")
            quality['is_valid'] = False

        # Deduplication check
        if quality['is_valid']:
            fingerprint = self.compute_fingerprint(item)
            quality['fingerprint'] = fingerprint

            if fingerprint in self.seen_hashes:
                quality['is_duplicate'] = True
                quality['issues'].append("Exact duplicate detected")
            else:
                self.seen_hashes.add(fingerprint)

            # Near-duplicate check
            if not quality['is_duplicate'] and text_content:
                if self.is_near_duplicate(text_content):
                    quality['is_duplicate'] = True
                    quality['issues'].append("Near-duplicate detected")

        item['_quality'] = quality
        return item

    def process_batch(self, items: List[Dict]) -> QualityReport:
        """Process a batch of items through quality pipeline"""
        report = QualityReport(total_records=len(items))
        validated = []

        for item in items:
            validated_item = self.validate_item(item)
            quality = validated_item.get('_quality', {})

            if quality.get('is_duplicate'):
                report.duplicates_detected += 1
            elif not quality.get('is_valid'):
                if not self._extract_text(item):
                    report.empty_records += 1
                else:
                    report.malformed_records += 1
            else:
                report.validated_records += 1
                validated.append(validated_item)

        return report

    def _extract_text(self, item: Dict) -> str:
        """Extract text content from item"""
        for field in ['content', 'body', 'text', 'description', 'title']:
            if item.get(field):
                return str(item[field])
        return ''
```

---

## 23.6 Monitoring and Alerting Infrastructure

Enterprise OSINT operations require visibility into pipeline health — are collectors running? Are feeds producing data? Are anomaly rates elevated?

```python
import time
from collections import defaultdict
from datetime import datetime, timedelta
from typing import Dict, Callable
import threading

class PipelineMonitor:
    """
    Real-time monitoring for OSINT pipeline health
    """

    def __init__(self):
        self.metrics: Dict[str, list] = defaultdict(list)
        self.alert_thresholds: Dict[str, float] = {}
        self.alert_handlers: list = []
        self._lock = threading.Lock()

    def record_metric(self, metric_name: str, value: float, tags: Dict = None) -> None:
        """Record a metric data point"""
        with self._lock:
            self.metrics[metric_name].append({
                'value': value,
                'timestamp': time.time(),
                'tags': tags or {}
            })
            # Keep only last 1000 points per metric
            if len(self.metrics[metric_name]) > 1000:
                self.metrics[metric_name] = self.metrics[metric_name][-1000:]

        # Check alert thresholds
        if metric_name in self.alert_thresholds:
            threshold = self.alert_thresholds[metric_name]
            if value > threshold:
                self._trigger_alert(metric_name, value, threshold, tags)

    def set_threshold(self, metric_name: str, threshold: float) -> None:
        """Set alert threshold for a metric"""
        self.alert_thresholds[metric_name] = threshold

    def add_alert_handler(self, handler: Callable) -> None:
        """Register a function to handle alerts"""
        self.alert_handlers.append(handler)

    def _trigger_alert(self, metric: str, value: float, threshold: float, tags: Dict) -> None:
        """Trigger alert handlers"""
        alert = {
            'metric': metric,
            'value': value,
            'threshold': threshold,
            'tags': tags,
            'timestamp': datetime.utcnow().isoformat()
        }
        for handler in self.alert_handlers:
            try:
                handler(alert)
            except Exception as e:
                print(f"Alert handler error: {e}")

    def get_rate(self, metric_name: str, window_seconds: int = 60) -> float:
        """Calculate metric rate over a time window"""
        cutoff = time.time() - window_seconds
        with self._lock:
            recent = [m for m in self.metrics.get(metric_name, [])
                     if m['timestamp'] > cutoff]
        if not recent:
            return 0.0
        return len(recent) / (window_seconds / 60)  # Per minute

    def get_summary(self) -> Dict:
        """Get current monitoring summary"""
        now = time.time()
        summary = {}

        with self._lock:
            for metric_name, points in self.metrics.items():
                if not points:
                    continue
                recent_1h = [p for p in points if now - p['timestamp'] < 3600]
                if recent_1h:
                    values = [p['value'] for p in recent_1h]
                    summary[metric_name] = {
                        'count_1h': len(recent_1h),
                        'avg_1h': sum(values) / len(values),
                        'max_1h': max(values),
                        'last_value': points[-1]['value'],
                        'last_timestamp': datetime.fromtimestamp(points[-1]['timestamp']).isoformat()
                    }

        return summary


# Slack alert handler example
def slack_alert_handler(alert: Dict) -> None:
    """Send alert to Slack webhook"""
    import requests, os

    webhook_url = os.getenv('SLACK_WEBHOOK_URL', '')
    if not webhook_url:
        return

    message = {
        "text": f"⚠️ OSINT Pipeline Alert",
        "attachments": [{
            "color": "danger",
            "fields": [
                {"title": "Metric", "value": alert['metric'], "short": True},
                {"title": "Value", "value": f"{alert['value']:.2f}", "short": True},
                {"title": "Threshold", "value": f"{alert['threshold']:.2f}", "short": True},
                {"title": "Time", "value": alert['timestamp'], "short": True}
            ]
        }]
    }

    try:
        requests.post(webhook_url, json=message, timeout=5)
    except Exception:
        pass
```

---

## Summary

Enterprise-scale OSINT requires architectural thinking that individual workflows don't need. The transition from single-machine scripts to distributed systems introduces complexity in deployment, operations, and debugging that must be planned for rather than discovered under production load.

Key architectural principles:

**Design for idempotency**: Every collection and processing operation should be safe to retry. Use content hashing to eliminate duplicates at ingestion, not downstream.

**Separate concerns**: Collection, processing, enrichment, storage, and alerting are distinct concerns. Separating them enables independent scaling and failure isolation.

**Cache aggressively**: External API calls are rate-limited and slow. Cache enrichment results (IP lookups, WHOIS data) in Redis for hours or days — the data doesn't change that fast.

**Monitor everything**: A pipeline that fails silently is worse than one that fails loudly. Instrument collection rates, processing latency, error rates, and alert threshold breaches.

---

## Common Mistakes and Pitfalls

- **Premature optimization**: Most OSINT operations don't need distributed infrastructure — start with well-written single-threaded code and scale when the data demands it
- **Missing backpressure**: Collectors that produce faster than consumers can process will exhaust memory; implement explicit backpressure mechanisms
- **Ignoring exactly-once delivery**: At-least-once delivery (the Kafka default) means processors must be idempotent or duplicates will corrupt analysis
- **No disaster recovery plan**: What happens if Elasticsearch goes down? If Kafka loses messages? Plan for partial failures
- **Cost blindness**: Enterprise cloud infrastructure for Kafka + Elasticsearch + distributed compute is expensive — model costs before committing to an architecture

---

## Further Reading

- Apache Kafka documentation — distributed streaming platform concepts
- Elasticsearch Guide — index management, query DSL, aggregations
- Celery documentation — distributed task queues for Python
- Designing Data-Intensive Applications (Kleppmann) — foundational distributed systems concepts
- The SRE Book (Google) — production operations and monitoring principles
