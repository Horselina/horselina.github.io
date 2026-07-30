---
layout: post
title: "taming the chaos: how i learned to stop worrying and fix race conditions in real-time voice ai"
date: 2026-07-27
category: ai-workflows
tags: [race conditions, real-time voice ai]
description: "summary of a learning experience with fixing race conditions."
---
# taming the chaos: how i learned to stop worrying and fix race conditions in real-time voice ai

## **Taming the Chaos**

I’ll be the first to admit it: when I first set out to build a seamless, one-on-one voice chatbot experience, I was naive. In my head, the architecture was a clean, linear pipeline. The user speaks, we transcribe, we send it to the LLM, and we read the response out loud. Simple, right?

Reality had other plans. What I actually built was a deeply unreliable, chaotic mess. The core issue? A labyrinth of race conditions that turned the chatbot into an awkward, interrupting mess.

### **The Suspects**

To understand the problem, you have to understand the pieces I strung together. Trying to be clever, I built a client-heavy architecture with the following components running concurrently:

1. Browser Speech Recognition: Using the native Web Speech API for low-latency, partial transcriptions.
2. Whisper-based Transcription: Running in the background for high-accuracy, finalized text.
3. `Silence Detection`: To figure out when the user stopped talking.
4. `Noise Floor Detection & Adaptation`: To dynamically adjust the silence threshold based on the user’s environment (e.g., a quiet office vs. a busy coffee shop).
5. `TTS (Text-to-Speech) Firing`: The final step to speak the AI’s response.

On paper, these components complemented each other. In practice, they were like five different orchestras trying to play in the same room without a conductor. Here's the actual kickoff — three of these five, wired to fire at once, on every single recording:

```ts
// frontend/src/pages/VoiceInterview.tsx
recognitionRef.current = recognition;
try { recognition.start(); } catch (err) { }
startWhisperChunkStreaming();
detectSilence(stream);
```

### **The Nightmare**

The race condition manifested in a few particularly ugly ways.

The most common scenario went like this: A user would pause for a breath. The `Silence Detection`, still calibrating against the `Noise Floor`, wouldn’t trigger. But the `Browser Speech Recognition` would interpret the micro-pause as an endpoint, firing off a partial transcript to the LLM.
Meanwhile, `Whisper` was still processing the audio chunk, waiting to send its final, accurate transcript.

Before `Whisper` could finish, the LLM would generate a response based on the partial text, and the `TTS` would fire. Suddenly, the chatbot was talking over the user who was actually still speaking.
It was a classic race condition: multiple asynchronous processes with shared state (the user's microphone input and the conversation context) and no strict locking mechanism. The timing was so fragile that a slight change in the user’s background noise would completely break the flow.

The smoking gun, once I went looking for it: the same "did the user just say they're done?" check existed in two independent places, each free to act on its own. Web Speech's version, checked on every recognition result:

```ts
// frontend/src/pages/VoiceInterview.tsx — inside recognition.onresult
const matchedPhrase = STOP_PHRASES.find(p =>
  (lower === p || new RegExp('(^|\\b)' + p.replace(/[.*+?^${}()|[\]\\]/g, '\\$&') + '$').test(lower))
);
if (matchedPhrase && confidence >= 0.7) {
  // ...
  shouldStop = true;
}
// ...
if (shouldStop) {
  finalizeAnswer();
}
```

And Whisper's version, checked independently every 4 seconds against whatever had accumulated in the chunk poll:

```ts
// frontend/src/pages/VoiceInterview.tsx — inside startWhisperChunkStreaming
const STOP_PHRASES = ["that's my response", "that is my response", "i am done", "i'm done", "that's all", "that is all", "done speaking", "that's it"];
if (grew && STOP_PHRASES.some(p => lower.endsWith(p))) {
  finalizeAnswer();
}
```

Two separate implementations of the same decision, neither aware the other existed, both able to call `finalizeAnswer()` whenever they felt like it.

## Phase 1: The "Hit-and-Trial" Swamp

I didn’t immediately see the architectural flaw. Instead, I entered the hit-and-trial phase—the dark forest of every developer's imposter syndrome.
I started playing whack-a-mole with the code.

- If the bot interrupted too early, I added a `setTimeout` to delay the TTS firing.
- If the noise adaptation was too slow, I manually tweaked the decibel threshold offsets.
- I tried debouncing the Web Speech API events.

The threshold-tweaking wasn't hand-waving — here's what the ambient-noise tracking actually looked like, decaying at different rates depending on whether someone was mid-sentence or not:

```ts
// frontend/src/utils/audioVAD.ts
const SPEECH_RATIO = config.speechRatio ?? 3.0;
const MIN_ABS_SPEECH_RMS = config.minAbsSpeechRms ?? 0.01;
const CALIBRATION_MS = config.calibrationMs ?? 800;
// ...
if (!speechStarted) {
  // Pre-speech: track ambient quickly so the noise floor settles.
  noiseFloor = noiseFloor * 0.95 + rms * 0.05;
} else if (!isSpeech) {
  // Post-speech silence: track rising noise slowly so we don't lose the speaker.
  noiseFloor = noiseFloor * 0.98 + rms * 0.02;
}
```

It was a hybrid approach born of necessity: part scientific method, part blind luck. I would change a variable, deploy it, and test it in my quiet home office. It would work! I’d celebrate. Then, I’d take my laptop to a café, and the ambient noise would completely derail the noise floor adaptation, causing the silence detection to never trigger, leaving the user staring at a blinking mic icon forever.
I had built a house of cards. I was humbled. It became clear that patching individual thresholds couldn’t fix a fundamental state-management issue.

The closest I got to a real fix at this stage wasn't tuning a number — it was admitting the three triggers could never be trusted to stay mutually exclusive on their own, and bolting a lock onto the one thing they all funneled into:

```ts
// frontend/src/pages/VoiceInterview.tsx
const finalizeAnswer = () => {
  if (finalizeLockRef.current) return;
  finalizeLockRef.current = true;
  // ... stop recognition, clear every interval, stop the recorder
};
```

It worked, in the sense that duplicate finalizations stopped happening. But a lock around the symptom is not the same as removing the race — it just made the collision harder to notice.

### Phase 2: The Methodical Pivot

I stepped back, drew a sequence diagram on a whiteboard, and finally accepted a hard truth: **The browser is a hostile environment for complex audio state management.**

Having multiple client-side listeners competing for the same audio stream—and making decisions based on that stream—was the root cause of the race condition. I couldn't reliably synchronize the Web Speech API, a Whisper worker, and audio analyzers on the front end.

### The Solution: Dumbing Down the Client, Empowering the Server

We shifted to a server-driven state machine. The client was relegated to what it should have been from the start: a dumb pipe.

#### Key Changes

```bash
# Killing the Web Speech API
native browser speech recognition completely removed

# Raw Audio Streaming
browser opens WebSocket and streams raw audio bytes (PCM data) directly to backend
```

#### Unifying the Transcription Flow

```bash
# Server-Side VAD and Noise Adaptation
server processes single, uninterrupted stream of audio bytes with adapted noise floor
# Silero VAD algorithm used

# Unified State Machine
IDLE -> Audio arrives -> LISTENING
VAD detects speech -> PROCESSING
VAD detects silence -> Server sends exact audio chunk to Whisper
Whisper returns text -> LLM generates response -> SPEAKING
TTS fires and streams back to client
```

#### Simplifying Client Dependencies
We removed end-client dependencies, ensuring a more reliable and efficient state management system.

### The Takeaway
By moving the logic to the server and enforcing a strict state machine, the race condition vanished. Whisper *cannot* be called until the server-side VAD confirms silence. The TTS *cannot* fire until Whisper and the LLM have finished. There is no overlap, only a clean, sequential handoff.

#### Lessons Learned
I learned that in real-time AI solutions, synchronization is infinitely more important than raw speed. Trying to squeeze every millisecond of latency out of the browser by running parallel transcription models just creates chaos. Sometimes, the most elegant architecture isn't about adding clever optimizations—it's about having the humility to strip things back, move the complexity to a controlled environment, and trust the pipeline.
