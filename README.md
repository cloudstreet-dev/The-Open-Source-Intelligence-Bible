# The Open Source Intelligence Bible

**A comprehensive technical guide to modern OSINT practice — from foundational methodology through AI-driven automation, applied across security, journalism, finance, and law.**

[![Deploy Docs](https://github.com/cloudstreet-dev/The-Open-Source-Intelligence-Bible/actions/workflows/deploy-docs.yml/badge.svg)](https://github.com/cloudstreet-dev/The-Open-Source-Intelligence-Bible/actions/workflows/deploy-docs.yml)

### [→ Read the book at cloudstreet-dev.github.io/The-Open-Source-Intelligence-Bible](https://cloudstreet-dev.github.io/The-Open-Source-Intelligence-Bible/)

---

## What this is

30 chapters and 5 appendices covering the full spectrum of OSINT practice — written for practitioners who build things and need to understand the methodology behind the tools, not just a list of them.

The book is organized in seven parts:

| Part | Chapters | Topic |
|---|---|---|
| I | 1–4 | Foundations: what OSINT is, the data landscape, legal frameworks, investigative workflow |
| II | 5–9 | Core techniques: social media, domain/network recon, public records, geospatial, advanced search |
| III | 10–13 | AI and data: LLMs, NLP pipelines, prompt engineering, graph analysis |
| IV | 14–17 | Automation: investigative pipelines, tool ecosystem, AI platforms, visualization |
| V | 18–22 | Applied domains: PI workflows, bug bounty, threat intel, financial crime, corporate due diligence |
| VI | 23–26 | Advanced: enterprise scale, adversarial OSINT, emerging tech, operational security |
| VII | 27–30 | Practice: stack design, case studies, failure modes, the future of the field |

Plus appendices covering tool reference, Python code examples, legal resources, a glossary, and further reading.

---

## Who it's for

- **Security and threat intelligence professionals** building structured OSINT methodology
- **Developers and data engineers** constructing investigative pipelines and automation
- **Private investigators** modernizing workflows with technology and AI
- **Journalists and researchers** conducting open-source investigations
- **Corporate security, compliance, and due diligence teams** assessing third-party and counterparty risk
- **Bug bounty hunters and vulnerability researchers** systematizing reconnaissance

The book assumes technical competence. Code examples are in Python. Readers who are not developers will still find the methodology and frameworks valuable.

---

## Technical depth

Every chapter includes working Python code. The book covers:

- `spaCy`, `transformers`, and LLM APIs for entity extraction and document analysis
- `NetworkX` and `Neo4j` for relationship graph construction and analysis
- `Celery`, `Kafka`, and `Elasticsearch` for enterprise-scale collection pipelines
- `Plotly` and `Gephi` for investigative visualization
- The Anthropic API (Claude) throughout for AI-assisted analysis, synthesis, and agentic workflows
- `requests`, `dnspython`, `pdfplumber`, `pytesseract`, and the full OSINT Python stack

---

## Legal and ethical positioning

All techniques described are intended for lawful, authorized, and ethical use. The book treats legal and ethical constraints as first-class design requirements — not footnotes. Chapter 3 covers the full legal framework. Every applied chapter includes ethical considerations specific to its domain.

This book does not enable or encourage unauthorized access, covert surveillance without legal authority, stalking or harassment, or evidence manipulation.

---

## Structure

```
book/
├── README.md               # Book introduction (site home page)
├── chapters/
│   ├── chapter-01.md       # What OSINT Is in the Modern Era
│   ├── chapter-02.md       # The Data Landscape and Sources
│   ├── ...
│   └── chapter-30.md       # The Future of OSINT Practice
└── appendices/
    ├── appendix-a-tool-reference.md
    ├── appendix-b-python-examples.md
    ├── appendix-c-legal-resources.md
    ├── appendix-d-glossary.md
    └── appendix-e-further-reading.md
```

---

## Published by CloudStreet

Special thanks to **Georgiy Treyvus**, CloudStreet Product Manager, for the original concept and structural vision behind this book.

*Nothing in this book constitutes legal advice. Content is provided for educational and professional development purposes.*
