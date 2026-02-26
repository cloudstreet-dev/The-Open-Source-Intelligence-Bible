# Chapter 9: Advanced Search Techniques and Historical Data Recovery

## Learning Objectives

By the end of this chapter, you will be able to:
- Use advanced search operators across multiple search engines to surface obscure data
- Recover deleted, modified, and archived content from multiple sources
- Apply Google dorking and advanced search syntax to investigative research
- Use the Wayback Machine and alternative archives strategically
- Extract cached content from search engines and CDNs
- Build systematic search workflows that cover the full content availability spectrum

---

## 9.1 The Gap Between Search and Discovery

The default search experience — type a query, receive ranked results — is optimized for general information retrieval, not investigative discovery. It is designed to surface popular, recent, high-authority content. For investigators, the most valuable information is often:

- Obscure rather than popular
- Specific rather than general
- Old rather than recent
- Deliberately de-indexed rather than highly ranked

Bridging the gap between surface search and investigative discovery requires mastering advanced search operators, alternative search engines, archival resources, and systematic search methodology.

---

## 9.2 Google Advanced Search Operators

Google's search operators provide significant control over search scope and results. Many investigators use Google without knowing this syntax exists; those who do use it have dramatically higher research effectiveness.

### Core Search Operators

```
# Exact phrase search
"John Smith" "AcmeCorp" "San Francisco"

# Site-specific search
site:linkedin.com "John Smith" "AcmeCorp"

# File type filtering
filetype:pdf "AcmeCorp annual report" 2023
filetype:xls "employee" site:acmecorp.com

# URL pattern search
inurl:admin site:targetdomain.com
inurl:backup site:targetdomain.com
inurl:login site:targetdomain.com

# Title search
intitle:"John Smith" "AcmeCorp"

# Text in page body
intext:"confidential" "AcmeCorp" filetype:pdf

# Date range restriction
"John Smith" "AcmeCorp" after:2022-01-01 before:2023-12-31

# Exclusion
"John Smith" -linkedin.com -facebook.com

# OR operator
"John Smith" (AcmeCorp OR "Acme Corporation" OR "Acme Corp")

# AROUND operator (terms appearing near each other)
"John Smith" AROUND(5) "AcmeCorp"

# Cache lookup
cache:targetdomain.com/specific-page

# Related sites
related:targetdomain.com
```

### Advanced Investigation Patterns

**Finding exposed directories and files**:
```
# Exposed directory listings
site:targetdomain.com intitle:"Index of /"

# Exposed configuration files
site:targetdomain.com filetype:env OR filetype:config OR filetype:xml

# Exposed databases
site:targetdomain.com filetype:sql
site:targetdomain.com filetype:db

# Exposed backup files
site:targetdomain.com filetype:bak
site:targetdomain.com inurl:backup

# Exposed credential files
site:targetdomain.com "password" filetype:txt
site:targetdomain.com inurl:passwd
```

**Social media deep search**:
```
# Twitter/X deep search
site:twitter.com "targetusername" "keyword"
site:x.com "target phrase" from:username

# LinkedIn profile discovery
site:linkedin.com/in "full name" "company name"

# Facebook search (limited)
site:facebook.com/posts "keyword"

# Finding profiles across platforms
"real name" site:reddit.com
"real name" (site:instagram.com OR site:tiktok.com)
```

**Document and records discovery**:
```
# Academic publications
site:scholar.google.com "author name" "topic"
site:researchgate.net "researcher name"

# Government documents
site:.gov "person name" "topic" filetype:pdf
site:.mil "technical term" filetype:pdf

# Court documents
site:courtlistener.com "person or company name"
site:pacermonitor.com "person or company name"

# News archives
site:nytimes.com "person name" before:2010-01-01
```

---

## 9.3 Beyond Google: Alternative Search Engines

Different search engines index different portions of the web with different priorities. Relying exclusively on Google misses significant content.

### Bing

Bing indexes different content than Google and often ranks different results for the same query. For investigative research:

- Bing sometimes surfaces older content that Google has de-ranked
- Different spam filtering means some content appears on Bing but not Google
- Bing's Image Search has different coverage than Google's

### DuckDuckGo

DuckDuckGo's lack of personalization makes it valuable for unbiased search:
- No filter bubble effects
- Different freshness/relevance weighting than Google
- Includes indexed content from The Wayback Machine

### Yandex

Russian-language and regional coverage makes Yandex particularly valuable for:
- Eastern European content
- Russian-language content
- Reverse image search — Yandex often performs better than Google for face matching and similar-image discovery

### Baidu

Mandarin-language content and China-focused coverage makes Baidu essential for:
- Investigations with Chinese dimensions
- Chinese company research
- Content not indexed in Western search engines

### Specialized Search Engines

**Carrot2**: Clusters search results by topic, useful for quickly understanding the landscape around a search query

**Boardreader**: Forum and discussion board search

**Pipl** (requires subscription): People search aggregating deep web content

**Spokeo/BeenVerified/Intelius**: People search with commercial data

**Ahmia** (for Tor-accessible public content): Dark web search limited to public/legal content

---

## 9.4 The Wayback Machine and Web Archiving

The Internet Archive's Wayback Machine (web.archive.org) is one of the most valuable OSINT resources in existence. It has been crawling and archiving the web since 1996 and contains over 800 billion pages.

### Strategic Wayback Machine Use

**Finding deleted content**:
When a page has been removed, first check if it was archived:
1. Navigate to web.archive.org
2. Enter the URL
3. Review the calendar view for crawl dates
4. Access the archived version closest to when the content was likely present

**Timeline reconstruction**:
The Wayback Machine enables tracking of how a website evolved over time:
- How has a company's "About" or "Team" page changed?
- What was on a domain before its current ownership?
- When was a piece of content first published?

**Common investigation scenarios**:
- Executive's LinkedIn-style biography that was later scrubbed
- Company press release deleted after fraud was discovered
- Social media posts archived before platform deletion
- Domain content from before a company changed hands

```python
import requests
from datetime import datetime

class WaybackMachineClient:
    """Internet Archive Wayback Machine API client"""

    API_BASE = "https://archive.org/wayback/available"
    CDX_API = "https://web.archive.org/cdx/search/cdx"

    def get_closest_snapshot(self, url, timestamp=None):
        """Get the archived version of a URL closest to a given timestamp"""
        params = {'url': url}
        if timestamp:
            params['timestamp'] = timestamp

        response = requests.get(self.API_BASE, params=params)
        if response.status_code == 200:
            data = response.json()
            if data.get('archived_snapshots', {}).get('closest'):
                return data['archived_snapshots']['closest']
        return None

    def get_all_snapshots(self, url, from_date=None, to_date=None,
                          mime_type=None, status_code=None, limit=100):
        """Get all archived snapshots for a URL"""
        params = {
            'url': url,
            'output': 'json',
            'fl': 'timestamp,original,statuscode,mimetype,length',
            'limit': limit,
        }

        if from_date:
            params['from'] = from_date  # Format: YYYYMMDD
        if to_date:
            params['to'] = to_date
        if mime_type:
            params['filter'] = f'mimetype:{mime_type}'
        if status_code:
            params['filter'] = f'statuscode:{status_code}'

        response = requests.get(self.CDX_API, params=params)
        if response.status_code == 200:
            lines = response.text.strip().split('\n')
            results = []
            for line in lines[1:]:  # Skip header
                if line:
                    parts = line.strip('[]').split(',')
                    if len(parts) >= 5:
                        results.append({
                            'timestamp': parts[0].strip('"'),
                            'original_url': parts[1].strip('"'),
                            'status_code': parts[2].strip('"'),
                            'mime_type': parts[3].strip('"'),
                            'length': parts[4].strip('"'),
                            'archive_url': f"https://web.archive.org/web/{parts[0].strip()}/{parts[1].strip('\"')}"
                        })
            return results
        return []

    def save_page_now(self, url):
        """Save a page to the Wayback Machine right now"""
        response = requests.get(
            f"https://web.archive.org/save/{url}",
            allow_redirects=False
        )

        if response.status_code == 302:
            archive_url = response.headers.get('Location', '')
            return {'status': 'saved', 'archive_url': archive_url}
        return {'status': 'failed', 'status_code': response.status_code}

    def get_domain_history(self, domain):
        """Get all archived pages for an entire domain"""
        params = {
            'url': f"{domain}/*",
            'output': 'json',
            'fl': 'timestamp,original,statuscode',
            'collapse': 'urlkey',
            'limit': 10000,
        }

        response = requests.get(self.CDX_API, params=params)
        if response.status_code == 200:
            lines = response.text.strip().split('\n')
            return [line for line in lines[1:] if line]
        return []

    def find_deleted_subpages(self, domain, keyword=None):
        """Find pages that existed on a domain but no longer do"""
        # Get all archived pages
        all_archived = self.get_domain_history(domain)

        # Check which currently 404
        deleted = []
        for entry in all_archived[:100]:  # Sample check, full check would be expensive
            try:
                parts = entry.strip('[]').split(',')
                if len(parts) >= 2:
                    url = parts[1].strip('"')
                    response = requests.get(url, timeout=5)
                    if response.status_code == 404:
                        deleted.append({'url': url, 'archived_entry': entry})
            except Exception:
                continue

        return deleted

# Usage
wayback = WaybackMachineClient()

# Find archived versions of a specific URL
snapshots = wayback.get_all_snapshots(
    'https://example.com/about',
    from_date='20180101',
    to_date='20231231'
)

# Save current state before it changes
save_result = wayback.save_page_now('https://example.com/press-release')
```

### Alternative Web Archives

The Wayback Machine is not the only archive:

**Cachedview.nl**: Provides access to Google, Bing, and Wayback Machine caches from a single interface.

**CachedPages.com**: Similar multi-cache access.

**Archive.today (archive.ph)**: An alternative web archive with different crawling patterns than the Wayback Machine. Particularly valuable because it captures pages that block the Wayback Machine crawler.

**Common Crawl**: A non-profit that releases raw web crawl data. Used primarily by researchers, but the WARC files are publicly downloadable.

**National Web Archives**: Many national libraries maintain web archives of their country's web. The Library of Congress, British Library, Bibliothèque nationale de France, and others maintain country-specific archives.

---

## 9.5 Search Engine Cache

Google, Bing, and other search engines maintain cached copies of pages they have indexed. Cache is often more recent than the Wayback Machine and provides an alternative to deleted content.

**Accessing cached versions**:
```
# Google cache
cache:example.com/specific-page

# Via URL
https://webcache.googleusercontent.com/search?q=cache:https://example.com/page

# Bing cache (via search results)
# Click the dropdown arrow next to a search result and select "Cached"
```

**Important limitations**:
- Caches are short-lived — Google typically retains cache for days to weeks
- Not all pages are cached
- Google has been progressively reducing cache availability

---

## 9.6 Paste Sites and Leak Repositories

Paste sites — text-sharing services like Pastebin — are frequently used to share leaked data, and their content is indexed by specialized search engines.

### Paste Site Search

**Pastebin Search** (limited, increasingly restricted)

**Intelligence X** (intelx.io): Specialized search engine that indexes paste sites, dark web, and other sources. Paid, with limited free access.

**DeHashed**: Credential leak search engine. Legal use requires documented legitimate purpose.

**GHunt / LeakLooker**: Community-maintained breach data search tools.

**Searching across paste sites**:
```python
import requests

def search_pastebin_google(query):
    """Use Google to search pastebin content (limited effectiveness)"""
    results = []

    # Google search limited to paste sites
    sites = [
        'pastebin.com',
        'ghostbin.com',
        'paste.ee',
        'dpaste.com',
        'hastebin.com',
    ]

    for site in sites:
        google_query = f'site:{site} "{query}"'
        # Note: programmatic Google search requires API; manual execution via browser
        results.append({'site': site, 'query': google_query})

    return results

def intelligence_x_search(query, api_key, buckets=None):
    """Search Intelligence X for a query term"""

    # Step 1: Initiate search
    search_response = requests.post(
        'https://2.intelx.io/intelligent/search',
        headers={'x-key': api_key, 'Content-Type': 'application/json'},
        json={
            'term': query,
            'buckets': buckets or [],  # e.g., ['pastes', 'darkweb']
            'lookuplevel': 0,
            'maxresults': 100,
            'timeout': 5,
        }
    )

    if search_response.status_code != 200:
        return []

    search_id = search_response.json().get('id')

    # Step 2: Retrieve results
    results_response = requests.get(
        f'https://2.intelx.io/intelligent/search/result',
        headers={'x-key': api_key},
        params={'id': search_id, 'format': 1}
    )

    if results_response.status_code == 200:
        return results_response.json().get('records', [])
    return []
```

---

## 9.7 Recovering Document Metadata

Documents recovered during OSINT investigations often contain valuable metadata that is invisible in the document content:

**Author information**: Document author name (often revealing when a company claims ignorance of a document's origin)

**Organization**: The registered organization name of the software

**Software used**: Reveals technology stack

**Edit history**: Revision count, total editing time, previous authors

**Creation and modification dates**: Can contradict claimed timeline

**Embedded path information**: Network paths embedded in older Office documents can reveal internal server names

```python
import subprocess
import json
from pathlib import Path

def extract_document_metadata(file_path):
    """Extract all metadata from a document using ExifTool"""
    try:
        result = subprocess.run(
            ['exiftool', '-json', '-all', str(file_path)],
            capture_output=True,
            text=True,
            timeout=30
        )

        if result.returncode == 0:
            data = json.loads(result.stdout)
            if data:
                metadata = data[0]
                # Focus on investigatively relevant fields
                relevant = {
                    'file_name': metadata.get('FileName'),
                    'file_type': metadata.get('FileType'),
                    'create_date': metadata.get('CreateDate'),
                    'modify_date': metadata.get('ModifyDate'),
                    'author': metadata.get('Author'),
                    'creator': metadata.get('Creator'),
                    'last_modified_by': metadata.get('LastModifiedBy'),
                    'company': metadata.get('Company'),
                    'title': metadata.get('Title'),
                    'subject': metadata.get('Subject'),
                    'keywords': metadata.get('Keywords'),
                    'software': metadata.get('Software') or metadata.get('Application'),
                    'revision_number': metadata.get('RevisionNumber'),
                    'total_edit_time': metadata.get('TotalEditTime'),
                    'template': metadata.get('Template'),
                    'gps_latitude': metadata.get('GPSLatitude'),
                    'gps_longitude': metadata.get('GPSLongitude'),
                }
                return {k: v for k, v in relevant.items() if v is not None}
    except Exception as e:
        return {'error': str(e)}

    return {}

def batch_metadata_extraction(directory_path, extensions=None):
    """Extract metadata from all documents in a directory"""
    extensions = extensions or ['.pdf', '.docx', '.xlsx', '.pptx', '.jpg', '.png']
    results = []

    for file_path in Path(directory_path).rglob('*'):
        if file_path.suffix.lower() in extensions:
            metadata = extract_document_metadata(file_path)
            if metadata and 'error' not in metadata:
                results.append({
                    'file': str(file_path),
                    'metadata': metadata
                })

    return results

# Analyze for patterns across multiple documents
def analyze_document_set(metadata_list):
    """Find patterns across a set of documents (e.g., leaked corporate files)"""
    from collections import Counter

    authors = Counter()
    companies = Counter()
    software = Counter()
    dates = []

    for item in metadata_list:
        m = item['metadata']
        if m.get('author'):
            authors[m['author']] += 1
        if m.get('company'):
            companies[m['company']] += 1
        if m.get('software'):
            software[m['software']] += 1
        if m.get('create_date'):
            dates.append(m['create_date'])

    return {
        'top_authors': authors.most_common(10),
        'companies': companies.most_common(10),
        'software_used': software.most_common(10),
        'date_range': (min(dates), max(dates)) if dates else None,
    }
```

---

## 9.8 Google Dorking for Security Research

Google dorking — using advanced search operators to find sensitive information inadvertently exposed online — is a fundamental technique in security research and authorized penetration testing. The Google Hacking Database (GHDB) maintained by Offensive Security contains thousands of documented dorks.

**Security research applications** (authorized use only):
```
# Find exposed admin panels
intitle:"Admin Panel" inurl:admin site:targetdomain.com

# Find login pages
intitle:"login" inurl:login site:targetdomain.com

# Find exposed cameras
intitle:"webcamXP" inurl:8080

# Find exposed printers
intitle:"HP Designjet" inurl:hp/device

# Find API keys in public code
site:github.com "api_key" "targetcompany"
site:github.com "AWS_SECRET_ACCESS_KEY"

# Find exposed database interfaces
intitle:"phpMyAdmin" inurl:/phpmyadmin

# Find exposed Elasticsearch instances
intitle:"Kibana" inurl:5601

# Find sensitive file types
site:targetdomain.com filetype:pdf "confidential"
site:targetdomain.com filetype:xls "salary" OR "payroll"
```

**Ethical and legal constraint**: Google dorking to find sensitive information on systems you do not have authorization to access may violate the CFAA even if the information appears in search results. The distinction between finding information in search results (generally acceptable) and using that information to access unauthorized systems (generally not acceptable) is critical.

The appropriate use of security-focused dorks is:
- Finding exposures on systems you own or have authorization to test
- Academic research and security education
- Reporting vulnerabilities to affected organizations (responsible disclosure)

---

## 9.9 Systematic Search Workflows

Professional investigations require systematic, documented search workflows rather than ad hoc searching.

### The OSINT Search Framework

**Query generation**: Before searching, generate a comprehensive list of search queries based on:
- All known name variants, aliases, and associated identifiers
- All known associated entities (companies, organizations, locations)
- Time-bounded queries for key periods
- Platform-specific queries for relevant platforms

**Coverage tracking**: Document which queries were run, when, and what they yielded. This prevents repeating searches and ensures coverage gaps are identified.

**Results management**: Systematic archiving of relevant results with source documentation.

```python
import json
from datetime import datetime
import itertools

class SearchWorkflow:
    """Systematic search workflow manager"""

    def __init__(self, investigation_name):
        self.investigation = investigation_name
        self.queries_run = []
        self.results = []
        self.coverage_log = {}

    def generate_query_matrix(self, names, companies, keywords, operators=None):
        """Generate systematic query combinations"""
        queries = []

        # Base name queries
        for name in names:
            queries.append(f'"{name}"')

        # Name + company combinations
        for name, company in itertools.product(names, companies):
            queries.append(f'"{name}" "{company}"')

        # Name + keyword combinations
        for name, keyword in itertools.product(names, keywords):
            queries.append(f'"{name}" "{keyword}"')

        # Site-specific queries
        sites = ['linkedin.com', 'twitter.com', 'facebook.com', 'instagram.com']
        for name, site in itertools.product(names, sites):
            queries.append(f'site:{site} "{name}"')

        # File type queries for documents
        for name in names:
            for filetype in ['pdf', 'docx', 'xlsx']:
                queries.append(f'"{name}" filetype:{filetype}')

        return queries

    def log_search(self, query, engine, results_count, relevant_count, notes=""):
        """Log a search execution"""
        entry = {
            'timestamp': datetime.now().isoformat(),
            'query': query,
            'engine': engine,
            'results_count': results_count,
            'relevant_count': relevant_count,
            'notes': notes
        }
        self.queries_run.append(entry)
        return entry

    def identify_coverage_gaps(self, subjects, time_periods):
        """Identify which subject/period combinations haven't been searched"""
        gaps = []
        for subject in subjects:
            for period in time_periods:
                key = f"{subject}_{period}"
                if key not in self.coverage_log:
                    gaps.append({'subject': subject, 'period': period})
        return gaps

    def export_search_log(self, output_path):
        """Export search log for documentation"""
        with open(output_path, 'w') as f:
            json.dump({
                'investigation': self.investigation,
                'total_queries': len(self.queries_run),
                'queries': self.queries_run,
            }, f, indent=2)
```

---

## Summary

Advanced search techniques transform OSINT from surface-level discovery to systematic intelligence gathering. Google's search operators, combined with alternative search engines, enable precisely targeted queries that surface content invisible to standard searches.

Historical content recovery — through the Wayback Machine, alternative archives, and search engine caches — enables investigators to access deleted, modified, and historical content that subjects often believe is gone. Document metadata extraction reveals authorship, organizational, and temporal information embedded in files.

Paste site and leak repository searching adds a specialized dimension for security-focused investigations. Systematic search workflows with documented coverage ensure investigation completeness and enable quality review.

The discipline that underlies all of these techniques is the same: generate comprehensive queries, document what was searched and what was found, and archive results with source documentation.

---

## Common Mistakes and Pitfalls

- **Query breadth neglect**: Searching for the formal name but not abbreviations, nicknames, and name variants
- **Recency bias in search engines**: Not using date restriction operators to find historical content
- **Single-engine reliance**: Missing significant content indexed by Bing, Yandex, or DuckDuckGo but not Google
- **Forgetting the Wayback Machine**: Not checking web archives for deleted content before concluding information doesn't exist
- **Ignoring document metadata**: Treating document content without analyzing embedded metadata
- **Undocumented searches**: Running searches without logging queries, making investigation coverage uncertain

---

## Further Reading

- Google Search Help — Official operator documentation
- GHDB (Exploit Database Google Hacking Database) — Security-focused dork collection
- Michael Bazzell, *OSINT Techniques* — comprehensive advanced search methodology
- The Wayback Machine CDX API documentation
- Archive.today — alternative archiving service documentation
