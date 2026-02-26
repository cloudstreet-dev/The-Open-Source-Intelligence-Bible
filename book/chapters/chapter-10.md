# Chapter 10: AI Fundamentals for Investigators

## Learning Objectives

By the end of this chapter, you will be able to:
- Understand the core AI and machine learning concepts relevant to OSINT applications
- Evaluate AI tools for investigative use on the basis of appropriate criteria
- Use large language models (LLMs) and multimodal models as investigative aids
- Recognize the limitations, failure modes, and biases of AI systems
- Apply AI tools responsibly with appropriate human oversight
- Understand the difference between AI as a tool and AI as an autonomous agent

---

## 10.1 Why Investigators Must Understand AI

AI has moved from an experimental curiosity to a production tool reshaping every phase of the intelligence cycle. Investigators who do not understand AI fundamentals will:

- Misuse AI tools by trusting outputs that require verification
- Miss opportunities that AI-enabled workflows provide
- Be unable to evaluate AI-generated content as part of their investigations
- Fail to build workflows that leverage AI effectively
- Be at a competitive disadvantage to colleagues and adversaries who do use AI

This chapter provides the conceptual foundation. The following chapters in Part III apply that foundation to specific OSINT capabilities. None of this requires a machine learning research background. It requires a practitioner's understanding of what these systems do, how they work at a conceptual level, and what their failure modes are.

---

## 10.2 Machine Learning Fundamentals

### What Machine Learning Is

Traditional software executes explicit rules written by programmers. Machine learning systems learn patterns from data and generalize those patterns to new inputs.

**Supervised learning**: A model is trained on labeled examples (input → correct output). The model learns to predict labels for new inputs. Example: a spam filter trained on millions of labeled spam/not-spam emails learns to classify new emails.

**Unsupervised learning**: A model finds structure in unlabeled data — clusters, patterns, anomalies — without explicit labels. Example: clustering news articles by topic without pre-defined topic labels.

**Self-supervised learning**: A model learns from the data itself by predicting parts of the input from other parts. This is how large language models are trained: they learn to predict the next word given previous words.

**Reinforcement learning from human feedback (RLHF)**: The training method used for modern conversational AI. Models are trained to maximize rewards given by human evaluators, steering them toward helpful, harmless, and honest behavior.

### Neural Networks and Deep Learning

Modern AI systems are built on artificial neural networks — mathematical structures loosely inspired by biological neural networks. Key concepts:

**Layers**: Neural networks process input through sequences of mathematical transformations (layers). Deep learning refers to networks with many layers.

**Parameters (weights)**: The learned values that define the network's behavior. Large language models like GPT-4 have hundreds of billions of parameters.

**Training**: The process of adjusting parameters to minimize prediction error on training data.

**Inference**: Using a trained model to process new inputs.

**Fine-tuning**: Adapting a pre-trained model to a specific task or domain by training it on task-specific data.

### Transformers and the Attention Mechanism

The transformer architecture — the basis of essentially all modern large language models — processes sequential data by computing attention: how much each element should "attend to" every other element when producing an output.

For text: when processing the word "bank" in a sentence, the model attends to context words ("river" or "financial") to determine meaning. This attention mechanism enables capturing long-range dependencies in text that earlier architectures could not handle effectively.

---

## 10.3 Large Language Models: What They Are and Aren't

Large Language Models (LLMs) — GPT-4, Claude, Gemini, LLaMA, and their successors — are the most consequential AI development for OSINT practice. Understanding what they actually are prevents both underuse and overuse.

### What LLMs Actually Do

An LLM is a statistical model of language. Given input text, it generates continuation text by sampling from a probability distribution over vocabulary tokens, conditioned on the input.

This sounds reductive, but the capability that emerges from this simple objective, trained on enormous text corpora, is remarkable:

- Answering questions based on knowledge embedded in training data
- Summarizing, translating, and rewriting text
- Extracting structured information from unstructured text
- Generating plausible text in specified styles and formats
- Reasoning through problems when prompted appropriately
- Analyzing code, mathematical expressions, and structured data

### What LLMs Are Not

**LLMs are not search engines.** They cannot retrieve specific facts from the internet in real-time (without tool integration). They generate text based on patterns in training data, not by looking things up.

**LLMs do not "know" facts with certainty.** They model what text about a topic typically looks like. If training data frequently says "X is the capital of France," the model will output "Paris" when asked — but not because it "knows" Paris is the capital in the way a database knows. It generates what sounds like a confident answer.

**LLMs hallucinate.** This is the most critical limitation for investigative use. LLMs will confidently generate plausible-sounding but factually incorrect text — invented citations, fabricated quotes, non-existent entities. The frequency and detectability of hallucination varies by model and task, but it is never zero.

**LLMs have knowledge cutoffs.** Training data ends at a specific date. Events after the cutoff are not known to the model without retrieval augmentation.

**LLMs can be manipulated.** Adversarial prompts can cause models to behave in unintended ways. Prompt injection attacks — embedding instructions in external content fed to the model — can manipulate model behavior.

### The Hallucination Problem for Investigators

Hallucination is not a bug being fixed — it is an inherent property of how these models generate text. For investigative use, this means:

**Never cite LLM-generated information as a primary source.** The LLM's claim that "John Smith was indicted in 2019" is not investigative evidence. It is a starting point for searching primary sources.

**Never use LLM-generated quotes.** A model asked to provide quotes from a person will generate plausible-sounding text that may not be anything the person ever said.

**Always verify dates, statistics, citations, and specific factual claims** generated by LLMs against primary sources before incorporating them in investigative products.

**LLMs are valuable for analysis, synthesis, and reasoning, not for factual retrieval.** The distinction is critical.

---

## 10.4 The Modern AI Ecosystem for OSINT

### Foundation Model Providers

**Anthropic Claude**: Strong analytical reasoning, long context windows (useful for processing large documents), and careful instruction-following. Claude's model family includes Opus (most capable), Sonnet (balanced), and Haiku (fastest/cheapest).

**OpenAI GPT-4/o-series**: Broad capability with excellent code generation and tool use. GPT-4 Vision (and o-series) adds multimodal capability.

**Google Gemini**: Tight integration with Google services and strong multimodal capability. Gemini Ultra is Google's most capable tier.

**Meta LLaMA**: Open-weight models that can be run locally, important for privacy-sensitive investigations where sending data to cloud APIs is unacceptable.

**Mistral**: European open-weight models with strong performance-to-cost ratio.

### Specialized AI Models Relevant to OSINT

**Computer Vision models**: CLIP (image-text matching), SAM (image segmentation), object detection models (YOLO variants), OCR models (Tesseract, PaddleOCR), face recognition models.

**Speech and Audio**: Whisper (OpenAI's speech-to-text), speaker diarization models.

**Multimodal models**: Models that process both text and images (Claude, GPT-4V, Gemini) enable direct image analysis for OSINT.

**Entity extraction models**: Fine-tuned NLP models for named entity recognition (spaCy, Hugging Face NLP models).

**Translation models**: NLLB (Meta), M2M-100, and cloud translation APIs enable processing non-English content.

---

## 10.5 Using LLMs in Investigative Practice

The most productive way to use LLMs in OSINT is as an analytical augmentation layer — accelerating tasks that are tedious at human scale while maintaining human judgment for all significant conclusions.

### Appropriate LLM Use Cases

**Document summarization**: Processing and summarizing lengthy documents (court filings, SEC documents, news archives) faster than a human can read them. Always verify key claims against the original.

**Entity and relationship extraction**: Identifying named entities and relationships from unstructured text. LLMs can extract structured data from messy text at a scale impossible manually.

**Pattern identification in large text corpora**: Asking an LLM to identify themes, contradictions, or anomalies across multiple documents.

**Translation**: Translating foreign-language content for investigators who don't speak the relevant language. Machine translation has become good enough for investigative purposes, though significant documents warrant professional translation.

**Hypothesis generation**: Given established facts, asking an LLM to generate plausible hypotheses or alternative explanations. Not a substitute for human analysis, but a useful brainstorming aid.

**Report drafting**: Accelerating the production of investigation reports from analytical notes. All AI-drafted content must be carefully reviewed.

**Code generation**: For investigators with technical needs, LLMs accelerate writing Python scripts for data processing, API queries, and analysis automation.

```python
import anthropic

client = anthropic.Anthropic()

def extract_entities_from_text(text, entity_types=None):
    """
    Use Claude to extract named entities and relationships from investigative text
    """
    entity_types = entity_types or ['Person', 'Organization', 'Location', 'Date', 'Financial Amount']

    prompt = f"""You are an OSINT analyst extracting structured information from text.

Extract all named entities from the following text. For each entity, provide:
- Entity type ({', '.join(entity_types)})
- Entity name (as it appears in text)
- Context (brief quote showing how it appears)
- Relationships to other entities mentioned

Text to analyze:
{text}

Output as a structured list. Be precise and conservative — only extract entities clearly mentioned in the text. Do not infer or add information not present."""

    message = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{"role": "user", "content": prompt}]
    )

    return message.content[0].text

def summarize_document(document_text, focus_questions=None):
    """
    Summarize a document with focus on specific investigative questions
    """
    focus = ""
    if focus_questions:
        focus = f"\n\nPay particular attention to:\n" + "\n".join(f"- {q}" for q in focus_questions)

    prompt = f"""Summarize the following document for an investigative analyst.

Provide:
1. Document type and source (if apparent)
2. Key findings and facts
3. Significant persons, organizations, and locations mentioned
4. Dates and timeline of events
5. Any contradictions, unusual statements, or areas requiring verification{focus}

Note where important information appears uncertain or requires verification.

Document:
{document_text[:8000]}"""  # Claude's context window handles much more, but truncating for demo

    message = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{"role": "user", "content": prompt}]
    )

    return message.content[0].text

def generate_alternative_hypotheses(confirmed_facts, primary_hypothesis):
    """
    Use AI to generate competing hypotheses for ACH analysis
    """
    facts_text = "\n".join(f"- {f}" for f in confirmed_facts)

    prompt = f"""You are an intelligence analyst conducting Analysis of Competing Hypotheses (ACH).

Confirmed facts:
{facts_text}

Primary hypothesis:
{primary_hypothesis}

Generate 3-5 alternative hypotheses that:
1. Are consistent with the available facts
2. Represent genuinely different explanations
3. Include at least one "null hypothesis" or innocent explanation
4. Include a hypothesis that considers deliberate deception

For each alternative hypothesis, note:
- Which facts support it
- Which facts are inconsistent with it
- What additional evidence would confirm or refute it

Be analytical and avoid reaching conclusions beyond what the facts support."""

    message = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{"role": "user", "content": prompt}]
    )

    return message.content[0].text
```

### Inappropriate LLM Use Cases

**Primary source research**: Using LLMs to answer "where does John Smith live?" rather than searching public records. The LLM may fabricate a plausible-sounding but false answer.

**Factual citation**: Using an LLM's claim as evidence of a fact without verification.

**Identifying individuals in images**: LLMs with vision capability should not be used to identify specific individuals from photographs without extreme verification caution.

**Generating quotes or statements attributed to real people**: This creates serious misrepresentation risks.

---

## 10.6 Multimodal AI for OSINT

Multimodal models that process both text and images have opened new OSINT capabilities:

**Image description and analysis**: Asking a vision model to describe image content, identify objects, read visible text, and characterize the scene.

**Document processing**: Processing scanned documents, photographs of text, or handwritten content.

**Visual geolocation assistance**: As discussed in Chapter 8, vision models can help identify geographic indicators in images.

**Video frame analysis**: Extracting and analyzing individual frames from video content.

```python
import anthropic
import base64
import httpx

def analyze_image_for_osint(image_source, investigation_context=None):
    """
    Analyze an image using Claude's vision capability for OSINT purposes
    image_source: URL or file path
    """
    client = anthropic.Anthropic()

    # Load image
    if image_source.startswith('http'):
        image_data = base64.standard_b64encode(
            httpx.get(image_source).content
        ).decode("utf-8")
        media_type = "image/jpeg"  # Adjust based on actual type
    else:
        with open(image_source, 'rb') as f:
            image_data = base64.b64encode(f.read()).decode('utf-8')
        # Detect media type from extension
        ext = image_source.lower().split('.')[-1]
        media_type = {'jpg': 'image/jpeg', 'jpeg': 'image/jpeg',
                     'png': 'image/png', 'gif': 'image/gif',
                     'webp': 'image/webp'}.get(ext, 'image/jpeg')

    context_instruction = ""
    if investigation_context:
        context_instruction = f"\n\nInvestigation context: {investigation_context}"

    message = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=[
            {
                "role": "user",
                "content": [
                    {
                        "type": "image",
                        "source": {
                            "type": "base64",
                            "media_type": media_type,
                            "data": image_data,
                        },
                    },
                    {
                        "type": "text",
                        "text": f"""Analyze this image from an open-source intelligence perspective.

Identify and describe:
1. Visible text (signs, labels, documents, licenses plates — read exactly as shown)
2. People present (physical descriptions only, do not attempt to identify)
3. Objects and items of potential investigative interest
4. Location indicators (geography, climate, cultural markers, infrastructure)
5. Time indicators (weather, lighting, visible dates, seasonal markers)
6. Vehicles (type, make if identifiable, license plate format/region)
7. Organizational/brand indicators (logos, uniforms, markings)

Note the quality and reliability of each observation. Do not speculate beyond what is clearly visible.{context_instruction}"""
                    }
                ],
            }
        ],
    )

    return message.content[0].text
```

---

## 10.7 AI Bias and Its Investigative Implications

AI systems encode the biases present in their training data. For investigators, this creates serious risks:

**Demographic bias**: Face recognition systems have documented higher error rates for women and people of color. Using biased AI for identity verification in investigations creates systematic unfairness.

**Geographic bias**: AI models trained primarily on English-language Western content have reduced accuracy for non-English content, non-Western cultural contexts, and less-represented geographic regions.

**Temporal bias**: AI models trained on historical data reflect historical patterns and biases. These may not apply to current situations.

**Selection bias**: Training data is never a random sample of all possible inputs. The populations and scenarios overrepresented in training data receive better performance.

**Amplification bias**: AI models can amplify biases present in their training data to produce outputs more extreme than the training patterns.

**Practical mitigations**:
- Use AI tools as hypothesis generators, not conclusion producers
- Verify AI findings through human review and primary source corroboration
- Be explicitly skeptical of AI outputs involving populations where you know the model has limited training data
- Test AI tools on examples where the correct answer is known before using them for unknown cases
- Document AI tool use and its limitations in investigative methodology notes

---

## 10.8 Responsible AI Use in Investigations

### The Human Oversight Requirement

For every investigative conclusion that will be acted upon — that will be included in a report, shared with a client, used as the basis for legal action, or published — a human must have:

1. Reviewed the AI output
2. Verified supporting evidence against primary sources
3. Applied professional judgment about confidence and accuracy
4. Taken personal responsibility for the finding

"The AI said so" is not an acceptable basis for a professional investigative conclusion.

### Prompt Design for Investigative Use

Effective use of LLMs in investigations requires careful prompt design:

**Be explicit about the task**: Vague prompts produce vague outputs. "Tell me about this company" is worse than "Extract all disclosed related-party transactions from this proxy statement."

**Specify output format**: "Provide a bullet-pointed list of..." or "Respond in JSON format with fields..." produces more usable outputs than unconstrained generation.

**Instruct for uncertainty disclosure**: "Note where you are uncertain or where the text does not clearly support your conclusion."

**Provide context**: "You are analyzing an SEC proxy statement. Identify..." gives the model relevant context for accurate analysis.

**Calibrate confidence**: "Rate your confidence in each finding as high, medium, or low."

### Data Privacy in AI Tool Use

Sending sensitive investigation data to cloud AI APIs creates privacy and confidentiality considerations:

- Client data, subject data, and investigative findings fed to cloud APIs are transmitted to and processed by the API provider
- API providers' data handling policies vary in how they treat input data
- Sensitive investigations may require use of locally-deployed models (LLaMA, Mistral) rather than cloud APIs
- API usage agreements should be reviewed for data retention and training provisions

---

## Summary

AI has become an unavoidable component of modern OSINT practice. Understanding AI fundamentals — what machine learning is, how large language models work, what their limitations are — enables investigators to use these tools effectively and avoid their failure modes.

LLMs are powerful analytical aids for document processing, entity extraction, hypothesis generation, and translation. They are not reliable factual sources and hallucinate with meaningful frequency. Human oversight of AI outputs is mandatory for all consequential investigative work.

Multimodal models extend AI capability to image and document analysis. AI bias is a real consideration that requires explicit attention, particularly for investigations involving populations or geographies underrepresented in training data.

The responsible use of AI in investigations treats AI as an accelerant for human analysis, not a replacement for it.

---

## Common Mistakes and Pitfalls

- **Citing LLM outputs as facts**: AI-generated factual claims require verification against primary sources
- **Trusting AI confidence signals**: Models that output "I'm confident that..." are not necessarily more accurate
- **Ignoring hallucination risk for dates, statistics, and citations**: These specific output types have elevated hallucination rates
- **Using cloud AI for sensitive data without reviewing data policies**: Investigative data requires confidentiality; cloud API data handling varies
- **Treating AI limitation disclosures as disclaimers to skip**: Model limitations are operational constraints, not legal boilerplate
- **Not specifying output format**: Unstructured AI outputs are harder to use and verify than structured ones

---

## Further Reading

- Anthropic model documentation — model capabilities and limitations
- OpenAI GPT-4 technical report — capability and evaluation documentation
- Emily Bender et al., "On the Dangers of Stochastic Parrots" — the foundational academic critique of LLMs
- Timnit Gebru and colleagues' work on AI bias
- AI and democracy/misinformation research from the Partnership on AI and AI Now Institute
