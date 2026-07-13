# Real-Time Voice Agents

A voice agent is a soft-real-time media system wrapped around an LLM. The reasoning is the easy part; the hard part is **timing**: get audio in, decide when the human actually stopped talking, think, and get audio back out fast enough that the conversation feels alive. Miss the timing and users talk over the agent, repeat themselves, and hang up. This chapter covers the architecture, the latency budget, the 2026 stack, and the production failure modes.

A scoping note: read this if you are building telephony, support, or voice-first products. If you are not, the [agent fundamentals](../07-agentic-systems/01-agent-fundamentals.md) and [tool-use](../17-tool-use-and-computer-agents/01-tool-use-landscape.md) chapters cover the parts that transfer. Specific model names, prices, and latency figures below are a mid-2026 snapshot and move fast; verify before quoting.

## Table of Contents

- [The Two Architectures](#the-two-architectures)
- [The Pipeline, Component by Component](#the-pipeline-component-by-component)
- [Latency Budgets](#latency-budgets)
- [The 2026 Stack](#the-2026-stack)
- [Production Concerns](#production-concerns)
- [Honest Maturity](#honest-maturity)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Two Architectures

Every voice agent is one of two shapes.

**Cascaded pipeline:** `mic -> VAD -> streaming STT -> endpointing -> LLM -> streaming TTS -> speaker`. Audio becomes **text** at each boundary, and every stage is a swappable component, often from a different vendor.

**Speech-to-speech (S2S):** a single multimodal model ingests audio frames and emits audio frames directly, with text produced only as a side channel. OpenAI's Realtime API, Google's Gemini Live native audio, and AWS Nova Sonic are the 2026 examples.

| Dimension | Cascaded | Speech-to-speech |
|-----------|----------|------------------|
| **Latency floor** | Higher in naive form; with full streaming, total collapses toward `max(stage)`, roughly 400-800ms | Lower structurally (no text round-trips) |
| **Controllability** | High: text at every handoff, swap LLM/voice independently, A/B test, inject prompts | Low: one model, provider swap is a re-architecture |
| **Interruptions** | Deterministic, runtime-controlled (VAD fires, cancel TTS, roll back turn) | More natural full-duplex, but opaque failures |
| **Cost** | Predictable, often per-minute | Token-based, re-sends audio history each turn, grows with call length |
| **Observability** | Text artifacts at every boundary, easy to log and audit | Needs a parallel transcription stream for logs |
| **Prosody / emotion** | Reconstructed from text, so affect is often lost at the STT bottleneck | Preserves and generates tone, laughter, emphasis natively |
| **Language / code-switching** | Bounded by each component | Handles mid-sentence code-switching more naturally |

**Decision guide.** Pick **cascaded** when you need per-component debugging and audit trails, regulatory compliance (HIPAA, finance), provider flexibility, deep tool calling, and predictable cost. This is the enterprise production default in 2026, partly because the ecosystem has many interchangeable STT and TTS vendors versus only a few S2S providers. Pick **S2S** when you need maximum conversational naturalness, the lowest interruption latency, and expressive voice (companions, coaching, consumer demos), and can tolerate weaker debuggability and variable cost. A common **hybrid** runs S2S for the conversational core with a parallel STT stream purely for logging and evaluation.

A reality check that applies to both: neither wins latency by default. WebSocket setup, VAD configuration, network hops, codec, and sample rate dominate, and a well-tuned cascade matches S2S in many deployments.

---

## The Pipeline, Component by Component

**Voice Activity Detection (VAD).** The first layer classifies each audio frame as speech or silence in real time. Silero VAD is the de-facto open-source standard, integrated natively in the major frameworks, and adds only about 10-50ms. VAD alone cannot tell a mid-sentence pause from the end of a turn, which is the next problem.

**Semantic endpointing and turn-taking, the crux.** Two approaches:
- **Silence-threshold endpointing** waits N milliseconds of silence. Simple, but it taxes every single turn: an 800ms timeout adds nearly a full second before the pipeline even starts.
- **Learned turn detection** reads the partial transcript and predicts whether the thought is semantically complete, firing *before* trailing silence. Concrete 2026 systems include a small transformer turn-detector (a ~135M-parameter model fine-tuned from a small base, running in tens of milliseconds on CPU, with a multilingual variant) and streaming-STT endpointing that emits a learned end-of-turn token with a tunable confidence threshold. The payoff is closing the agent's turn-gap toward ~300ms without cutting users off.

**Barge-in / interruption.** Turn detection stays active *while the agent is speaking*. When the user's track fires VAD, the runtime cancels the active TTS stream, rolls back the interrupted LLM turn, and re-enters STT. WebRTC transport handles interruptions noticeably better than WebSocket because UDP avoids head-of-line blocking on packet loss.

**Streaming transcripts.** Streaming STT emits partial transcripts every ~50ms while the user is still talking, so transcription runs in parallel with speech and adds almost nothing after the user stops. Distinguish three STT latencies: partial-transcript, final-transcript (after speech ends), and endpointing latency. Voice agents optimize the last.

**TTS and time-to-first-audio (TTFA).** The agent must start speaking the moment the first words exist, not after the full sentence is generated. TTS streams audio in chunks; TTFA (time to the first audio byte) is the headline metric, with modern streaming engines targeting ~100-200ms for the first chunk.

---

## Latency Budgets

In natural human conversation, the gap between one person finishing and the other starting averages **~200ms**. Below roughly 700ms end-to-end an agent feels human; above it, callers start interrupting and repeating. A realistic per-turn budget for a fully-streaming cascade:

| Stage | Typical budget | Notes |
|-------|---------------|-------|
| Network transport (one way) | 30-80ms | WebRTC/UDP sub-100ms; SIP adds 20-50ms per hop |
| VAD frame decision | 10-50ms | Runs continuously |
| Endpointing / turn decision | 150-300ms | Dominated by your silence/confidence config |
| STT final transcript | 50-100ms after speech ends | Partials already streamed during speech |
| **LLM time-to-first-token** | **150-400ms** (can be higher) | Usually the single largest controllable cost |
| TTS time-to-first-audio | 100-200ms | First chunk only; the rest streams |
| **Practical end-to-end (TTFA)** | **~600-800ms** good | Streaming collapses total toward `max(stage)`, not the sum |

Naive non-streaming stacks run 1000-2000ms or worse. The levers, in order of impact: **stream everything** (partial transcripts, token streaming, chunked TTS, which turns a sum into a max); **tune endpointing** (a learned turn detector instead of a fixed long silence timeout); **pick a fast first-token path** (smaller/faster model or speculative drafting, since LLM time-to-first-token is the long pole); **choose TTS by TTFA** and start audio on the first clause; and **use WebRTC over UDP** with jitter under ~20ms.

---

## The 2026 Stack

Treat specific names, versions, and prices as a point-in-time snapshot.

- **Orchestration frameworks:** LiveKit Agents and Pipecat are the open-source leaders (WebRTC-native, bring-your-own STT/LLM/TTS or plug an S2S model, ship VAD plus a turn-detector). Vapi, Retell, and Bland are managed platforms. Managed tends to be cheaper below roughly 10k minutes/month once engineering time is counted; the framework path is reported cheaper per call above that.
- **STT:** Deepgram and AssemblyAI lead real-time streaming, both with built-in endpointing and reported word-error rates in the ~6-7% range (benchmark-dependent).
- **S2S realtime models:** OpenAI's Realtime API (the `gpt-realtime` family; check the [model taxonomy](../02-model-landscape/01-model-taxonomy.md) for the current supported version, since the suffix has churned), Google Gemini Live native audio, and AWS Nova Sonic. These support function calling and, for OpenAI, remote MCP servers inside the realtime session.
- **TTS:** Cartesia and ElevenLabs lead on low-latency streaming; vendor TTFA claims range from tens to ~200ms and are benchmark-dependent.
- **Transport:** WebRTC (UDP, Opus codec) for client-to-agent media; WebSocket (TCP) is simpler for the server-to-model leg but suffers head-of-line blocking on loss. A common hybrid is WebRTC client to relay, WebSocket relay to model API. SIP bridges to the phone network (PSTN), where each carrier hop adds latency.

---

## Production Concerns

**ASR errors are the dominant failure.** The benchmark finding worth internalizing: authentication is the bottleneck, because once the agent mishears a name, email, or confirmation code, everything downstream fails. Defenses: confidence thresholds that route low-confidence spans to a clarifying question ("did you say...?"), and custom vocabulary / keyword boosting for names, SKUs, and alphanumerics.

**Tool calls mid-conversation.** Both architectures support function calling. Emit a verbal filler ("let me check that") to cover tool latency so the line is not dead while a tool runs.

**Memory across turns.** S2S models re-send audio history each turn, so token cost and context pressure grow with call length; trim tool outputs and summarize. Durable cross-session memory in the voice path is still weak; keeping memory in the text/LLM layer (cascaded) is cheaper. See [Agent Memory and State](../07-agentic-systems/05-agent-memory-and-state.md).

**Telephony.** SIP trunks bridge to PSTN and deliver 8kHz mu-law audio, which breaks VAD and turn detection if not handled; this is a documented S2S failure mode on phone lines.

**Observability and evaluation.** Cascaded stacks give text artifacts for free; S2S needs a parallel transcription stream. The benchmark to know is **tau-Voice** (Sierra, arXiv:2603.13686), which extends agentic tool-use evaluation to full-duplex voice with noise, accents, and interruptions, and isolates voice-specific failure classes: ASR mishearing, interruption mismanagement, multi-step context loss, and failure to recover from dropouts. See [Evaluating Agentic Systems](../07-agentic-systems/10-evaluating-agentic-systems.md).

**Cost: audio tokens cost much more than text.** For OpenAI's realtime model, audio input is reported at roughly 8x the text input rate and audio output roughly 4x text output, and audio is duration-encoded (very roughly, user speech around 1 token per 100ms, synthesized speech around 1 token per 50ms). Prompt caching and trimmed tool outputs are reported to cut per-minute cost substantially. Budget voice agents per-minute, not per-token-in-the-abstract, and see [FinOps and Token Economics](../11-infrastructure-and-mlops/04-finops-and-token-economics.md).

---

## Honest Maturity

What works well in 2026: sub-second, natural-feeling single-turn latency on tuned cascaded stacks and on S2S; learned turn detection that beats silence thresholds; deterministic barge-in in cascaded runtimes; expressive prosody in S2S; and a mature tooling ecosystem.

What is still hard: full-duplex naturalness (graceful overlap, backchanneling, reliable mid-thought interruption); noisy real-world audio; mid-sentence code-switching; and long-context memory in the voice path. The headline caveat from tau-Voice: even frontier voice agents retain only roughly 30-45% of the equivalent text agent's task ability under realistic noisy conditions, and the large majority of failures are agent behavior, not the test harness. **Voice is not yet "a text agent plus a microphone."** Design for the gap: confirm critical slots (names, codes, amounts) explicitly, keep a human-handoff path, and evaluate under noise and interruptions, not just clean audio.

---

## Interview Questions

### Q: Walk me through the latency budget of a voice agent. Where do the milliseconds go, and what is the single biggest lever?

**Strong answer:**
Natural conversation has about a 200ms turn gap, and below ~700ms end-to-end the agent feels human. The budget splits across transport (30-80ms each way), VAD (10-50ms), endpointing (150-300ms, set by your silence or confidence config), STT final transcript (50-100ms after speech ends, since partials stream during speech), LLM time-to-first-token (150-400ms and usually the long pole), and TTS time-to-first-audio (100-200ms). The single biggest lever is streaming everything, because it turns the sum of stages into roughly the max: partial transcripts, token streaming, and chunked TTS overlap instead of serializing. After that, replacing a fixed long silence timeout with a learned turn detector removes a per-turn tax, and choosing a fast first-token model attacks the long pole.

### Q: Cascaded pipeline or speech-to-speech: how do you choose?

**Strong answer:**
Cascaded for anything needing auditability, compliance, provider flexibility, deep tool use, and predictable cost, because text at every boundary gives you logs, swappable components, and moderation hooks; it is the enterprise default. Speech-to-speech when conversational naturalness and the lowest interruption latency matter most and you can accept opaque failures, fewer vendors, and token cost that grows with call length, since it re-sends audio history each turn. Many teams run a hybrid: S2S for the conversational core, with a parallel STT stream purely for logging and evaluation. I would not assume S2S is automatically lower latency; a well-tuned cascade is competitive, and transport and endpointing config dominate either way.

---

## References

- Sierra, "tau-Voice: advancing agent benchmarking to knowledge and voice" arXiv:2603.13686, and [blog](https://sierra.ai/blog/bench-advancing-agent-benchmarking-to-knowledge-and-voice)
- OpenAI, [Introducing the Realtime API](https://openai.com/index/introducing-the-realtime-api/)
- LiveKit, [turn detection for voice agents](https://livekit.com/blog/turn-detection-voice-agents-vad-endpointing-model-based-detection) and [a transformer for end-of-turn detection](https://livekit.com/blog/using-a-transformer-to-improve-end-of-turn-detection)
- AssemblyAI, [turn detection and endpointing](https://www.assemblyai.com/blog/turn-detection-endpointing-voice-agent)
- Deepgram, [speech-to-speech vs cascade architecture](https://deepgram.com/learn/speech-to-speech-vs-cascade-voice-agent-architecture)
- Pipecat, [open-source voice agent framework](https://github.com/pipecat-ai/pipecat)
- Cekura, [voice AI evaluation metrics](https://www.cekura.ai/blogs/voice-ai-evaluation-metrics)

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **VAD (Voice Activity Detection)** | Software that classifies each audio frame in real time as either speech or silence | The first stage in any voice pipeline; gates all downstream processing so the system only acts when the user is talking |
| **Silero VAD** | A popular open-source neural VAD model that runs on CPU with roughly 10–50ms latency | De-facto open standard in voice agent frameworks; adds negligible delay and integrates natively with LiveKit and Pipecat |
| **Endpointing** | Deciding when a speaker has truly finished their turn, as opposed to pausing mid-sentence | The crux of voice agent timing; getting it wrong causes the agent to interrupt or wait too long on every turn |
| **Silence-Threshold Endpointing** | Triggering end-of-turn after N milliseconds of detected silence | Simple to implement but adds a per-turn tax of 600–800ms; the worst-case option for natural conversation feel |
| **Learned Turn Detection** | Using a small trained model to predict end-of-turn from the partial transcript before silence occurs | Cuts per-turn latency to ~300ms without cutting users off; the modern production approach |
| **Barge-in / Interruption** | The ability of the user to start speaking while the agent is still talking, causing the agent to stop immediately | Critical for natural conversation; requires VAD to run continuously even while TTS is playing |
| **STT (Speech-to-Text)** | Software that converts audio from a microphone into a text transcript | Produces the text input the LLM reasons over; word-error rate and endpointing latency are the key quality metrics |
| **Streaming STT** | An STT system that emits partial transcripts every ~50ms while the user is still speaking | Allows transcription to run in parallel with speech, so the LLM sees the text almost immediately after the user stops |
| **TTS (Text-to-Speech)** | Software that converts the model's text output into spoken audio | The last mile of the voice pipeline; TTFA is its headline latency metric |
| **TTFA (Time-to-First-Audio)** | The delay from when TTS receives the first text until the first audio byte is played to the user | The voice equivalent of TTFT; targeting 100–200ms is the modern standard for streaming TTS engines |
| **Cascaded Pipeline** | A voice architecture that chains separate components: VAD → STT → LLM → TTS, each swappable | Enterprise default for auditability, compliance, provider flexibility, and deep tool use |
| **Speech-to-Speech (S2S)** | A single multimodal model that ingests raw audio and emits raw audio without converting to text in between | Lowest structural latency and most natural prosody; less controllable and harder to debug than cascaded |
| **WebRTC** | A browser and server protocol for real-time audio/video transport over UDP | Preferred for client-to-agent media; UDP avoids head-of-line blocking that plagues WebSocket during packet loss |
| **WebSocket** | A persistent TCP connection used for bidirectional streaming communication | Simpler than WebRTC for server-to-model connections; suffers head-of-line blocking on packet loss |
| **UDP** | User Datagram Protocol; a fast, connectionless network protocol that allows some packet loss | Preferred for real-time media because late packets are more harmful than dropped packets |
| **SIP (Session Initiation Protocol)** | The standard signaling protocol for telephony and VoIP calls | Bridges voice agents to the phone network (PSTN); each SIP hop adds 20–50ms of latency |
| **PSTN (Public Switched Telephone Network)** | The traditional telephone network; delivers 8kHz mu-law audio | Imposes audio quality and sample-rate constraints that can break VAD and turn detection if not handled |
| **Opus Codec** | An open audio codec optimized for low-latency voice over the internet | Standard codec in WebRTC; good quality at low bit rates and minimal encoding/decoding delay |
| **Jitter** | Variation in packet arrival time across a network connection | Must stay under ~20ms for voice to feel natural; high jitter causes choppy audio |
| **Full-Duplex** | Both parties can speak and hear simultaneously, like a real phone call | S2S models handle this more naturally; cascaded pipelines require explicit interruption management |
| **Backchanneling** | Short verbal acknowledgments ("uh-huh", "yeah") that signal active listening without taking a turn | Still unreliable in 2026 voice agents; a known gap between AI voice and human conversation naturalness |
| **Code-Switching** | Flipping between two languages mid-sentence, common in multilingual speakers | S2S handles it more naturally than cascaded pipelines constrained by per-component language models |
| **ASR (Automatic Speech Recognition)** | Synonym for STT; the process of transcribing spoken audio to text | ASR errors—especially on names, codes, and numbers—are the dominant failure mode in production voice agents |
| **Keyword Boosting** | Raising the recognition probability of specific words (product names, SKUs, codes) in the STT model | Reduces authentication failures caused by the model mishearing critical alphanumeric identifiers |
| **Verbal Filler** | A phrase like "let me check that" spoken by the agent while a tool call executes in the background | Prevents dead air during tool latency, which callers interpret as a dropped call |
| **LLM TTFT (Time-to-First-Token)** | How long the LLM takes to produce its first output token after receiving the transcribed input | Usually the single largest controllable latency in a voice pipeline; dictates which LLM tier is viable |
| **LiveKit Agents** | An open-source WebRTC-native voice agent framework that integrates VAD, turn detection, STT, LLM, and TTS | One of two open-source orchestration leaders (alongside Pipecat) for building production voice agents |
| **Pipecat** | An open-source Python framework for building real-time voice and multimodal AI pipelines | The other major open-source orchestration option; component-swappable architecture with built-in VAD |
| **Deepgram** | A commercial real-time STT provider with built-in endpointing and ~6–7% word-error rate | Leading streaming STT vendor for production voice agents; offers keyword boosting and custom vocabulary |
| **AssemblyAI** | A commercial STT provider with streaming transcription and turn-detection features | Alternative to Deepgram with similar latency profile and built-in endpointing |
| **ElevenLabs** | A commercial TTS provider specializing in low-latency streaming synthesis and voice cloning | Leading TTS vendor by voice quality and cloning capability |
| **Cartesia** | A commercial TTS provider targeting ultra-low TTFA streaming synthesis | Competing with ElevenLabs on first-audio latency |
| **tau-Voice** | A benchmark (Sierra, arXiv:2603.13686) evaluating voice agents on full-duplex conversations with noise, accents, and interruptions | Reveals the gap between voice agent and text agent capability; frontier voice agents score ~30–45% of equivalent text agents |
| **AWS Nova Sonic** | Amazon's S2S real-time voice model available via AWS | One of three major 2026 S2S options alongside OpenAI Realtime API and Google Gemini Live |
| **MCP (Model Context Protocol)** | An open protocol for connecting models to external tools and data sources via a standardized interface | Enables OpenAI's Realtime API to call remote tools mid-voice-session without leaving the audio pipeline |
| **Head-of-Line Blocking** | A TCP behavior where a lost packet stalls all subsequent packets until it is retransmitted | Why WebSocket performs worse than WebRTC for real-time audio under any packet loss |

---

*Previous: [Safety and Governance](../17-tool-use-and-computer-agents/07-safety-and-governance.md) · Next: [Multimodal Generation](../19-multimodal-generation/01-multimodal-generation.md)*
