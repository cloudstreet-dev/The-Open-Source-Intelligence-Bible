# Chapter 13: Network Analysis and Graph Intelligence

## Learning Objectives

By the end of this chapter, you will be able to:
- Model investigative data as graphs and apply network analysis techniques
- Identify key nodes, brokers, and communities within social and corporate networks
- Apply link analysis to uncover hidden relationships
- Use graph databases for persistent relationship data storage
- Visualize networks effectively for investigative reporting
- Apply AI to enhance network analysis capability
- Detect anomalous network structures indicative of fraud or coordinated behavior

---

## 13.1 Why Graph Thinking Transforms OSINT

Information does not exist in isolation. Every person has relationships — professional, social, financial, familial. Every organization has connections — subsidiaries, partners, customers, regulators. Every digital asset has infrastructure relationships — domains, IPs, certificates, hosting providers.

OSINT investigations that treat data as isolated facts miss the most important layer: the relational structure that connects facts together. A person who appears benign in isolation may have critical connections to sanctioned entities, known fraudsters, or criminal networks. A company that looks straightforward may be one node in a complex beneficial ownership chain designed to obscure the actual controlling party.

Graph analysis — modeling entities as nodes and relationships as edges, then applying network analysis algorithms — reveals this relational layer systematically.

---

## 13.2 Graph Fundamentals for Investigators

### Basic Graph Concepts

A **graph** consists of **nodes** (also called vertices) and **edges** (connections between nodes). In OSINT investigations:

**Nodes represent**: People, organizations, addresses, phone numbers, email addresses, domains, IP addresses, financial accounts, vehicles, properties.

**Edges represent**: Employment relationships, ownership, co-residence, communications, financial transactions, shared infrastructure, legal relationships.

**Graph types**:
- **Undirected**: Edges have no direction (A connected to B = B connected to A)
- **Directed**: Edges have direction (A owns B ≠ B owns A)
- **Weighted**: Edges carry values (frequency of communications, financial amounts)
- **Attributed**: Nodes and edges carry additional data (timestamps, confidence scores)

### Key Network Metrics

**Degree**: Number of direct connections a node has. High degree = many direct connections.

**In-degree/Out-degree** (directed graphs): Separate counts of incoming and outgoing connections. Useful for identifying sources (many out-edges) and targets (many in-edges) of influence or financial flows.

**Betweenness centrality**: A node's position on paths between other nodes. High betweenness indicates a broker or bridge connecting otherwise disconnected groups. Particularly important for identifying key intermediaries in financial networks or influence operations.

**Eigenvector centrality**: Weighted importance based on connection to other important nodes. Being connected to well-connected nodes is more valuable than being connected to isolated ones.

**Clustering coefficient**: Proportion of a node's neighbors that are also connected to each other. High clustering indicates tight community membership.

**Path length**: The shortest number of edges to traverse between two nodes. Short paths indicate closer relationship.

---

## 13.3 Building Investigation Graphs with NetworkX

```python
import networkx as nx
import matplotlib.pyplot as plt
import matplotlib.patches as mpatches
from matplotlib.lines import Line2D
import json
from datetime import datetime
from typing import Dict, List, Optional, Tuple, Any

class InvestigationGraph:
    """
    Graph-based investigation data model
    Supports both directed and undirected relationship types
    """

    # Node type colors for visualization
    NODE_COLORS = {
        'person': '#4A90E2',
        'organization': '#E24A4A',
        'address': '#4AE24A',
        'domain': '#E2A44A',
        'ip_address': '#A44AE2',
        'financial_account': '#4AE2A4',
        'phone': '#E2E24A',
        'email': '#4AACE2',
        'property': '#8B4513',
        'vehicle': '#708090',
        'unknown': '#999999'
    }

    def __init__(self, investigation_name: str):
        self.name = investigation_name
        self.G = nx.MultiDiGraph()  # Multigraph allows multiple edges between same nodes
        self.created = datetime.now().isoformat()

    def add_entity(
        self,
        entity_id: str,
        entity_type: str,
        label: str,
        attributes: dict = None,
        source: str = None,
        confidence: float = 1.0
    ):
        """Add an entity (node) to the investigation graph"""
        node_attrs = {
            'type': entity_type,
            'label': label,
            'source': source,
            'confidence': confidence,
            'added_at': datetime.now().isoformat(),
            **(attributes or {})
        }
        self.G.add_node(entity_id, **node_attrs)

    def add_relationship(
        self,
        source_id: str,
        target_id: str,
        relationship_type: str,
        attributes: dict = None,
        source: str = None,
        confidence: float = 1.0,
        start_date: str = None,
        end_date: str = None
    ):
        """Add a relationship (edge) between entities"""
        edge_attrs = {
            'type': relationship_type,
            'source': source,
            'confidence': confidence,
            'start_date': start_date,
            'end_date': end_date,
            'added_at': datetime.now().isoformat(),
            **(attributes or {})
        }
        self.G.add_edge(source_id, target_id, **edge_attrs)

    def calculate_centrality(self) -> dict:
        """Calculate key centrality metrics for all nodes"""
        undirected = self.G.to_undirected()

        metrics = {}

        # Degree centrality
        degree_cent = nx.degree_centrality(undirected)

        # Betweenness centrality (expensive for large graphs)
        if len(self.G.nodes) < 10000:
            between_cent = nx.betweenness_centrality(undirected)
        else:
            between_cent = {n: 0 for n in self.G.nodes}

        # PageRank (directionality-aware authority)
        try:
            pagerank = nx.pagerank(self.G, max_iter=100)
        except Exception:
            pagerank = {n: 1/len(self.G.nodes) for n in self.G.nodes}

        for node in self.G.nodes:
            metrics[node] = {
                'degree_centrality': degree_cent.get(node, 0),
                'betweenness_centrality': between_cent.get(node, 0),
                'pagerank': pagerank.get(node, 0),
                'degree': self.G.degree(node),
                'in_degree': self.G.in_degree(node),
                'out_degree': self.G.out_degree(node),
            }

        return metrics

    def find_communities(self, algorithm='louvain') -> List[set]:
        """Detect communities (clusters) in the network"""
        undirected = self.G.to_undirected()

        if algorithm == 'louvain':
            try:
                import community as community_louvain
                partition = community_louvain.best_partition(undirected)
                # Convert to list of sets
                community_map = {}
                for node, community_id in partition.items():
                    if community_id not in community_map:
                        community_map[community_id] = set()
                    community_map[community_id].add(node)
                return list(community_map.values())
            except ImportError:
                pass

        # Fall back to Girvan-Newman
        communities = list(nx.algorithms.community.girvan_newman(undirected))
        if communities:
            return [set(c) for c in communities[0]]
        return []

    def find_shortest_path(self, source_id: str, target_id: str) -> Optional[list]:
        """Find the shortest path between two entities"""
        try:
            undirected = self.G.to_undirected()
            path = nx.shortest_path(undirected, source_id, target_id)
            return path
        except nx.NetworkXNoPath:
            return None
        except nx.NodeNotFound as e:
            return None

    def get_neighborhood(self, entity_id: str, depth: int = 2) -> 'InvestigationGraph':
        """Extract the local neighborhood around an entity"""
        undirected = self.G.to_undirected()
        ego_graph = nx.ego_graph(undirected, entity_id, radius=depth)

        # Create new investigation graph from neighborhood
        neighborhood = InvestigationGraph(f"Neighborhood of {entity_id}")
        for node in ego_graph.nodes:
            neighborhood.G.add_node(node, **self.G.nodes[node])
        for edge in ego_graph.edges:
            for key, data in self.G[edge[0]][edge[1]].items():
                neighborhood.G.add_edge(edge[0], edge[1], **data)

        return neighborhood

    def detect_suspicious_structures(self) -> List[dict]:
        """
        Identify network structures that may indicate suspicious activity
        """
        suspicious = []

        # 1. Detect shell company chains (long linear ownership chains)
        ownership_edges = [
            (u, v) for u, v, d in self.G.edges(data=True)
            if d.get('type') in ('owns', 'subsidiary_of', 'controls')
        ]
        ownership_graph = nx.DiGraph()
        ownership_graph.add_edges_from(ownership_edges)

        for node in ownership_graph.nodes:
            # Find nodes in a linear chain of 3+ ownership hops
            out_edges = list(ownership_graph.out_edges(node))
            if len(out_edges) == 1:  # Single ownership chain
                successor = out_edges[0][1]
                if ownership_graph.out_degree(successor) == 1:
                    suspicious.append({
                        'type': 'ownership_chain',
                        'description': f'Linear ownership chain starting at {node}',
                        'nodes': [node, successor],
                        'risk_level': 'MEDIUM'
                    })

        # 2. Detect shared infrastructure (many domains pointing to same IP)
        ip_domains = {}
        for u, v, d in self.G.edges(data=True):
            if d.get('type') == 'resolves_to':
                target_type = self.G.nodes[v].get('type', '')
                if target_type == 'ip_address':
                    if v not in ip_domains:
                        ip_domains[v] = []
                    ip_domains[v].append(u)

        for ip, domains in ip_domains.items():
            if len(domains) > 10:  # Many domains on one IP
                suspicious.append({
                    'type': 'domain_hosting_cluster',
                    'description': f'IP {ip} hosts {len(domains)} domains',
                    'nodes': [ip] + domains[:5],
                    'risk_level': 'HIGH' if len(domains) > 50 else 'MEDIUM'
                })

        # 3. Detect circular ownership (potential control obfuscation)
        try:
            cycles = list(nx.simple_cycles(ownership_graph))
            for cycle in cycles[:5]:  # Limit to first 5
                suspicious.append({
                    'type': 'circular_ownership',
                    'description': f'Circular ownership detected: {" → ".join(cycle + [cycle[0]])}',
                    'nodes': cycle,
                    'risk_level': 'HIGH'
                })
        except Exception:
            pass

        return suspicious

    def visualize(self, output_file: str = 'investigation_graph.png',
                  highlight_nodes: List[str] = None, figsize: tuple = (20, 15)):
        """
        Generate a visualization of the investigation graph
        """
        plt.figure(figsize=figsize)

        # Layout
        if len(self.G.nodes) < 50:
            pos = nx.spring_layout(self.G, k=2, iterations=50, seed=42)
        else:
            pos = nx.kamada_kawai_layout(self.G)

        # Node colors by type
        node_colors = []
        node_sizes = []

        # Calculate degree for size scaling
        degrees = dict(self.G.degree())

        for node in self.G.nodes:
            node_type = self.G.nodes[node].get('type', 'unknown')
            color = self.NODE_COLORS.get(node_type, '#999999')
            if highlight_nodes and node in highlight_nodes:
                color = '#FF0000'  # Red for highlighted nodes
            node_colors.append(color)
            # Size proportional to degree
            node_sizes.append(300 + degrees[node] * 100)

        # Draw nodes
        nx.draw_networkx_nodes(
            self.G, pos,
            node_color=node_colors,
            node_size=node_sizes,
            alpha=0.9
        )

        # Draw edges with different styles by type
        nx.draw_networkx_edges(
            self.G, pos,
            edge_color='#555555',
            arrows=True,
            alpha=0.5,
            arrowsize=10
        )

        # Draw labels for important nodes
        important_nodes = {
            n: self.G.nodes[n].get('label', n)
            for n in self.G.nodes
            if degrees[n] > 2 or (highlight_nodes and n in highlight_nodes)
        }
        nx.draw_networkx_labels(
            self.G, pos,
            labels=important_nodes,
            font_size=8
        )

        # Legend
        legend_elements = [
            mpatches.Patch(facecolor=color, label=node_type.replace('_', ' ').title())
            for node_type, color in self.NODE_COLORS.items()
            if any(self.G.nodes[n].get('type') == node_type for n in self.G.nodes)
        ]
        plt.legend(handles=legend_elements, loc='upper left', fontsize=8)

        plt.title(f'Investigation Graph: {self.name}\n{len(self.G.nodes)} entities, {len(self.G.edges)} relationships')
        plt.axis('off')
        plt.tight_layout()
        plt.savefig(output_file, dpi=150, bbox_inches='tight')
        plt.close()

        return output_file

    def export_to_gephi(self, output_file: str):
        """Export graph to GEXF format for Gephi visualization"""
        nx.write_gexf(self.G, output_file)

    def export_summary(self) -> dict:
        """Generate a summary of the investigation graph"""
        centrality = self.calculate_centrality()

        # Most central entities by betweenness (brokers)
        top_brokers = sorted(
            centrality.items(),
            key=lambda x: x[1]['betweenness_centrality'],
            reverse=True
        )[:10]

        # Most connected entities
        top_connected = sorted(
            centrality.items(),
            key=lambda x: x[1]['degree'],
            reverse=True
        )[:10]

        return {
            'investigation': self.name,
            'total_entities': len(self.G.nodes),
            'total_relationships': len(self.G.edges),
            'entity_types': dict(
                nx.get_node_attributes(self.G, 'type').values()
                if self.G.nodes else {}
            ),
            'top_brokers': [
                {
                    'entity': entity_id,
                    'label': self.G.nodes[entity_id].get('label', entity_id),
                    'betweenness': round(metrics['betweenness_centrality'], 4),
                }
                for entity_id, metrics in top_brokers
            ],
            'top_connected': [
                {
                    'entity': entity_id,
                    'label': self.G.nodes[entity_id].get('label', entity_id),
                    'connections': metrics['degree'],
                }
                for entity_id, metrics in top_connected
            ],
            'suspicious_structures': self.detect_suspicious_structures(),
        }
```

---

## 13.4 Graph Databases for Persistent Investigation Data

For large-scale investigations, storing graphs in memory is impractical. Graph databases provide persistent, queryable storage optimized for relationship traversal.

### Neo4j for OSINT

Neo4j is the most widely deployed graph database. It uses the Cypher query language for expressive graph traversal.

```python
from neo4j import GraphDatabase
import json

class Neo4jInvestigationStore:
    """Neo4j-backed investigation graph database"""

    def __init__(self, uri, username, password):
        self.driver = GraphDatabase.driver(uri, auth=(username, password))

    def close(self):
        self.driver.close()

    def add_entity(self, entity_id: str, entity_type: str, properties: dict):
        """Add or update an entity in the graph database"""
        cypher = f"""
        MERGE (e:{entity_type} {{id: $id}})
        SET e += $props
        RETURN e
        """
        with self.driver.session() as session:
            result = session.run(cypher, id=entity_id, props=properties)
            return result.single()

    def add_relationship(self, source_id: str, target_id: str,
                        rel_type: str, properties: dict = None):
        """Add a relationship between two entities"""
        cypher = f"""
        MATCH (a {{id: $source_id}})
        MATCH (b {{id: $target_id}})
        MERGE (a)-[r:{rel_type.upper().replace(' ', '_')}]->(b)
        SET r += $props
        RETURN r
        """
        with self.driver.session() as session:
            result = session.run(
                cypher,
                source_id=source_id,
                target_id=target_id,
                props=properties or {}
            )
            return result.single()

    def find_all_paths(self, source_entity_id: str, target_entity_id: str,
                      max_hops: int = 6):
        """Find all paths between two entities up to max_hops"""
        cypher = """
        MATCH path = allShortestPaths((a {id: $source_id})-[*1..{max_hops}]-(b {id: $target_id}))
        RETURN path
        LIMIT 20
        """.replace('{max_hops}', str(max_hops))

        with self.driver.session() as session:
            results = session.run(cypher, source_id=source_entity_id,
                                 target_id=target_entity_id)
            paths = []
            for record in results:
                path = record['path']
                path_nodes = [
                    {'id': node.get('id'), 'type': list(node.labels)[0] if node.labels else 'Unknown'}
                    for node in path.nodes
                ]
                path_rels = [rel.type for rel in path.relationships]
                paths.append({
                    'nodes': path_nodes,
                    'relationships': path_rels,
                    'length': len(path_rels)
                })
            return paths

    def find_common_connections(self, entity_ids: List[str]) -> list:
        """Find entities connected to all specified entities"""
        cypher = """
        MATCH (shared)
        WHERE ALL(id IN $entity_ids WHERE EXISTS {
            MATCH (e {id: id})-[*1..3]-(shared)
        })
        AND NOT shared.id IN $entity_ids
        RETURN shared.id, shared, COUNT{(shared)-[]-() } as degree
        ORDER BY degree DESC
        LIMIT 20
        """
        with self.driver.session() as session:
            results = session.run(cypher, entity_ids=entity_ids)
            return [{'id': r['shared.id'], 'degree': r['degree']} for r in results]

    def find_circular_structures(self) -> list:
        """Detect circular ownership/control structures"""
        cypher = """
        MATCH path = (a:Organization)-[:OWNS|CONTROLS*2..8]->(a)
        RETURN [n in nodes(path) | n.id] as cycle
        LIMIT 20
        """
        with self.driver.session() as session:
            results = session.run(cypher)
            return [r['cycle'] for r in results]

    def analyze_influence_network(self, entity_id: str, depth: int = 3) -> dict:
        """Analyze the influence network of an entity"""
        cypher = """
        MATCH (center {id: $entity_id})
        CALL apoc.path.subgraphAll(center, {
            maxLevel: $depth,
            relationshipFilter: 'EMPLOYS|OWNS|CONTROLS|AFFILIATED_WITH'
        })
        YIELD nodes, relationships
        RETURN nodes, relationships
        """
        with self.driver.session() as session:
            results = session.run(cypher, entity_id=entity_id, depth=depth)
            record = results.single()
            if record:
                return {
                    'center': entity_id,
                    'network_size': len(record['nodes']),
                    'relationship_count': len(record['relationships'])
                }
            return {}
```

---

## 13.5 Detecting Coordinated Inauthentic Networks

One of the most important applications of network analysis in modern OSINT is detecting coordinated inauthentic behavior — networks of accounts or entities acting in coordination to amplify narratives, conduct fraud, or hide beneficial ownership.

### Coordination Detection Patterns

```python
import pandas as pd
from itertools import combinations
from scipy import stats

def detect_coordination_signals(activity_data: pd.DataFrame, time_window_seconds: int = 60) -> dict:
    """
    Detect coordinated behavior from activity data
    activity_data: DataFrame with columns ['account_id', 'action', 'timestamp', 'content_hash']
    """
    results = {
        'temporal_coordination': [],
        'content_coordination': [],
        'network_coordination': []
    }

    # 1. Temporal coordination: accounts acting within same time window
    activity_data['timestamp_dt'] = pd.to_datetime(activity_data['timestamp'])

    # Group by time windows and find co-occurring accounts
    activity_data['time_bucket'] = activity_data['timestamp_dt'].dt.floor(f'{time_window_seconds}s')

    time_groups = activity_data.groupby('time_bucket')['account_id'].apply(list)

    coordination_counts = {}
    for _, accounts in time_groups.items():
        if len(accounts) > 1:
            for pair in combinations(sorted(set(accounts)), 2):
                key = pair
                coordination_counts[key] = coordination_counts.get(key, 0) + 1

    # Pairs with high temporal coordination
    for pair, count in sorted(coordination_counts.items(), key=lambda x: x[1], reverse=True):
        if count > 5:  # Threshold: coordinated on 5+ occasions
            results['temporal_coordination'].append({
                'accounts': list(pair),
                'coordination_count': count,
                'risk': 'HIGH' if count > 20 else 'MEDIUM'
            })

    # 2. Content coordination: posting identical or near-identical content
    if 'content_hash' in activity_data.columns:
        content_groups = activity_data.groupby('content_hash')['account_id'].apply(list)
        for content_hash, accounts in content_groups.items():
            unique_accounts = list(set(accounts))
            if len(unique_accounts) > 2:
                results['content_coordination'].append({
                    'content_hash': content_hash,
                    'accounts': unique_accounts,
                    'count': len(unique_accounts),
                    'risk': 'HIGH'
                })

    return results

def build_coordination_graph(coordination_signals: dict) -> InvestigationGraph:
    """Build a graph of detected coordination networks"""
    coord_graph = InvestigationGraph("Coordination Detection")

    # Add accounts as nodes
    all_accounts = set()
    for signal in coordination_signals.get('temporal_coordination', []):
        all_accounts.update(signal['accounts'])
    for signal in coordination_signals.get('content_coordination', []):
        all_accounts.update(signal['accounts'])

    for account in all_accounts:
        coord_graph.add_entity(account, 'person', account)

    # Add coordination edges
    for signal in coordination_signals.get('temporal_coordination', []):
        accounts = signal['accounts']
        for i in range(len(accounts)):
            for j in range(i+1, len(accounts)):
                coord_graph.add_relationship(
                    accounts[i], accounts[j],
                    'temporal_coordination',
                    attributes={'count': signal['coordination_count']},
                    confidence=min(1.0, signal['coordination_count'] / 100)
                )

    return coord_graph
```

---

## 13.6 Financial Network Analysis

Financial networks — beneficial ownership, fund flows, sanctioned entity relationships — are a critical domain for graph analysis.

```python
def analyze_beneficial_ownership_chain(graph: InvestigationGraph, entity_id: str) -> dict:
    """
    Trace beneficial ownership through corporate layers
    Returns the ultimate beneficial owner(s) if findable
    """
    ownership_chain = {
        'starting_entity': entity_id,
        'layers': [],
        'ultimate_owners': [],
        'shell_indicators': [],
    }

    # Traverse the ownership graph upward
    current_level = [entity_id]
    visited = set()
    depth = 0

    while current_level and depth < 20:  # Prevent infinite loops
        next_level = []
        layer = {'depth': depth, 'entities': []}

        for entity in current_level:
            if entity in visited:
                continue
            visited.add(entity)

            # Find entities that OWN this entity
            owners = [
                source for source, target, data in graph.G.in_edges(entity, data=True)
                if data.get('type') in ('owns', 'controls', 'beneficial_owner_of')
            ]

            entity_data = graph.G.nodes.get(entity, {})
            layer['entities'].append({
                'id': entity,
                'label': entity_data.get('label', entity),
                'type': entity_data.get('type', 'unknown'),
                'owners': owners,
                'is_shell_indicator': _is_shell_indicator(entity_data)
            })

            if entity_data.get('type') == 'organization':
                if entity_data.get('is_shell_indicator'):
                    ownership_chain['shell_indicators'].append(entity)

            if not owners:
                # This entity has no further owners — may be ultimate owner
                if entity_data.get('type') == 'person':
                    ownership_chain['ultimate_owners'].append({
                        'id': entity,
                        'label': entity_data.get('label', entity),
                        'depth': depth
                    })

            next_level.extend(owners)

        if layer['entities']:
            ownership_chain['layers'].append(layer)

        current_level = next_level
        depth += 1

    return ownership_chain

def _is_shell_indicator(entity_data: dict) -> bool:
    """Check if entity data suggests a shell company"""
    indicators = [
        entity_data.get('jurisdiction') in ['BVI', 'Cayman Islands', 'Panama', 'Seychelles', 'Marshall Islands'],
        entity_data.get('employee_count', 99) < 2,
        'nominee' in entity_data.get('notes', '').lower(),
        entity_data.get('registered_agent_only', False),
    ]
    return any(indicators)
```

---

## Summary

Network analysis transforms OSINT from fact collection to relationship intelligence. By modeling investigative data as graphs — entities as nodes, relationships as edges — investigators can reveal hidden connections, identify key brokers, detect coordinated behavior, and trace beneficial ownership through complex corporate structures.

NetworkX provides Python-native graph analysis capability suitable for investigative applications. Neo4j and other graph databases enable persistent, scalable graph storage with expressive query capability. Visualization tools communicate network findings to non-technical audiences.

Specific high-value applications include corporate beneficial ownership tracing, coordinated inauthentic behavior detection in social networks, financial flow analysis, and infrastructure relationship mapping for threat intelligence.

---

## Common Mistakes and Pitfalls

- **Node ambiguity**: Failing to disambiguate nodes — treating "John Smith" the CEO and "John Smith" the accountant as the same node
- **Edge conflation**: Treating all relationship types as equivalent when relationship type matters for analysis
- **Scale blindness**: Centrality metrics that work for small graphs produce misleading results at scale without normalization
- **Missing temporal dimension**: Static graph analysis misses how networks evolve over time
- **Overinterpreting clustering**: Community detection algorithms always find communities even in random graphs; communities must be interpreted with domain knowledge
- **Visualization overload**: Large graph visualizations are often unreadable; focus on subgraphs or use hierarchical visualization

---

## Further Reading

- Barabási, Albert-László. *Network Science* (free at networksciencebook.com)
- Krebs, Valdis. Work on social network analysis and dark networks
- Financial Action Task Force (FATF) guidance on beneficial ownership analysis
- Neo4j documentation and graph data science library
- ICIJ's Linkurious platform methodology for offshore leaks data
