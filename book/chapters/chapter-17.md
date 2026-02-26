# Chapter 17: Visualization and Reporting

## Learning Objectives

By the end of this chapter, you will be able to:
- Select visualization types appropriate to different investigative findings
- Build timeline visualizations, relationship maps, and geographic displays
- Design professional investigation reports for different audiences
- Create interactive visualizations for complex investigations
- Present investigative uncertainty appropriately
- Build automated report generation workflows

---

## 17.1 Why Visualization Matters in Investigation

An investigation that produces correct findings but fails to communicate them effectively is a failed investigation. Visualization and reporting transform analytical conclusions into products that drive decisions, support legal proceedings, and enable action.

The challenge is matching visualization type to investigative finding type. Not all findings benefit from visual treatment, and the wrong visualization can obscure rather than illuminate. This chapter develops the framework for effective investigative communication.

---

## 17.2 Visualization Type Selection

### When to Visualize vs. When to Write

**Visualize when**:
- You are showing relationships between multiple entities
- You are showing change over time
- You are showing geographic patterns
- You are showing network structure
- You are showing comparison across multiple attributes
- The pattern would require many sentences to describe but is obvious in a chart

**Write narrative when**:
- You are explaining reasoning and analytical logic
- You are communicating uncertainty and confidence levels
- You are summarizing findings with caveats
- The finding is a single fact or small number of facts
- You need to establish evidentiary chain of custody

### Finding-to-Visualization Mapping

| Finding Type | Best Visualization |
|---|---|
| Relationships between entities | Network/link graph |
| Sequence of events | Timeline |
| Geographic activity patterns | Map with markers/heatmap |
| Financial flows | Sankey diagram / flow chart |
| Corporate ownership structure | Hierarchical tree |
| Comparison of entities | Table / parallel coordinates |
| Change in quantity over time | Line chart |
| Distribution of values | Histogram / box plot |
| Proportion of categories | Bar chart (avoid pie charts) |

---

## 17.3 Timeline Visualization

Timelines are among the most frequently useful investigative visualizations. They establish sequences, reveal gaps, and enable spotting of inconsistencies.

```python
import plotly.graph_objects as go
from datetime import datetime, timedelta
import pandas as pd
from typing import List, Dict, Optional

def create_investigation_timeline(
    events: List[Dict],
    title: str = "Investigation Timeline",
    output_file: str = "timeline.html"
) -> str:
    """
    Create an interactive timeline visualization

    events: list of dicts with keys:
        - date: datetime or ISO string
        - event: event description
        - category: event category (person, corporate, legal, financial, etc.)
        - source: source of information
        - confidence: 'confirmed', 'probable', 'possible'
        - significance: 'high', 'medium', 'low'
    """

    # Color mapping by category
    category_colors = {
        'person': '#4A90E2',
        'corporate': '#E24A4A',
        'legal': '#E2A44A',
        'financial': '#4AE24A',
        'digital': '#A44AE2',
        'geographic': '#4AE2A4',
        'other': '#999999'
    }

    # Confidence symbols
    confidence_symbols = {
        'confirmed': 'circle',
        'probable': 'diamond',
        'possible': 'triangle-up'
    }

    # Significance sizes
    significance_sizes = {
        'high': 20,
        'medium': 14,
        'low': 8
    }

    # Parse and sort events
    parsed_events = []
    for event in events:
        date = event.get('date', '')
        if isinstance(date, str):
            try:
                date = datetime.fromisoformat(date.replace('Z', '+00:00'))
            except ValueError:
                try:
                    date = datetime.strptime(date, '%Y-%m-%d')
                except ValueError:
                    continue

        parsed_events.append({**event, 'date_parsed': date})

    parsed_events.sort(key=lambda x: x['date_parsed'])

    # Create figure
    fig = go.Figure()

    # Group events by category for separate traces
    categories = list(set(e.get('category', 'other') for e in parsed_events))

    for category in categories:
        category_events = [e for e in parsed_events if e.get('category', 'other') == category]

        x_dates = [e['date_parsed'] for e in category_events]
        y_positions = [0] * len(category_events)  # All on same line initially

        hover_texts = [
            f"<b>{e.get('event', '')}</b><br>"
            f"Date: {e['date_parsed'].strftime('%Y-%m-%d')}<br>"
            f"Category: {category}<br>"
            f"Source: {e.get('source', 'Unknown')}<br>"
            f"Confidence: {e.get('confidence', 'Unknown')}"
            for e in category_events
        ]

        symbols = [confidence_symbols.get(e.get('confidence', 'possible'), 'circle')
                   for e in category_events]
        sizes = [significance_sizes.get(e.get('significance', 'medium'), 14)
                 for e in category_events]

        fig.add_trace(go.Scatter(
            x=x_dates,
            y=y_positions,
            mode='markers+text',
            name=category.title(),
            marker=dict(
                color=category_colors.get(category, '#999999'),
                size=sizes,
                symbol=symbols,
                line=dict(width=2, color='white')
            ),
            text=[e.get('event', '')[:30] + '...' if len(e.get('event', '')) > 30
                  else e.get('event', '')
                  for e in category_events],
            textposition='top center',
            hovertemplate='%{customdata}<extra></extra>',
            customdata=hover_texts,
        ))

    fig.update_layout(
        title=dict(text=title, font=dict(size=16)),
        xaxis=dict(
            title='Date',
            type='date',
            showgrid=True,
            gridcolor='lightgray'
        ),
        yaxis=dict(
            showticklabels=False,
            showgrid=False,
            zeroline=False,
        ),
        height=400,
        plot_bgcolor='white',
        paper_bgcolor='white',
        legend=dict(
            orientation='h',
            yanchor='bottom',
            y=-0.3,
            xanchor='center',
            x=0.5
        )
    )

    # Add annotation for confidence legend
    fig.add_annotation(
        text="● Confirmed  ◆ Probable  ▲ Possible",
        xref='paper', yref='paper',
        x=0.5, y=-0.15,
        showarrow=False,
        font=dict(size=10),
        align='center'
    )

    fig.write_html(output_file)
    return output_file


# Example usage
timeline_events = [
    {
        'date': '2020-03-15',
        'event': 'AcmeCorp incorporated in Delaware',
        'category': 'corporate',
        'source': 'Delaware Secretary of State',
        'confidence': 'confirmed',
        'significance': 'high'
    },
    {
        'date': '2021-06-01',
        'event': 'John Smith joins as CEO',
        'category': 'person',
        'source': 'SEC 8-K filing',
        'confidence': 'confirmed',
        'significance': 'medium'
    },
    {
        'date': '2022-08-15',
        'event': 'SEC investigation opened',
        'category': 'legal',
        'source': 'SEC public announcement',
        'confidence': 'confirmed',
        'significance': 'high'
    }
]

# create_investigation_timeline(timeline_events, title="AcmeCorp Timeline")
```

---

## 17.4 Network Visualization

Network visualizations communicate relationship structures that are difficult to describe in text.

```python
import networkx as nx
import plotly.graph_objects as go
from typing import Dict, List

def create_interactive_network(
    nodes: List[Dict],
    edges: List[Dict],
    title: str = "Investigation Network",
    output_file: str = "network.html"
) -> str:
    """
    Create an interactive network visualization using Plotly

    nodes: [{'id': str, 'label': str, 'type': str, 'size': int}]
    edges: [{'source': str, 'target': str, 'label': str, 'weight': float}]
    """

    # Create NetworkX graph for layout
    G = nx.Graph()

    for node in nodes:
        G.add_node(node['id'], **node)

    for edge in edges:
        G.add_edge(edge['source'], edge['target'], **edge)

    # Calculate layout
    if len(nodes) < 20:
        pos = nx.spring_layout(G, k=3, iterations=100, seed=42)
    elif len(nodes) < 100:
        pos = nx.kamada_kawai_layout(G)
    else:
        pos = nx.random_layout(G, seed=42)

    # Node type colors
    type_colors = {
        'person': '#4A90E2',
        'organization': '#E24A4A',
        'address': '#4AE24A',
        'domain': '#E2A44A',
        'ip_address': '#A44AE2',
        'financial': '#4AE2A4',
        'unknown': '#999999'
    }

    # Create edge traces
    edge_traces = []
    for edge in edges:
        if edge['source'] in pos and edge['target'] in pos:
            x0, y0 = pos[edge['source']]
            x1, y1 = pos[edge['target']]

            edge_traces.append(go.Scatter(
                x=[x0, x1, None],
                y=[y0, y1, None],
                mode='lines',
                line=dict(width=max(1, edge.get('weight', 1) * 2), color='#888888'),
                hoverinfo='none',
                showlegend=False
            ))

    # Create node traces by type
    node_groups = {}
    for node in nodes:
        node_type = node.get('type', 'unknown')
        if node_type not in node_groups:
            node_groups[node_type] = []
        node_groups[node_type].append(node)

    node_traces = []
    for node_type, type_nodes in node_groups.items():
        x_vals = []
        y_vals = []
        texts = []
        hover_texts = []
        sizes = []

        for node in type_nodes:
            if node['id'] in pos:
                x, y = pos[node['id']]
                x_vals.append(x)
                y_vals.append(y)
                texts.append(node.get('label', node['id'])[:20])
                hover_texts.append(
                    f"<b>{node.get('label', node['id'])}</b><br>"
                    f"Type: {node_type}<br>"
                    f"Connections: {G.degree(node['id'])}"
                )
                # Size proportional to degree
                sizes.append(max(10, G.degree(node['id']) * 5 + node.get('size', 10)))

        node_traces.append(go.Scatter(
            x=x_vals,
            y=y_vals,
            mode='markers+text',
            name=node_type.replace('_', ' ').title(),
            marker=dict(
                size=sizes,
                color=type_colors.get(node_type, '#999999'),
                line=dict(width=2, color='white')
            ),
            text=texts,
            textposition='top center',
            hovertemplate='%{customdata}<extra></extra>',
            customdata=hover_texts,
        ))

    # Assemble figure
    fig = go.Figure(
        data=edge_traces + node_traces,
        layout=go.Layout(
            title=dict(text=title, font=dict(size=16)),
            showlegend=True,
            hovermode='closest',
            margin=dict(b=20, l=5, r=5, t=40),
            xaxis=dict(showgrid=False, zeroline=False, showticklabels=False),
            yaxis=dict(showgrid=False, zeroline=False, showticklabels=False),
            height=600,
            plot_bgcolor='white',
        )
    )

    fig.write_html(output_file)
    return output_file
```

---

## 17.5 Report Writing for Investigators

Effective investigative reports follow consistent structure and disciplined language.

### Report Structure

```markdown
# [Investigation Name] — Intelligence Report

**Classification**: [Confidentiality level]
**Date**: [Report date]
**Prepared by**: [Analyst/Organization]
**Investigation Period**: [Dates covered]
**Report Version**: 1.0

---

## Executive Summary

[2-4 sentences stating the most important findings and overall assessment.
Written for a reader who may not read the full report.]

## Subjects of Investigation

[Brief description of each entity investigated with their relationship to the investigation purpose]

## Key Findings

### Finding 1: [Descriptive title]
**Confidence**: HIGH / MEDIUM / LOW
**Sources**: [Citation list]

[Finding description in narrative form. Clear distinction between confirmed facts and inferences.]

### Finding 2: [Descriptive title]
...

## Timeline of Key Events

[Chronological narrative or reference to attached timeline visualization]

## Relationship Map

[Reference to attached network visualization or embedded relationship description]

## Risk Assessment

[Assessment of key risks or concerns relevant to the investigation purpose]

## Information Gaps

[List what remains unknown that would be material to the investigation purpose]
[Note collection gaps — sources not searched, jurisdiction limits, etc.]

## Methodology

[Brief description of sources queried, collection methods used, and dates of collection]
[Any limitations on methodology]

## Confidence and Limitations

[Overall confidence level with justification]
[Specific limitations on findings]

## Recommendations

[If appropriate, specific next steps for the client]

---

## Source Documentation

[Complete citation for each source used, with URL and collection date]

| # | Source Type | Source | URL | Date Accessed | Information Provided |
|---|---|---|---|---|---|
| 1 | Public Record | SEC EDGAR | edgar.sec.gov/... | 2024-01-15 | 10-K filing confirming... |
```

### Automated Report Generation

```python
import anthropic
from datetime import datetime
import json

def generate_ai_assisted_report(
    investigation_name: str,
    subjects: List[str],
    findings: List[Dict],
    report_type: str = 'due_diligence'
) -> str:
    """
    Generate a professional investigation report using AI drafting
    with structured findings as input
    """

    client = anthropic.Anthropic()

    findings_text = "\n\n".join([
        f"FINDING {i+1} [{f.get('confidence', 'medium').upper()} CONFIDENCE]:\n"
        f"Type: {f.get('type', 'general')}\n"
        f"Source: {f.get('source', 'unknown')}\n"
        f"Date: {f.get('date', 'unknown')}\n"
        f"Details: {f.get('content', '')}"
        for i, f in enumerate(findings)
    ])

    report_type_instructions = {
        'due_diligence': "This is a corporate due diligence report for an investment or business relationship evaluation.",
        'background_check': "This is a professional background check report for employment or partnership evaluation.",
        'litigation_support': "This is an OSINT report prepared to support legal proceedings.",
        'threat_assessment': "This is a threat intelligence assessment for security purposes.",
    }

    prompt = f"""You are an experienced investigative analyst preparing a professional intelligence report.

{report_type_instructions.get(report_type, 'This is a general intelligence report.')}

INVESTIGATION: {investigation_name}
SUBJECTS: {', '.join(subjects)}
REPORT DATE: {datetime.now().strftime('%Y-%m-%d')}

COLLECTED FINDINGS:
{findings_text}

Write a professional intelligence report in the following structure:

## Executive Summary
[2-3 sentences. Most important findings and overall risk assessment.]

## Key Findings

[For each significant finding, write a clear paragraph with:
- What was found (confirmed facts)
- Source citation
- Analytical significance
- Confidence level
Distinguish clearly between facts and inferences.]

## Risk Assessment
[Structured assessment of risks relevant to the investigation purpose]

## Information Gaps and Limitations
[What was not found or could not be verified; collection limitations]

## Methodology Note
[Brief description of sources consulted]

IMPORTANT REQUIREMENTS:
- Use language that clearly distinguishes confirmed facts from inferences
- Use words of estimative probability appropriately (confirmed, probable, possible)
- Do not overstate confidence beyond what the findings support
- Cite sources for all significant claims
- Do not fabricate or infer information not present in the findings"""

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=4000,
        messages=[{"role": "user", "content": prompt}]
    )

    report_body = response.content[0].text

    # Assemble complete report with header and source documentation
    source_table = "| # | Source Type | Details | Date |\n|---|---|---|---|\n"
    for i, finding in enumerate(findings, 1):
        source_table += f"| {i} | {finding.get('type', 'General')} | {finding.get('source', 'Unknown')} | {finding.get('date', 'Unknown')} |\n"

    full_report = f"""# Intelligence Report: {investigation_name}

**Date**: {datetime.now().strftime('%Y-%m-%d')}
**Investigation Period**: Through {datetime.now().strftime('%Y-%m-%d')}
**Report Version**: 1.0

---

{report_body}

---

## Source Documentation

{source_table}

---

*This report is based on publicly available information and records accessed during the investigation period. Findings should be treated as intelligence product, not legal evidence, unless otherwise noted. Verify all significant findings against primary sources before consequential use.*
"""

    return full_report
```

---

## 17.6 Communicating Uncertainty Visually

Uncertainty communication in visualization is an underappreciated discipline. Investigators must convey the confidence level of findings without misleading readers.

```python
def add_confidence_indicators_to_chart(fig, findings: List[Dict]):
    """
    Add visual confidence indicators to any Plotly figure
    """
    confidence_legend = {
        'confirmed': {'color': 'green', 'opacity': 1.0, 'text': 'Confirmed (multiple sources)'},
        'probable': {'color': 'orange', 'opacity': 0.8, 'text': 'Probable (single reliable source)'},
        'possible': {'color': 'red', 'opacity': 0.5, 'text': 'Possible (unverified)'},
        'speculative': {'color': 'gray', 'opacity': 0.3, 'text': 'Speculative (inferred)'}
    }

    # Add confidence legend annotation
    annotation_text = "<br>".join([
        f'<span style="color:{v["color"]}">■</span> {v["text"]}'
        for k, v in confidence_legend.items()
    ])

    fig.add_annotation(
        text=f"<b>Confidence Level:</b><br>{annotation_text}",
        xref='paper', yref='paper',
        x=1.0, y=1.0,
        xanchor='right',
        showarrow=False,
        font=dict(size=10),
        bgcolor='white',
        bordercolor='lightgray',
        borderwidth=1
    )

    return fig
```

---

## Summary

Visualization and reporting are where investigative analysis becomes actionable intelligence. Selecting the right visualization type for each finding type — timelines for events, networks for relationships, maps for geography — communicates patterns that prose cannot.

Professional investigative reports follow consistent structure: executive summary, findings with confidence levels and citations, methodology, and limitations. AI-assisted drafting accelerates report production but all AI-generated content requires human review before delivery.

Communicating uncertainty visually and in writing is a professional obligation. Findings must be presented with confidence levels that accurately reflect the quality and completeness of the supporting evidence.

---

## Common Mistakes and Pitfalls

- **Visualization overload**: Creating complex visualizations that obscure rather than illuminate findings
- **Confidence misrepresentation**: Presenting probable findings as confirmed through visual design choices
- **Missing source documentation**: Reports without complete source citations cannot be verified or defended
- **Audience calibration failure**: Using technical visualization for non-technical audiences or oversimplifying for technical ones
- **No version control for reports**: Updated reports without version tracking create confusion about which version is current
- **AI-drafted content without review**: AI report drafting errors can appear professional and be missed without careful review

---

## Further Reading

- Edward Tufte, *The Visual Display of Quantitative Information*
- Alberto Cairo, *How Charts Lie* — visual misinformation and honest data visualization
- Plotly documentation — interactive visualization library
- Gephi documentation — network visualization platform
- SANS analyst program — analytical report writing standards
