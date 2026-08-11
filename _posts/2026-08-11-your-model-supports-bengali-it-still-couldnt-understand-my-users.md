---
layout: post
title: "Your Model Supports Bengali. It Still Couldn't Understand My Users."
date: 2026-08-11
category: ai-workflows
tags: [speech to text, asr, whisper, indian languages, dialects]
description: "Whisper large-v3 and Sarvam both support Bengali, and both mis-transcribed my Tripura and Darjeeling samples. Why ASR coverage is counted in languages while failure happens below that level, and why it fails silently."
---

Every voice product starts with the same assumption: speech-to-text is a solved problem. You pick a model, wire up the API, and move on to the interesting parts.

I believed this for about a week.

I was building voice-first workflows meant to be used by people in India, which meant the very first thing in my pipeline — the transcription layer — had to handle how people here actually talk. Not the way they talk when reading a script into a good microphone. The way they talk when they're mid-thought, with the accent and regional inflection they grew up with.

And it had to do it without being told what language to expect. That was a hard requirement, not a nice-to-have. I couldn't put a language dropdown in front of a user and ask them to declare themselves before speaking — half the point of a voice interface is that you just talk. So what I actually needed was **automatic language identification and transcription in a single pass**: audio in, correct text out, with the system working out for itself what it was listening to.

That turned out to be the hardest part of the entire system, and I never fully solved it.

## Round One: Whisper

Whisper was the obvious starting point, and not just because it's free and local. It's one of the few models that does conceptually exactly what I described — it identifies the language and transcribes it in the same pass, without being told in advance. Architecturally that's the right shape for the problem.

Credit where it's due, too: its coverage of major Indian languages is genuinely commendable. Hindi, Bengali, Tamil, Telugu, Kannada, Malayalam, Marathi, Gujarati, Punjabi, Urdu, Nepali, Assamese — they're all in there, in one open model you can run on your own hardware. A few years ago that would have been an entire research programme. Anyone who writes this off as an English model that "also does other languages" hasn't used it.

I also wasn't sandbagging the test. This is the configuration I ran:

```python
MODEL_SIZE   = os.getenv("FW_MODEL",   "large-v3")   # biggest available
DEVICE       = os.getenv("FW_DEVICE",  "cuda")
COMPUTE_TYPE = os.getenv("FW_COMPUTE", "float16")
LANGUAGE     = os.getenv("FW_LANG",    None)         # None = auto-detect
```

`large-v3` on GPU — the largest, most accurate variant, not a `base` or `small` chosen for
latency. And `LANGUAGE` defaults to `None`, which is the auto-detection requirement expressed
in one line: don't tell the model what it's about to hear, let it work that out.

It was fine. It was not good enough.

The failures weren't dramatic — Whisper rarely produced obvious gibberish. It produced *confidently wrong, fluent-looking text*, which is worse. It would smooth an unfamiliar pronunciation into the nearest common word and hand me back a grammatical sentence that meant something different from what the speaker said. In a pipeline where that transcript feeds a downstream LLM, a plausible wrong transcript is far more damaging than an obviously broken one. Nothing downstream flags it. The error just propagates with full confidence.

Naturally, I tried to tune my way out of it. Here's where I ended up, and it's worth reading as a record of what didn't work:

```python
segments, info = self.model.transcribe(
    unprocessed,
    language=LANGUAGE,          # None → auto-detect (handles Hinglish)
    vad_filter=True,
    vad_parameters=dict(min_silence_duration_ms=300),
    beam_size=5,
    best_of=5,
    no_speech_threshold=0.45,   # default 0.6
    log_prob_threshold=-1.2,    # default -1.0
    compression_ratio_threshold=2.4,
    condition_on_previous_text=True,
)
```

`beam_size=5, best_of=5` is maximum search effort. The two loosened thresholds are the
interesting ones — I dropped `no_speech_threshold` from 0.6 to 0.45 and `log_prob_threshold`
from -1.0 to -1.2, which both push in the same direction: *keep* the low-confidence transcript
instead of discarding it. I was getting dropped audio on accented speech and I fixed it by
telling the model to be less picky. Which worked, in the sense that I got text back. It just
meant I was now keeping the output the model itself was least sure about.

And `condition_on_previous_text=True` is the quiet one. Each decode gets the previous
transcript as context, so once the model settles into the wrong reading of a speaker, that
wrong reading conditions everything that follows. It's a coherence feature that doubles as an
error-propagation mechanism. I left it on because turning it off made the output choppier.

None of this was a fix. It was rearranging which failure I was willing to look at.

Here's the part that surprised me when I went looking for corroboration: **Whisper is the best-performing model on this problem, not the worst.** AI4Bharat's [Svarah benchmark](https://github.com/AI4Bharat/Svarah) — 9.6 hours of Indian-accented English from 117 speakers across 65 districts — evaluated eight ASR systems and found the Whisper family came out on top, with medium and large variants reaching single-digit WER and the *smallest* gap between Indian-accented and native-speaker performance. The authors [hypothesize](https://arxiv.org/pdf/2305.15760) this is a side effect of the sheer diversity in Whisper's weakly-supervised training data.

So I wasn't using a bad model. I was running the largest variant of the best-performing family on the benchmark built specifically to measure this, and still hitting a wall. That reframed the problem: this isn't a "pick a better model" problem, because I had already picked the better model.

It's also worth noting where benchmark conditions and my conditions diverge. Svarah includes spontaneous conversational data, but a benchmark is still recorded with intent. My users were talking to a product, in a room with a fan on, sometimes over a phone speaker, without knowing or caring that a model was listening. Benchmark WER is a floor, not a forecast.

## Round Two: Sarvam

The natural next move was a model built specifically for this. [Sarvam AI](https://www.sarvam.ai/models) is the most credible Indian-language speech stack I've used — genuinely production-ready, sensible API, and their Saaras models advertise coverage of all 22 official Indian languages plus English, with specific attention to code-mixed and noisy speech.

Scope first, because it matters for how much weight to put on any of this. I tested the free
tier, in early 2026, on my own recordings. It was not a systematic evaluation and it was not a
large sample. I didn't log which model version served my requests, and their lineup has moved
since — Saarika has been superseded, and the Saaras line has gone through more than one release
in the meantime. So read what follows as one developer's snapshot from a specific few weeks,
not as a benchmark, and certainly not as a claim about what the current models do.

With that said: for standard Bengali it was a clear improvement over Whisper. For Indian-accented English, better. For code-mixed input, noticeably better — that's exactly the case it's designed for, and it shows.

Then I ran my own samples of Bengali as spoken in **Tripura**, and Bengali from the regions **around Darjeeling**. Most of them came back mis-transcribed.

Not the loud kind of failure. The same quiet normalization Whisper did, one layer up: the model pulled the input toward the variety it knew and returned something fluent and wrong. I want to be careful here — I'm describing observed behavior and my read of the cause, not something I instrumented. But it was consistent enough that I stopped trying to tune around it.

## The Pattern: Coverage Is Counted in Languages, But Failure Happens Below That

This is the part that took me embarrassingly long to see.

Bengali isn't a gap in anyone's coverage. It's in the Eighth Schedule of the Constitution. It's the second-most-spoken language in India, roughly 97 million speakers at the 2011 census. Every serious Indian ASR stack supports it — Sarvam does, AI4Bharat's IndicConformer does, Whisper does. If you were auditing model coverage on a spreadsheet, Bengali would be a green cell everywhere.

And it still failed, because **"supports Bengali" means "supports Standard Colloquial Bengali"** — the Kolkata and West Bengal register that dominates broadcast, publishing, and therefore training data. That's one variety among many, and the two I was testing sit a long way from it.

**Tripura** sits in the eastern Bengali dialect region, the continuum running through Sylhet and the Barak Valley. This isn't a light accent on standard Bengali. Linguists describe [eastern Bengali varieties](https://en.wikipedia.org/wiki/Eastern_Bengali_dialects) as diverging in phonology, lexicon, *and* morphosyntax — to the point that Sylheti and its neighbours are generally not mutually intelligible with Standard Bengali, and many linguists treat Sylheti as a separate language rather than a dialect. Some eastern varieties carry **contrastive tone**. Standard Bengali does not. Others have vowel distinctions and diphthongs with no equivalent in the standard inventory.

Sit with that as an engineering problem. You're asking a model to recognise phonemic contrasts that don't exist in its training distribution. It has no representation for them, so it does the only thing it can: maps them onto the nearest thing it does know. That's not a tuning failure. There's nothing to tune.

**Darjeeling** is a different problem with the same outcome. It's a contact zone — at the 2011 census the district ran roughly Nepali 46%, Bengali 29%, Hindi 17%, and Nepali holds official status alongside Bengali in the Darjeeling, Kalimpong, and Kurseong hill subdivisions. Bengali there is a minority language spoken in constant three-way contact, and it carries that. A model trained on Kolkata Bengali has never heard Bengali shaped this way.

Neither of these speakers is doing anything unusual. They're speaking their own language, in their own region, the way everyone around them speaks it. The model's category for "Bengali" just doesn't have room for them.

## Auto-Detection Makes This Worse, Not Better

Here's where my hard requirement came back to bite me.

Language identification is a classifier over a fixed label set. Whisper picks a language token and that token then conditions everything the decoder does afterwards. It's an elegant design and it's the reason the model can do detect-and-transcribe in one pass at all.

But it means there is no "none of the above." Every utterance gets a label, whether or not a correct one exists in the set.

Feed it Tripura Bengali and there is no right answer available. The eastern varieties aren't in the label set — Sylheti has no token, and neither does any other regional variety. So the classifier picks the nearest thing it has, almost certainly `bn`, and from that moment the decode is conditioned on Standard Colloquial Bengali. The normalization I kept seeing isn't only acoustic drift. It's partly *enforced* by the pipeline: the model has already committed to which Bengali it's hearing before it transcribes a single word, and it committed to the wrong one because the right one was never an option.

Darjeeling has a nastier version of the same problem. In a district running roughly 46% Nepali, 29% Bengali, and 17% Hindi, all three of those *are* Whisper-supported labels — so the classifier has several plausible options and no strong reason to prefer the correct one on a short, contact-influenced utterance. There the failure mode isn't just the wrong variety. It's a real chance of the wrong language outright, and the transcript that follows is downstream of that choice.

I should be clear that this is me reasoning from documented architecture, not something I measured with a confusion matrix — and as you'll see in a second, I made fairly sure I *couldn't* measure it. But it changed how I think about the trade. Auto-detection is a genuine convenience and I still think it's the right interface. It also quietly converts an open-ended acoustic problem into a forced-choice classification over a label set that doesn't contain your user — and it does that before any of the parts you can debug.

There's a postscript to this that I find harder to write than the rest of the post.

That hard requirement I opened with — never make the user declare their language, always auto-detect? It survived in my standalone scripts, where `LANGUAGE` defaults to `None`. It did not survive contact with the deployed service. Here's the signature of the transcription method in the web app I actually shipped:

```python
def transcribe_audio(self, audio_file_path, language="en", task="transcribe"):
    ...
    # Set language to None for auto-detection if 'auto' is specified
    whisper_language = None if language == "auto" else language
```

The default is `"en"`. Auto-detection still works, but only if the caller explicitly asks for it — it went from being the default behaviour to being an opt-in flag, and every call site that didn't think about it got hardcoded English. Somewhere between the prototype and production, under the ordinary pressure of getting something working, the requirement inverted. I didn't make a decision to abandon it. I just wrote a default argument one afternoon.

That's worth sitting with, because it's the same failure as everything else in this post, one level up: the accommodation is available, nothing forces you to use it, and nothing complains when you don't.

## The Real Problem Is That It Fails Silently

If a model doesn't support a language, you find out immediately. You pass an unsupported code, you get an error. Or you get output so obviously wrong that it fails in testing.

Variety-level failure gives you neither. The API accepts `bn`. It returns well-formed Bengali. No error, no exception. Every automated check in your pipeline passes. The transcript is grammatical, plausible, and about something slightly different from what your user said — and it flows straight into an LLM that has no way to know it's working from a corrupted input, and every incentive to produce a confident answer anyway.

Now, the part where I have to indict my own code.

`faster-whisper` does not actually hide any of this from you. Every call to `transcribe()`
returns a tuple, and the second element carries exactly what you'd want: the language it
decided on, and `language_probability`, its confidence in that decision.

I went back through three codebases where I'd called it. Here is how I used it:

```python
segments, _    = self.model.transcribe(...)    # not even bound
segments, info = self.model.transcribe(...)    # bound, never read again
language_detected = info.language              # captured! ...and that's all
```

Three different levels of engagement. In the first I discarded the object with an underscore.
In the second I bound it and never referenced it again. In the third — the one that actually
shipped — I did the responsible-looking thing, pulled `info.language` out, returned it in the
API response, and felt good about myself.

In none of the three did I ever read `language_probability`.

I captured *which* language the model picked. I never once captured **how sure it was**. And
the entire argument of this post is that the picking is where it goes wrong. The exact signal
that would have said "I am 41% confident this is Bengali and it might be Nepali" was handed to
me, free, on every decode, for the whole project. I looked straight past it three times.

It gets worse, because the shipped service *does* report a confidence number — and I wrote it
myself:

```python
confidence = 0.8                              # base
if len(text) > 50:   confidence += 0.1        # longer = more confident?
if len(text) > 100:  confidence += 0.05
if text.endswith(('.', '!', '?')):
    confidence += 0.05                        # punctuation = more confident?
```

Read what that fallback rewards. A long, fluent, well-punctuated transcript scores high. That
is a precise description of the failure mode this whole post is about — the model smoothing a
variety it doesn't know into confident, grammatical, wrong text. My confidence metric wasn't
merely failing to catch that. It was *scoring it as a success.*

I don't think I'm unusual here. Look at any tutorial, any integration snippet, any Stack
Overflow answer — it's `segments, _ = model.transcribe(...)` all the way down. The ergonomics
of the API make the real confidence signal something you have to deliberately reach for, and
nothing about a normal-looking transcript ever prompts you to reach for it. So you reach for a
proxy instead, and the proxy tells you what you want to hear.

So the failure is silent, but it's silent partly by our own construction. That's what makes it
expensive to catch: it doesn't announce itself in your logs, because you didn't write the log
line. It announces itself when a user gives up on your product, and you never learn why.

The cheap fix, if you take one thing from this post: log `language_probability` on every
decode and alert when it drops. It costs you one line. It would have saved me weeks.

For anyone building on top of ASR, the practical version of this is: **"the model supports my language" is not the question.** The question is whether it supports your users' variety of it, and no model card on earth will tell you that. You have to go and test with actual speakers, from actual regions, and you have to do it before you build anything downstream that assumes the transcript is right.

## Where I Landed

I don't have a clean ending. I didn't find a model that solved it.

What I do have is a corrected mental model, which turned out to be worth more than I expected.

**The gap isn't "Indian languages."** That's being worked on, competently and at scale. AI4Bharat's data collection ran across 400+ districts. Sarvam is shipping real products. Whisper generalises to Indian accents better than it has any right to. Anyone claiming nobody is building for India hasn't looked.

**The gap is the layer below language** — variety, region, register. It's invisible on a coverage spreadsheet, because the spreadsheet has one row per language, and that unit of measurement is itself the bug. India has 22 scheduled languages and a vastly larger number of ways of speaking them, and right now the second number doesn't appear anywhere in how we evaluate these systems.

If I could ask for one thing, it wouldn't be a bigger model. It would be an evaluation set: Svarah, but *within* languages rather than across them. Tripura Bengali, Darjeeling Bengali, and the equivalents in every other supported language, recorded in the conditions people actually speak in. Right now nobody can put a number on how badly ASR degrades between Kolkata and Agartala — and you can't fund what you can't measure.

The models follow the benchmark. They always do. Somebody just has to build the benchmark first.
