---
layout: post
title: "Solving Transcription Errors for Technical Jargon: Combining Fuzzy Matching and LLMs"
date: 2026-08-03
category: ai-workflows
tags: [asr, whisper, fuzzy matching, llm, ollama]
description: "Whisper mis-hears technical jargon: XSS becomes 'excess', Burp Suite becomes 'burp sweet'. How role-scoped keyword extraction, rapidfuzz, and two deliberately narrow local LLM calls fixed it without reaching for a bigger model."
---

If you've ever built an AI product, you know that feeling. You get the core architecture running, the UI is looking sleek, and you run your first real-world test. Then, you look at the output, and your heart sinks.

A few months ago, my team and I were building an AI-powered interview practice platform designed specifically for software engineering and cybersecurity candidates. The idea was simple: simulate rigorous technical interviews, analyze a user's spoken responses in real-time, and provide detailed feedback on their technical accuracy and communication skills.

But to do that, we needed perfect transcripts. And that's where we hit a wall—a wall made of mangled technical jargon.

## The Problem: When ASR Meets Tech Speak

For the transcription layer, we run speech-to-text through `faster-whisper`, a CTranslate2 reimplementation of OpenAI's Whisper models. Real-time voice means every second of latency is a second the candidate is staring at a blinking mic icon, so we run the lightweight `base` model on GPU rather than the largest, most accurate variant—a deliberate speed-over-accuracy trade that pays off in general conversation but costs us on anything niche.

```python
# main.py
model = WhisperModel("base", device="cuda")  # change to "cpu" if needed
```

For general conversation, `base` is perfectly usable. But throw a room full of software engineers, cybersecurity candidates, and Android security researchers at it, and it still stumbles on domain jargon.

Because even the best ASR models are trained on general web text, they try to force-fit niche, highly specific terms into common English phrases. In practice, that means the kind of misfire this whole pipeline exists to catch:
*   "XSS vulnerabilities" heard as *"excess vulnerabilities"*
*   "Burp Suite" heard as *"burp sweet"*
*   "Zygote" (the Android process, not the biology term) heard as *"why goat"*
*   "SELinux" heard as *"see Linux"*
*   "Mali GPU" heard as *"molly GPU"*

If we pushed transcripts like these straight to our downstream evaluation and feedback agents, the output would be garbage. We couldn't just tell users to "speak clearer." We had to build a translation layer from "ASR-English" to "Tech-English."

## Phase 1: The Fuzzy Matching Layer

Our first instinct was a deterministic safety net—but not a static, hand-maintained dictionary. A glossary of every Kubernetes term, every OWASP category, every Android internals name would be permanently out of date and permanently incomplete. Instead, we generate the reference term list *per transcript*, scoped to whichever interview track the candidate is in.

```python
# keyword_extractor.py — role-scoped domain reference
ROLE_CONTEXT = {
    "pentester": """
Cybersecurity penetration testing interview.

Domain reference (do not force usage):
SQL injection, XSS, CSRF, SSTI, RCE,
Burp Suite, Nmap, Metasploit,
authentication, authorization, JWT.
""",
    "fullstack": """
Full-stack developer interview.

Domain reference (do not force usage):
React, Django, FastAPI,
REST, GraphQL,
SQL, NoSQL, APIs.
""",
    "android_security": """
Android security interview.

Domain reference (do not force usage):
Binder, SELinux, Zygote,
Kernel, Mali GPU,
TrustZone, CVEs.
"""
}
```

A candidate picks their track once, at the start of the session, and that role travels with the transcript through every stage that follows. An LLM call reads the raw Whisper output plus the role's domain reference and pulls out a short list of technical terms that are already there or unambiguously implied—never inventing new ones. That list becomes the reference set for fuzzy matching.

```python
# fuzzy_reinforce.py
from rapidfuzz import process, fuzz

def reinforce_with_keywords(
    transcript: str,
    keywords: list[str],
    threshold: int = 88
) -> str:
    """
    Replace tokens with role-relevant keywords
    when phonetic similarity is high.
    """
    if not keywords:
        return transcript

    tokens = transcript.split()
    reinforced = []

    for token in tokens:
        match = process.extractOne(
            token, keywords, scorer=fuzz.partial_ratio
        )
        if match:
            keyword, score, _ = match
            if score >= threshold:
                reinforced.append(keyword)
                continue
        reinforced.append(token)

    return " ".join(reinforced)
```

The code's own comment calls this "phonetic similarity," though `partial_ratio` is really measuring string overlap, not sound. Close enough in practice—ASR's mistakes tend to produce text that's still visually/texturally close to the real word, even when it's phonetically inspired. Anything that scores 88 or higher against the extracted keyword list gets swapped in, token by token.

**The result:** a cheap, fast, deterministic pass that catches the obvious misses without ever needing a glossary someone has to maintain by hand.

## The Lingering Problem: Context is King

But token-level fuzzy matching has a ceiling. It doesn't read the sentence, it reads one word at a time against a keyword list—it has no way to know that "excess" should become "XSS" in a security context but stay exactly as "excess" in a budget conversation.

Role context helps here too, but only at the boundary of *what's even a candidate keyword in the first place*. If the session role is `fullstack`, "react" is one of the terms the extraction step is watching for, and a mangled ASR token near it is worth reinforcing toward "React." If the role is `pentester`, those same five letters are almost certainly just the verb—there's no "react" in the pentester domain reference, so nothing pulls a stray token toward capitalizing it.

That's the actual job role context does in this system: not resolving a sentence's meaning, but narrowing *which* words the rest of the pipeline is even allowed to bias toward.

## Phase 2: A Second, Narrower LLM Pass

The obvious next move is throwing a big model at the whole problem—GPT-4o or Claude, "here's the transcript, fix it." We didn't. Live interview audio means every extra round trip to a large hosted model is latency the candidate feels, and the cost of doing that on every turn of every interview adds up fast.

Instead we run **Mistral, locally, via Ollama**—and we don't ask it to do the whole job in one shot. We split jargon correction into two deliberately narrow calls that never touch each other's job.

**Call one — extract, don't rewrite.** The role-scoped prompt from Phase 1 isn't just a list of terms; it's paired with a strict extraction rule, including a worked example baked right into the template:

```python
# keyword_extractor.py — the extraction prompt
prompt = f"""
{role_context}

TASK:
Extract ONLY technical terms or short technical phrases that:
- Are explicitly mentioned in the transcript, OR
- Are unambiguously implied by technical phrasing
  (example: "driver related vulnerabilities" → "driver vulnerabilities")

RULES:
- DO NOT introduce new concepts
- DO NOT force role-specific terms
- DO NOT extract generic nouns or verbs
- DO NOT rewrite the transcript
- DO NOT explain
- Output ONLY a JSON array of lowercase strings
"""
```

That one-shot example tells you a lot about how conservative this step actually is. It's not inferring "Zygote" from a vague description of "an Android process that isolates apps"—it's trimming filler words off a phrase that's already almost entirely correct. The model is a term-extractor, not a mind reader.

**Call two — fix grammar, touch nothing else.** Once fuzzy matching has reinforced the transcript against those extracted terms, a second, completely separate Mistral call does one job: clean up sentence flow without changing meaning.

```python
# autocorrect.py
def build_prompt(role: str, text: str) -> str:
    context = ROLE_CONTEXT.get(
        role, "Preserve ALL technical terminology exactly."
    )
    return f"""
You are performing FINAL-PASS grammatical correction.

STRICT RULES:
- DO NOT add information
- DO NOT remove information
- DO NOT paraphrase
- DO NOT explain
- ONLY fix obvious grammar and sentence flow
- Preserve technical terms EXACTLY

Role constraints:
{context}

Transcript:
{text}

Output ONLY the corrected transcript.
""".strip()
```

Neither call is asked to "interpret and rephrase" the way a general-purpose proofreader would. One extracts, one polishes grammar—and both are explicitly forbidden from paraphrasing. That's a deliberate hedge against the exact failure mode a single do-everything LLM call invites: a model that's "helpfully" confident enough to swap in the wrong technical term nobody asked it to touch.

## The Final Architecture

The repo's own README states the pipeline in five lines, and it holds up:

1. **Transcribe** the audio with `faster-whisper` (`base`, GPU).
2. **Extract keywords** with Mistral, scoped to the candidate's chosen interview role.
3. **Reinforce** the transcript with `rapidfuzz`, swapping in extracted keywords wherever token similarity clears the threshold.
4. **Autocorrect** with a second, narrower Mistral call—grammar only, no paraphrasing, terms preserved exactly.
5. **Return** the corrected transcript over the WebSocket.

```python
# main.py — the pipeline wired end to end
keywords = generate_keywords_mistral(raw_transcript, role)
reinforced = reinforce_with_keywords(raw_transcript.lower(), keywords)
prompt = build_prompt(role, reinforced)
final_transcript = autocorrect_with_mistral(prompt)
```

## Lessons Learned

Building this taught us a crucial lesson about the current state of AI development: **bigger isn't always better, and deterministic isn't always dead.**

A single large model call sounds simpler on a whiteboard, but it's slower, more expensive, and—if you let it "interpret" freely—more prone to quietly swapping in the wrong term with total confidence. By splitting the job into a fast deterministic fuzzy pass and two narrowly-scoped local LLM calls that are explicitly forbidden from paraphrasing, we built something that's fast enough for a live interview and honest enough not to invent things it wasn't asked about.

Now, when a candidate on the pentester track says, "I'd start recon with Nmap, then check for SQL injection or XSS before touching privilege escalation," the system isn't guessing at unfamiliar words—it already knows exactly what it's listening for. And for a team of developers building tools for other developers, there's no better feeling than that.
