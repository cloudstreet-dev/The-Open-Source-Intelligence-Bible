# Chapter 25: Emerging Technologies and Future AI Capabilities

## Learning Objectives

By the end of this chapter, you will be able to:
- Assess current AI capabilities relevant to OSINT and their trajectory
- Apply multimodal AI systems to image, audio, and video analysis tasks
- Understand the implications of generative AI for both investigation and disinformation
- Evaluate emerging data sources (satellite frequency, IoT, biometric data)
- Anticipate how quantum computing and advanced cryptography affect OSINT
- Design forward-compatible investigation workflows that leverage emerging tools

---

## 25.1 The AI Inflection Point in OSINT

The period from 2022 to 2025 witnessed the most rapid capability expansion in AI history. Large language models transitioned from research curiosities to production tools; multimodal models gained the ability to analyze images, audio, and video; autonomous agents demonstrated the ability to use tools and execute multi-step plans. These changes are not incremental — they represent qualitative shifts in what an investigator armed with AI can accomplish.

This chapter maps current capabilities, traces where the trajectory points, and explores the implications for OSINT practitioners on both sides of the investigator/subject relationship.

---

## 25.2 Multimodal AI for OSINT

Modern AI models can reason across text, images, audio, and video — capabilities with direct OSINT applications.

### Image Analysis

```python
import anthropic
import base64
from pathlib import Path
from typing import Union, Dict, List
import requests

def analyze_image_for_osint(image_source: Union[str, bytes], analysis_focus: str = "general") -> str:
    """
    Analyze an image using Claude's vision capability
    image_source: URL string, file path string, or bytes
    analysis_focus: "general", "geolocation", "document", "crowd", "infrastructure"
    """
    client = anthropic.Anthropic()

    # Prepare image content
    if isinstance(image_source, str) and image_source.startswith('http'):
        # URL-referenced image
        image_content = {
            "type": "image",
            "source": {
                "type": "url",
                "url": image_source
            }
        }
    else:
        # File path or bytes
        if isinstance(image_source, str):
            image_bytes = Path(image_source).read_bytes()
        else:
            image_bytes = image_source

        # Detect media type
        if image_bytes[:4] == b'\x89PNG':
            media_type = "image/png"
        elif image_bytes[:2] == b'\xff\xd8':
            media_type = "image/jpeg"
        elif image_bytes[:4] == b'RIFF':
            media_type = "image/webp"
        elif image_bytes[:3] == b'GIF':
            media_type = "image/gif"
        else:
            media_type = "image/jpeg"  # Default

        image_content = {
            "type": "image",
            "source": {
                "type": "base64",
                "media_type": media_type,
                "data": base64.standard_b64encode(image_bytes).decode('utf-8')
            }
        }

    # Build analysis prompt based on focus
    prompts = {
        "general": """Analyze this image from an OSINT investigation perspective. Identify:
1. Location indicators (signs, landmarks, architecture, vegetation, terrain)
2. Time indicators (shadows, weather, seasonal clues, visible dates/times)
3. People (approximate number, visible identifiers, activities)
4. Infrastructure (buildings, roads, vehicles, equipment)
5. Text content (signs, labels, documents)
6. Any unusual or significant elements

Be specific and evidence-based. Note confidence level for each observation.""",

        "geolocation": """Analyze this image to determine geographic location and time. Focus on:
1. Architecture style, building materials, construction type
2. Vegetation, trees, plants, landscape features
3. Street furniture: signs, mailboxes, utility poles, vehicles
4. Language visible in any text
5. Sky conditions, sun angle, shadows
6. Background geographic features (mountains, water, skyline)
7. Infrastructure patterns (road markings, traffic signals)

For each clue, explain what it indicates about location. Provide your best geographic estimate with confidence level.""",

        "document": """Analyze this document image. Extract:
1. All readable text content
2. Document type and format
3. Dates, reference numbers, signatures
4. Organizational identifiers (logos, letterheads, stamps)
5. Redaction patterns (what appears to be hidden)
6. Document authenticity indicators
7. Metadata visible in the image

Organize extracted text with context about its location in the document.""",

        "infrastructure": """Analyze this infrastructure/facility image:
1. Facility type and function
2. Security measures visible
3. Access control systems
4. Utility systems (power, water, communications)
5. Vehicles and equipment present
6. Operational status indicators
7. Changes from expected/normal state

Note any indicators of specific organizations or government entities."""
    }

    prompt = prompts.get(analysis_focus, prompts["general"])

    response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=2000,
        messages=[{
            "role": "user",
            "content": [
                image_content,
                {"type": "text", "text": prompt}
            ]
        }]
    )

    return response.content[0].text


def batch_image_analysis(image_paths: List[str], focus: str = "general") -> List[Dict]:
    """
    Analyze multiple images in sequence
    """
    results = []

    for path in image_paths:
        print(f"Analyzing: {path}")
        try:
            analysis = analyze_image_for_osint(path, focus)
            results.append({
                'path': path,
                'focus': focus,
                'analysis': analysis,
                'status': 'success'
            })
        except Exception as e:
            results.append({
                'path': path,
                'focus': focus,
                'error': str(e),
                'status': 'error'
            })

    return results


def compare_images_for_changes(image1_source, image2_source, context: str = "") -> str:
    """
    Compare two images to detect changes — useful for monitoring facilities,
    social media profile photos, or document revisions
    """
    client = anthropic.Anthropic()

    def prepare_image(source):
        if isinstance(source, str) and source.startswith('http'):
            return {"type": "image", "source": {"type": "url", "url": source}}
        else:
            if isinstance(source, str):
                img_bytes = Path(source).read_bytes()
            else:
                img_bytes = source
            return {
                "type": "image",
                "source": {
                    "type": "base64",
                    "media_type": "image/jpeg",
                    "data": base64.standard_b64encode(img_bytes).decode('utf-8')
                }
            }

    prompt = f"""Compare these two images and identify all changes between them.

{f'Context: {context}' if context else ''}

Describe:
1. What has CHANGED between image 1 and image 2
2. What has been ADDED to image 2
3. What has been REMOVED from image 2
4. What is UNCHANGED
5. The significance of the changes in context

Be specific about the location and nature of each change."""

    response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=1500,
        messages=[{
            "role": "user",
            "content": [
                prepare_image(image1_source),
                {"type": "text", "text": "IMAGE 1 (earlier):"},
                prepare_image(image2_source),
                {"type": "text", "text": "IMAGE 2 (later):"},
                {"type": "text", "text": prompt}
            ]
        }]
    )

    return response.content[0].text
```

### Video and Audio Analysis

Video analysis is increasingly valuable for OSINT — social media videos contain location data, behavioral signals, and verifiable content. Audio analysis applies to leaked recordings, public testimony, and propaganda content.

```python
import anthropic
import subprocess
import tempfile
from pathlib import Path
from typing import Dict, List
import os

def extract_video_frames(video_path: str, fps: float = 1.0) -> List[str]:
    """
    Extract frames from video at specified frames-per-second rate
    Requires: ffmpeg installed on system
    """
    output_dir = tempfile.mkdtemp()
    output_pattern = os.path.join(output_dir, 'frame_%04d.jpg')

    cmd = [
        'ffmpeg',
        '-i', video_path,
        '-vf', f'fps={fps}',
        '-q:v', '2',
        output_pattern,
        '-hide_banner', '-loglevel', 'error'
    ]

    try:
        subprocess.run(cmd, check=True, capture_output=True)
        frames = sorted(Path(output_dir).glob('*.jpg'))
        return [str(f) for f in frames]
    except subprocess.CalledProcessError as e:
        raise RuntimeError(f"FFmpeg error: {e.stderr.decode()}")


def transcribe_audio(audio_path: str) -> str:
    """
    Transcribe audio using OpenAI Whisper (local) or API
    Requires: pip install openai-whisper
    """
    try:
        import whisper
        model = whisper.load_model("base")
        result = model.transcribe(audio_path)
        return result.get('text', '')
    except ImportError:
        # Fallback: use OpenAI API
        import openai, os
        client = openai.OpenAI(api_key=os.getenv('OPENAI_API_KEY', ''))

        with open(audio_path, 'rb') as audio_file:
            transcript = client.audio.transcriptions.create(
                model="whisper-1",
                file=audio_file
            )
        return transcript.text


def analyze_video_for_osint(video_path: str) -> Dict:
    """
    Complete video analysis pipeline for OSINT
    """
    analysis = {
        'video_path': video_path,
        'transcript': None,
        'frame_analyses': [],
        'summary': None
    }

    # 1. Extract audio and transcribe
    print("Extracting and transcribing audio...")
    audio_path = video_path.replace('.mp4', '_audio.mp3')
    try:
        subprocess.run([
            'ffmpeg', '-i', video_path, '-q:a', '0', '-map', 'a',
            audio_path, '-hide_banner', '-loglevel', 'error'
        ], check=True, capture_output=True)

        if os.path.exists(audio_path):
            analysis['transcript'] = transcribe_audio(audio_path)
    except Exception as e:
        print(f"Audio extraction error: {e}")

    # 2. Extract key frames for visual analysis
    print("Extracting frames...")
    try:
        frames = extract_video_frames(video_path, fps=0.5)  # One frame per 2 seconds
        analysis['frame_count'] = len(frames)

        # Analyze a sample of frames
        for frame in frames[:10]:  # Limit to 10 frames
            frame_analysis = analyze_image_for_osint(frame, "general")
            analysis['frame_analyses'].append({
                'frame': frame,
                'analysis': frame_analysis
            })
    except Exception as e:
        print(f"Frame extraction error: {e}")

    # 3. Synthesize findings with AI
    if analysis['transcript'] or analysis['frame_analyses']:
        client = anthropic.Anthropic()

        synthesis_prompt = f"""Synthesize OSINT intelligence from this video analysis:

TRANSCRIPT:
{analysis['transcript'][:2000] if analysis['transcript'] else 'No audio transcript available'}

VISUAL ANALYSIS SAMPLES:
{chr(10).join([f"Frame {i+1}: {fa['analysis'][:300]}" for i, fa in enumerate(analysis['frame_analyses'][:5])])}

Provide:
1. KEY FINDINGS: What is most significant about this video?
2. LOCATION ASSESSMENT: Where was this recorded? (use visual and audio cues)
3. TEMPORAL ASSESSMENT: When was this recorded?
4. AUTHENTICITY INDICATORS: Any signs of manipulation or staging?
5. INTELLIGENCE VALUE: What does this add to an investigation?"""

        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1500,
            messages=[{"role": "user", "content": synthesis_prompt}]
        )

        analysis['summary'] = response.content[0].text

    return analysis
```

---

## 25.3 Generative AI: Both Threat and Tool

Generative AI creates OSINT challenges by enabling:

**Synthetic media**: AI-generated images (DALL-E, Midjourney, Stable Diffusion), deepfake video (realistic face swaps), and cloned voice (voice synthesis from short recordings) can fabricate evidence that passes casual inspection.

**Automated disinformation at scale**: What previously required a human farm can now be executed by LLM agents generating contextually appropriate content across thousands of fake accounts.

**Synthetic persona infrastructure**: AI can generate consistent, contextually rich fake identities — with plausible backstories, writing styles, and engagement patterns — far more convincingly than the crude bots of previous years.

### Detection Approaches

```python
import requests
from typing import Dict
import re

def check_image_ai_generation(image_path: str) -> Dict:
    """
    Check image for AI generation indicators using multiple detection approaches
    """
    results = {
        'ai_generation_indicators': [],
        'authenticity_signals': [],
        'conclusion': 'UNCERTAIN',
        'confidence': 0.0
    }

    # 1. Metadata analysis
    try:
        import subprocess
        exiftool_result = subprocess.run(
            ['exiftool', '-json', image_path],
            capture_output=True, text=True, timeout=10
        )
        if exiftool_result.returncode == 0:
            import json
            metadata = json.loads(exiftool_result.stdout)
            if metadata:
                meta = metadata[0]

                # Check for AI generation metadata
                software = meta.get('Software', '')
                if any(tool in software.lower() for tool in ['stable diffusion', 'midjourney', 'dall-e']):
                    results['ai_generation_indicators'].append(f"Software metadata: {software}")

                # Check for typical DSLR/phone camera metadata
                camera_fields = ['Make', 'Model', 'LensModel', 'GPS*', 'DateTimeOriginal']
                for field in camera_fields:
                    if meta.get(field):
                        results['authenticity_signals'].append(f"Camera metadata present: {field}")

    except FileNotFoundError:
        results['ai_generation_indicators'].append("ExifTool not available — metadata unchecked")
    except Exception as e:
        pass

    # 2. Hive Moderation API (AI detection service)
    try:
        import os
        hive_key = os.getenv('HIVE_API_KEY', '')
        if hive_key:
            with open(image_path, 'rb') as f:
                response = requests.post(
                    'https://api.thehive.ai/api/v2/task/sync',
                    headers={'Authorization': f'Token {hive_key}'},
                    files={'image': f},
                    timeout=30
                )
                if response.status_code == 200:
                    data = response.json()
                    # Parse Hive AI detection results
                    # Structure depends on Hive API version
                    results['hive_response'] = data

    except Exception as e:
        pass

    # 3. Statistical analysis (simplified)
    # Real-world AI detection uses much more sophisticated signal analysis
    # e.g., checking for GAN artifacts in DCT frequency domain

    indicator_count = len(results['ai_generation_indicators'])
    authenticity_count = len(results['authenticity_signals'])

    if indicator_count > 0 and authenticity_count == 0:
        results['conclusion'] = 'LIKELY AI GENERATED'
        results['confidence'] = min(0.8, 0.3 * indicator_count)
    elif authenticity_count > 2 and indicator_count == 0:
        results['conclusion'] = 'LIKELY AUTHENTIC'
        results['confidence'] = min(0.7, 0.2 * authenticity_count)
    else:
        results['conclusion'] = 'UNCERTAIN — Manual verification recommended'
        results['confidence'] = 0.3

    return results


def detect_deepfake_indicators(video_path: str) -> Dict:
    """
    Detect potential deepfake indicators in video
    Note: Reliable deepfake detection requires specialized ML models
    This provides a framework for manual inspection assistance
    """
    indicators = {
        'technical_flags': [],
        'behavioral_flags': [],
        'inconsistency_flags': [],
        'manual_checks': []
    }

    # Frame extraction for analysis
    try:
        import subprocess
        frames_cmd = [
            'ffprobe', '-v', 'quiet', '-print_format', 'json',
            '-show_streams', video_path
        ]
        result = subprocess.run(frames_cmd, capture_output=True, text=True, timeout=15)

        if result.returncode == 0:
            import json
            probe_data = json.loads(result.stdout)
            video_streams = [s for s in probe_data.get('streams', []) if s.get('codec_type') == 'video']

            if video_streams:
                stream = video_streams[0]
                fps_parts = stream.get('r_frame_rate', '0/1').split('/')
                fps = int(fps_parts[0]) / int(fps_parts[1]) if len(fps_parts) == 2 else 0

                if fps not in [24, 25, 29.97, 30, 60]:
                    indicators['technical_flags'].append(f"Unusual frame rate: {fps}")

    except Exception as e:
        pass

    # Manual inspection checklist
    indicators['manual_checks'] = [
        "Check eye blinking rate — deepfakes often have reduced or abnormal blinking",
        "Examine hairline and hair movement for artifacts",
        "Look for face boundary inconsistencies, especially under motion",
        "Check lighting consistency — shadows and highlights should match source",
        "Examine teeth and mouth during speech for rendering artifacts",
        "Check for inconsistent skin texture or unnatural smoothness",
        "Verify audio-visual synchronization carefully",
        "Look for temporal flickering in specific facial regions",
        "Cross-reference with known authentic footage of the same person"
    ]

    return indicators
```

---

## 25.4 Agentic AI for OSINT

Agentic AI systems — LLMs that can plan, use tools, and execute multi-step tasks autonomously — represent the next evolution in AI-assisted investigation. Rather than answering individual questions, an agentic system can:

1. Receive a high-level investigation goal
2. Decompose it into subtasks
3. Execute web searches, API calls, and data processing
4. Synthesize findings across steps
5. Identify next steps based on intermediate results
6. Produce structured final reports

```python
import anthropic
import json
from typing import Dict, List, Callable, Any

# Tool definitions for the agentic OSINT investigator
OSINT_TOOLS = [
    {
        "name": "web_search",
        "description": "Search the web for information about a topic, person, company, or domain",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "Search query"},
                "date_range": {"type": "string", "description": "Optional: 'week', 'month', 'year'"}
            },
            "required": ["query"]
        }
    },
    {
        "name": "dns_lookup",
        "description": "Perform DNS lookups for a domain",
        "input_schema": {
            "type": "object",
            "properties": {
                "domain": {"type": "string", "description": "Domain to look up"},
                "record_types": {
                    "type": "array",
                    "items": {"type": "string"},
                    "description": "DNS record types to query: A, AAAA, MX, NS, TXT, CNAME"
                }
            },
            "required": ["domain"]
        }
    },
    {
        "name": "whois_lookup",
        "description": "Get WHOIS registration data for a domain",
        "input_schema": {
            "type": "object",
            "properties": {
                "domain": {"type": "string", "description": "Domain to look up"}
            },
            "required": ["domain"]
        }
    },
    {
        "name": "company_lookup",
        "description": "Look up company registration data from OpenCorporates",
        "input_schema": {
            "type": "object",
            "properties": {
                "company_name": {"type": "string"},
                "jurisdiction": {"type": "string", "description": "Two-letter country/state code"}
            },
            "required": ["company_name"]
        }
    },
    {
        "name": "analyze_image",
        "description": "Analyze an image for OSINT-relevant content",
        "input_schema": {
            "type": "object",
            "properties": {
                "image_url": {"type": "string", "description": "URL of image to analyze"},
                "focus": {"type": "string", "description": "Analysis focus: general, geolocation, document"}
            },
            "required": ["image_url"]
        }
    },
    {
        "name": "report_finding",
        "description": "Record a significant finding in the investigation report",
        "input_schema": {
            "type": "object",
            "properties": {
                "category": {"type": "string"},
                "finding": {"type": "string"},
                "confidence": {"type": "string", "enum": ["HIGH", "MEDIUM", "LOW"]},
                "source": {"type": "string"},
                "evidence_url": {"type": "string"}
            },
            "required": ["category", "finding", "confidence", "source"]
        }
    }
]


class AgenticOSINTInvestigator:
    """
    Agentic OSINT investigator using Claude with tool use
    """

    def __init__(self, tool_implementations: Dict[str, Callable]):
        self.client = anthropic.Anthropic()
        self.tools = OSINT_TOOLS
        self.tool_implementations = tool_implementations
        self.findings: List[Dict] = []
        self.investigation_log: List[Dict] = []

    def investigate(self, goal: str, max_turns: int = 20) -> Dict:
        """
        Run an agentic investigation toward a specified goal
        """
        system_prompt = """You are an experienced OSINT investigator. Your role is to systematically investigate the given topic using available tools, gather evidence from multiple sources, and compile verified findings.

Investigation principles:
- Verify claims across multiple independent sources before treating as confirmed
- Distinguish HIGH confidence (multiple corroborating sources) from MEDIUM (single source) from LOW (inference or partial data)
- Use tools methodically — don't repeat searches without good reason
- Report findings as you discover them using report_finding
- Stop when you have a comprehensive picture or have exhausted available avenues

Do not speculate beyond what the evidence supports. Do not access systems without authorization. Focus on passive, public data sources only."""

        messages = [{"role": "user", "content": goal}]

        for turn in range(max_turns):
            response = self.client.messages.create(
                model="claude-sonnet-4-6",
                max_tokens=4096,
                system=system_prompt,
                tools=self.tools,
                messages=messages
            )

            self.investigation_log.append({
                'turn': turn + 1,
                'stop_reason': response.stop_reason,
                'content_blocks': len(response.content)
            })

            # Process response
            messages.append({"role": "assistant", "content": response.content})

            if response.stop_reason == "end_turn":
                # Investigation complete
                break

            if response.stop_reason == "tool_use":
                # Execute tool calls
                tool_results = []

                for block in response.content:
                    if block.type == "tool_use":
                        tool_name = block.name
                        tool_input = block.input

                        print(f"[Tool] {tool_name}: {json.dumps(tool_input)[:100]}")

                        # Execute tool
                        if tool_name in self.tool_implementations:
                            try:
                                result = self.tool_implementations[tool_name](**tool_input)

                                # Special handling for report_finding
                                if tool_name == "report_finding":
                                    self.findings.append(tool_input)

                            except Exception as e:
                                result = {"error": str(e)}
                        else:
                            result = {"error": f"Tool {tool_name} not implemented"}

                        tool_results.append({
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": json.dumps(result) if not isinstance(result, str) else result
                        })

                messages.append({"role": "user", "content": tool_results})

        # Extract final text response
        final_text = ""
        if messages and messages[-1]["role"] == "assistant":
            for block in messages[-1].get("content", []):
                if hasattr(block, "type") and block.type == "text":
                    final_text = block.text
                    break

        return {
            'goal': goal,
            'findings': self.findings,
            'turns_used': len(self.investigation_log),
            'final_summary': final_text,
            'log': self.investigation_log
        }
```

---

## 25.5 Emerging Data Sources

### Satellite and Aerial Imagery Trends

Commercial satellite imagery has democratized geospatial intelligence. Planet Labs now offers near-daily global coverage at 3m resolution; Maxar provides sub-meter imagery on demand. Future trends:

- **SAR (Synthetic Aperture Radar)**: Penetrates cloud cover; detects changes in surface structure. Increasingly available commercially through ICEYE, Capella Space.
- **Hyperspectral imaging**: Identifies specific materials, vegetation health, and surface composition beyond what RGB imagery reveals.
- **Thermal imaging**: Detects heat signatures useful for facility monitoring, crowd estimation, and industrial activity.

### IoT and Sensor Data

Connected devices produce massive amounts of publicly accessible data:

- **Weather stations**: Personal weather stations publish hyperlocal data through Weather Underground and similar platforms — useful for verifying claims about conditions at specific locations and times.
- **Air quality sensors**: PurpleAir's network of 20,000+ sensors provides near-real-time air quality across the globe — useful for detecting industrial incidents.
- **Ship and aircraft transponders**: AIS and ADS-B were covered in Chapter 8; the trend is toward higher coverage density and longer retention.

### Regulatory and Compliance Data Growth

Regulatory requirements drive disclosure:

- **SEC cyber incident reporting**: Since December 2023, material cybersecurity incidents must be disclosed via 8-K within 4 business days
- **CISA critical infrastructure reporting**: CIRCIA creates reporting requirements for critical infrastructure sectors
- **FinCEN Beneficial Ownership Registry**: The CTA creates a federal beneficial ownership database (access currently restricted to authorized users)
- **EU AI Act**: Requires documentation of high-risk AI systems — potentially a new source for understanding deployed AI capabilities

---

## 25.6 Quantum Computing Implications

Quantum computing is not yet a practical OSINT tool, but its eventual maturity creates implications:

**Cryptographic impact**: Quantum computers using Shor's algorithm can theoretically break RSA and elliptic curve cryptography. This affects:
- Encrypted data that adversaries have stored for future decryption ("harvest now, decrypt later")
- Certificate Transparency infrastructure
- Signal verification of documents and communications

**Quantum sensing**: Quantum sensors can detect signals too weak for classical sensors — potentially enabling detection of previously hidden infrastructure.

**Practical timeline**: Cryptographically relevant quantum computers remain years away; current quantum advantage is in specialized optimization problems, not general-purpose computation. NIST's post-quantum cryptography standards (2024) define the migration path.

---

## Summary

The AI capability frontier is advancing faster than investigative practices can adapt. The practitioners who will be most effective in the near future are those who:

1. **Stay close to model capabilities**: LLMs, vision models, and agent frameworks evolve rapidly. What required custom ML two years ago is now a single API call.

2. **Understand both tool and threat**: The same generative AI capabilities that accelerate investigation also enable synthetic media, deepfakes, and automated disinformation. Forensic verification skills must keep pace.

3. **Design agentic workflows**: The shift from AI-as-assistant to AI-as-investigator is underway. Investigators who design effective AI agent workflows will accomplish in hours what previously required days.

4. **Apply structured verification**: As AI-generated content becomes indistinguishable from authentic content through visual inspection alone, forensic verification through metadata, provenance, and cross-source corroboration becomes more critical.

---

## Common Mistakes and Pitfalls

- **Over-trusting AI analysis**: Vision models make confident errors; always cross-verify critical findings through independent methods
- **Ignoring AI content generation capabilities when assessing adversaries**: Organizations increasingly underestimate adversary disinformation capacity
- **Treating quantum risk as distant**: "Harvest now, decrypt later" attacks are happening now against encrypted data that will matter in 5-10 years
- **Failing to account for model cutoffs**: AI models have training data cutoffs — they cannot know about recent events and may confidently state outdated information
- **Automation without oversight**: Agentic AI systems that operate without human checkpoints will make consequential errors without detection

---

## Further Reading

- Anthropic model documentation — current capabilities and limitations
- NIST AI RMF (AI Risk Management Framework) — framework for responsible AI deployment
- IARPA SMART program — government research on geospatial intelligence
- Partnership on AI — multi-stakeholder AI governance research
- Stanford Center for AI Safety — risk and capability assessments
- IEEE Spectrum — emerging technology coverage
