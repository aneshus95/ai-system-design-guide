# Multimodal Generation

Generating images, video, and audio is a different engineering problem from understanding them. A generative-media product is an **asset-processing pipeline of long-running, non-deterministic, expensive jobs**, and most of the work is orchestrating that safely, proving provenance, and evaluating quality you cannot reproduce exactly. This chapter leads with those durable concerns and treats the specific models as a perishable snapshot. Every capability figure below is a reported or vendor claim unless tied to a primary source.

**Scoping (read first):** this topic is load-bearing for creative, media, marketing, game, and avatar products, and largely noise for backend services, data platforms, RAG, and text-only agents. The one piece that reaches *any* product touching user-uploaded or AI-generated media is **provenance and safety**, because the legal obligations attach to generating or distributing synthetic media, not to your domain. If your system never emits a pixel or a waveform, only that section applies.

## Table of Contents

- [Production Pipeline Patterns](#production-pipeline-patterns)
- [Provenance and Safety](#provenance-and-safety)
- [Evaluating Generative Quality](#evaluating-generative-quality)
- [The Model Landscape](#the-model-landscape)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Production Pipeline Patterns

This is the part that outlives any model.

**Chaining is a DAG, not one call.** The canonical creative pipeline is a graph of stages, often each a different model from a different vendor: `prompt -> image (keyframes/style) -> image-to-video -> lip-sync -> voice/TTS -> music/SFX -> mux`. The durable lesson is **separation of concerns at the stage boundary**, so any stage is swappable, the same cascaded-versus-monolithic tradeoff the [voice agents](../18-voice-and-audio-agents/01-realtime-voice-agents.md) chapter draws. Node-graph tools (ComfyUI) codify this for images and video; treat the workflow graph as versioned code, not UI state.

**Async, queue, and long-job handling is mandatory.** Image generation is seconds; **video generation is tens of seconds to minutes per clip**, so no synchronous request survives it. The universal pattern: submit returns a job ID immediately, work runs on a GPU worker pool, and the client learns completion via a webhook callback (with a polling fallback, because webhooks get dropped). Verify webhook signatures, and **autoscale on queue depth**, not CPU. Because a single video job costs real money and minutes, attach an idempotency key per logical request so a client retry or a redelivered webhook does not pay twice.

**Cost and latency control**, in rough order of leverage: cache on the full request fingerprint (prompt plus all parameters plus seed plus model version) so identical requests never re-bill; draft cheaply (low resolution, few steps, a small fast model) and re-render only approved drafts at full quality, the single biggest saver in video; batch where supported; pick the tier deliberately (turbo and distilled variants trade quality for large latency and cost wins); and keep warm worker pools, because loading a multi-gigabyte checkpoint can take tens of seconds.

**Asset and prompt management.** Treat prompts, negative prompts, seeds, model and version, and all conditioning inputs (control maps, reference images, adapter IDs and weights) as a structured, versioned record stored with every output. You cannot reproduce or debug a generation without the full parameter set, and providers silently change models behind a stable name. This manifest doubles as your provenance log.

**Seeds and reproducibility, the hard truth.** A seed makes a *local, pinned* run reproducible, but reproducibility degrades the moment you cross hardware or batch boundaries and is effectively absent on hosted APIs. The root causes are general, not generative-specific: floating-point arithmetic is non-associative, so different GPU kernels accumulate in different orders; batch size changes the kernel strategy (the most common source of numerical noise); and some GPU operations are nondeterministic unless explicitly forced. Practical stance: self-hosted with pinned hardware, fixed batch, fixed library versions, and a seed gives near-reproducible images; any hosted API, and especially video, should be treated as non-reproducible, so design QA around perceptual similarity, not exact match.

---

## Provenance and Safety

The technology churns; the obligation to label and the inability to perfectly enforce it are durable. Two complementary layers, **and both are removable.**

**C2PA / Content Credentials** is an open standard that cryptographically binds provenance to an asset (a "nutrition label for media"). A manifest holds assertions about creation and edits, a signed claim, and references to source assets, with **hard bindings** (cryptographic hashes of the bytes, tamper-evident) and **soft bindings** (fingerprints or watermarks that survive re-encoding). "Durable Content Credentials" recover a stripped manifest by looking up a surviving watermark or content fingerprint in an external repository, which exists precisely because manifests get separated from assets by re-encoding and screenshots. Adoption is real and growing: major image and video generators now attach C2PA metadata and watermarks, some camera makers sign photos at capture, and platforms read credentials to apply AI labels. A cautionary example to teach: one camera maker added then suspended C2PA after a signing-key vulnerability, because the trust model is only as good as key custody.

**Watermarking (SynthID-style)** adds an imperceptible signal to generated media. The durable caveat for engineers: **invisible watermarks are not robust against a motivated adversary.** Peer-reviewed work shows a regeneration attack (add noise, denoise with a diffusion model) provably removes invisible image watermarks below a perturbation threshold while preserving quality, and text watermarks are degraded by paraphrase and back-translation. Watermarks deter casual misuse and enable platform labeling; they do not stop adversaries. Layer C2PA signed provenance, a watermark that survives metadata stripping, and classifier-based detection, and assume each can be defeated alone.

**Safety filters** apply input moderation (block disallowed prompts) and output moderation (NSFW and identity classifiers) before returning an asset. The high-severity classes are non-consensual intimate imagery, deepfakes of real people, voice cloning without consent, and copyright or likeness infringement; mitigations include prompt and region blocklists, known-face rejection, consent-gated likeness, rate limiting, and abuse logging tied to the provenance manifest.

**The 2026 regulatory backdrop** (this is the part that reaches non-media products that merely host media): the EU AI Act's transparency article requires providers to mark generated output in a machine-readable way and deployers to disclose deepfakes, with obligations applying from August 2026 (see [AI Governance and Compliance](../13-reliability-and-safety/04-ai-governance-and-compliance.md)); the US TAKE IT DOWN Act (2025) targets non-consensual intimate imagery including AI deepfakes with platform takedown duties; and state likeness laws (the Tennessee ELVIS Act and many sexually-explicit-deepfake statutes) protect voice and likeness. Music and voice generation additionally carry unsettled copyright litigation, which makes a model's licensing and indemnity status a real procurement question.

---

## Evaluating Generative Quality

Two unavoidable facts: automated metrics are weak, and human preference is the real ground truth but expensive.

**Automated metrics and why they are unreliable.** FID, the long-standing image metric, contradicts human raters, mishandles distortions, and is biased and unstable at realistic sample sizes; CLIPScore measures text-image alignment but is gameable, and combining weak metrics does not make a strong one. FVD inherits FID's problems and is unstable at the small sample sizes typical of video. For audio, FAD is the FID analogue with the same caveats, and MOS (subjective 1-5) is ceiling-limited now that TTS approaches human quality. Use these as coarse signals, never as correctness.

**Human preference is the gold standard.** The field ranks image and video models with blind pairwise arenas scored by Elo or TrueSkill over many votes. Treat public arena standings as a coarse signal and validate on *your own* prompt distribution, the same discipline the [benchmarks](../14-evaluation-and-observability/03-benchmarks-and-leaderboards.md) chapter prescribes for LLMs.

**Regression testing for non-reproducible pipelines** is the durable practice, because providers silently update models behind stable names. Build a versioned golden prompt set covering your real use cases and failure modes; on hosted APIs drop exact match and compare *distributions* of metric scores; use perceptual similarity (perceptual hashing, SSIM, CLIP-embedding distance to a baseline) rather than equality; add a VLM-as-judge on a rubric (prompt adherence, artifacts, brand safety) for what hashes cannot see; track judge and metric distributions over time and alert on drift, which often means the upstream provider changed the model; and pin to dated model versions wherever the API allows. Gate releases on statistical significance, not raw deltas, and version prompts independently of models.

---

## The Model Landscape

Kept light and dated, because it churns monthly and much of the circulating spec-sheet detail comes from low-quality sources.

- **Image.** The open-weight frontier is led by the FLUX family (a recent release is a large rectified-flow transformer runnable on a single high-end consumer GPU with quantization, and notably needs no fine-tuning for character or style reference); the Stable Diffusion lineage remains the broad tooling base. The licensing gotcha to teach: open *weights* does not mean open *use*, since several variants (the `dev` line) are non-commercial and require a paid license for commercial work, while the distilled `schnell` variant is permissively licensed (Apache-2.0). Conditioning composes: ControlNet (structural control from pose, depth, edge, or segmentation maps), reference adapters (use an image as a prompt for style or likeness), inpainting and outpainting, and regional prompting; the production combo is an identity adapter plus structural control plus a text theme in one graph. LoRA is the default for style or subject personalization.
- **Video.** Proprietary API leaders (OpenAI's Sora 2, Google's Veo with native synchronized audio, plus strong contenders from ByteDance, Kuaishou, Runway, and others) sit above a fast-improving open-weight tier. A correction worth flagging, and a live example of how fast this section decays: OpenAI deprecated the Sora 2 / Videos API in March 2026 (with a reported shutdown later in 2026) and discontinued the consumer Sora app soon after, so verify a video model's status before building on it rather than trusting any snapshot, this chapter included. Billing is per second of output (broadly a few cents to under a dollar per second), generation takes tens of seconds to minutes, and outputs are non-reproducible, which forces async jobs, draft-then-render, and hard per-user cost caps.
- **Audio.** Voice and TTS (ElevenLabs is the reference, with cloning and dubbing) and music (Suno and Udio, at the center of ongoing copyright litigation and licensing settlements). Composition with video mirrors the voice chapter's split: native joint audio-video gives the best synchronization with the least control, while a cascaded approach (generate silent video, then add TTS and music and lip-sync separately) gives independent control and dubbing at the cost of effort, which is why most production dubbing pipelines stay cascaded.

---

## Interview Questions

### Q: Design the backbone of a service that turns a script into a narrated, music-backed video. What are the hard parts?

**Strong answer:**
The backbone is an asynchronous DAG of stages, each a swappable model: prompt to keyframe images, image-to-video for motion, TTS for narration, a music model for score, and lip-sync to bind voice to video, then a mux step. Because video generation takes tens of seconds to minutes, every stage is a job: submit returns an ID, work runs on an autoscaling GPU worker pool scaled on queue depth, and completion fires a webhook with a polling fallback. The hard parts are cost and reliability. Cost: cache on the full request fingerprint, draft cheaply and only re-render approved drafts at full quality, and cap per-user spend, since each retry costs real money. Reliability: attach idempotency keys so a retried or redelivered job does not pay twice, and store a full manifest of prompt, seed, parameters, and model versions per asset, both for debugging and as the provenance record. I would also gate generation with input and output moderation and attach C2PA credentials, because the synthetic-media obligations apply regardless of my domain.

### Q: How do you regression-test a generative pipeline when outputs are not reproducible?

**Strong answer:**
You give up exact match and test perceptually and statistically. I keep a versioned golden prompt set covering real use cases and known failure modes, and run it on every change. Where reproducibility holds, self-hosted with pinned hardware and seeds, I can use seed-locked checks; on hosted APIs I assume non-reproducibility, so I compare distributions of metric scores rather than single outputs and use perceptual similarity, perceptual hashes, SSIM, and CLIP-embedding distance to a baseline, plus a VLM-as-judge on a rubric for prompt adherence and artifacts. I track those distributions over time and alert on drift, because a sudden shift usually means the provider silently updated the model behind a stable name, so I also pin to dated model versions when the API allows. Releases gate on statistical significance, not raw deltas, and I version prompts independently of models so I can attribute a regression to the right change.

---

## References

- [C2PA specification](https://spec.c2pa.org/) (Content Credentials)
- "Invisible Image Watermarks Are Provably Removable Using Generative AI" (NeurIPS 2024, arXiv:2306.01953)
- Google DeepMind, [SynthID](https://deepmind.google/science/synthid/)
- "Rethinking FID: Towards a Better Evaluation Metric for Image Generation" (CVPR 2024) arXiv:2401.09603
- Black Forest Labs, [FLUX](https://bfl.ai/) and its [licensing](https://bfl.ai/licensing)
- EU AI Act [Article 50](https://artificialintelligenceact.eu/article/50/) (transparency for generated content)

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **DAG (Directed Acyclic Graph)** | A pipeline of stages where each stage feeds into the next with no cycles; used to model chained generative steps | Formalizes the creative pipeline (prompt → image → video → audio → mux) so any stage can be swapped independently |
| **Asynchronous Job / Async Queue** | A pattern where a long-running task is submitted and returns an ID immediately; the caller polls or receives a webhook when done | Mandatory for video generation (tens of seconds to minutes per clip) to avoid HTTP timeouts |
| **Webhook** | An HTTP callback sent by a server to a URL you provide when an event (like job completion) occurs | Replaces polling in async pipelines; client is notified rather than having to ask repeatedly |
| **Webhook Signature Verification** | Checking a cryptographic signature on an incoming webhook to confirm it came from the expected sender | Prevents malicious actors from spoofing job-completion events and injecting unauthorized content |
| **Idempotency Key** | A unique ID attached to a request so retries and duplicate deliveries are recognized and not processed twice | Prevents double-billing when a webhook is redelivered or a client retries a failed job |
| **GPU Worker Pool** | A set of GPU-equipped servers that execute generative model inference jobs pulled from a queue | The compute backbone of any image or video generation service; must autoscale on queue depth, not CPU |
| **Autoscaling on Queue Depth** | Adding or removing GPU workers based on how many jobs are waiting in the queue | Correct scaling signal for generative pipelines; CPU utilization is misleading for GPU-bound work |
| **Draft-then-Render** | Generating a cheap low-resolution preview first, then re-rendering only approved drafts at full quality | The single biggest cost lever in video pipelines; avoids paying full price for outputs that get rejected |
| **Request Fingerprint / Cache Key** | A hash of the full generation request (prompt + all parameters + seed + model version) used as a cache lookup key | Enables exact-hit response caching so identical requests are never billed twice |
| **Seed** | A number that initializes a random-number generator to make a generation locally reproducible | Makes a run reproducible on pinned hardware; does not guarantee cross-machine or cross-batch reproducibility |
| **Non-Reproducibility** | The property of hosted generative APIs where the same seed and prompt can yield slightly different outputs | Forces QA to compare distributions and perceptual similarity rather than exact pixel match |
| **Provenance Manifest** | A structured record of every parameter (prompt, seed, model version, conditioning inputs) stored with each generated asset | Required for debugging, regression testing, and compliance; also serves as the asset's provenance log |
| **C2PA (Coalition for Content Provenance and Authenticity)** | An open standard that cryptographically binds a creation history ("nutrition label") to a media asset | Enables platforms to display AI labels and lets users verify whether an image or video was AI-generated |
| **Content Credentials** | A C2PA-standard signed manifest embedded in or linked to a media file recording its creation chain | The practical implementation of C2PA; supported by major image and video generators and some camera makers |
| **Hard Binding** | A C2PA link based on a cryptographic hash of the media bytes; breaks if the file is re-encoded or resaved | Tamper-evident but fragile; removed by any re-encoding or screenshot |
| **Soft Binding / Durable Content Credentials** | A C2PA link based on a watermark or fingerprint that survives re-encoding; can recover a stripped manifest from a repository | Resilient fallback when hard bindings are broken by file conversion or social-media compression |
| **SynthID** | Google DeepMind's watermarking system that embeds an imperceptible signal in generated images, video, and text | Enables platform-level AI labeling and casual misuse detection; not robust against motivated adversaries |
| **Invisible Watermark** | A signal embedded in media that is imperceptible to humans but detectable by software | Durable against casual distribution but provably removable by regeneration attacks |
| **Regeneration Attack** | Adding noise to a watermarked image and then denoising it with a diffusion model to remove the watermark | Demonstrates that invisible watermarks alone cannot stop a motivated adversary |
| **FID (Fréchet Inception Distance)** | An automated metric that compares the statistical distribution of generated images to real images using a neural network | Long-standing benchmark for image quality; unreliable at small sample sizes and contradicts human ratings |
| **FVD (Fréchet Video Distance)** | The video analogue of FID, comparing generated and real video distributions | Inherits FID's instability; especially problematic at the small sample sizes typical of video evaluation |
| **CLIPScore** | A metric measuring alignment between a text prompt and a generated image using CLIP embeddings | Gameable; high CLIP similarity does not guarantee human preference |
| **FAD (Fréchet Audio Distance)** | The FID analogue for audio; compares generated and real audio distributions | Coarse signal for audio quality; subject to the same instability as FID |
| **MOS (Mean Opinion Score)** | A subjective 1–5 rating of audio quality collected from human listeners | Gold standard for TTS quality; now ceiling-limited as TTS approaches human quality |
| **Pairwise Arena / Elo** | An evaluation where human raters compare two generated outputs side-by-side; ratings update like chess Elo scores | The most reliable way to rank generative models; public arena standings are a coarse signal, not a substitute for domain evals |
| **SSIM (Structural Similarity Index)** | A perceptual image quality metric measuring luminance, contrast, and structure similarity | Used for regression testing when exact pixel match is impossible; more human-aligned than raw pixel difference |
| **Perceptual Hash** | A fingerprint of an image based on its visual content rather than raw bytes; similar images have similar hashes | Detects near-duplicate outputs in regression testing even after minor encoding changes |
| **VLM-as-Judge** | Using a vision-language model to score generated images or videos against a rubric | Automates quality evaluation at scale for attributes (prompt adherence, artifacts, brand safety) that hashes cannot measure |
| **CLIP Embedding Distance** | The cosine distance between CLIP embeddings of two images or an image and a text prompt | Measures semantic similarity; used in regression testing to detect distribution drift when providers silently update models |
| **ControlNet** | A conditioning adapter for diffusion models that guides generation using structural inputs like pose, depth, or edge maps | Enables precise structural control over image composition without modifying the base model |
| **LoRA (Low-Rank Adaptation)** | A fine-tuning technique that trains only a small set of adapter weights on top of a frozen base model | Default method for personalizing image generation to a specific style or subject |
| **Inpainting** | Generating new content inside a masked region of an existing image | Enables object replacement, background editing, and localized style changes |
| **Outpainting** | Extending an image beyond its original borders by generating new surrounding content | Used to reframe images or increase canvas size while maintaining visual coherence |
| **Rectified Flow Transformer** | A class of generative model architecture used in the FLUX family that learns straight-line flow paths between noise and data | Allows high-quality image generation in fewer inference steps than earlier diffusion models |
| **FLUX** | A family of open-weight image generation models built on rectified flow transformer architecture | Current open-weight frontier for image quality; `schnell` variant is Apache-2.0 licensed |
| **Stable Diffusion** | The broad family of open-weight latent diffusion image generation models | Largest ecosystem of fine-tunes, tools, and community resources for image generation |
| **Sora 2** | OpenAI's video generation model (deprecated March 2026) | Historical reference; illustrates how fast the video model landscape turns over |
| **Veo** | Google's video generation model with native synchronized audio | Competing with Runway and Sora-class models for highest-quality AI video output |
| **Lip-Sync** | Animating a face or avatar's mouth movements to match a given audio track | A stage in the narrated-video pipeline; preserves believability when TTS narration is added to video |
| **Mux (Multiplex)** | Combining separately generated video and audio streams into a single output file | The final stage in a DAG-based video pipeline after narration, music, and lip-sync are complete |
| **EU AI Act Article 50** | The transparency requirement in EU law mandating that providers mark AI-generated output in machine-readable form | Applies from August 2026; affects any system emitting or distributing synthetic media in the EU |
| **TAKE IT DOWN Act** | A 2025 US federal law targeting non-consensual intimate imagery including AI deepfakes, with platform takedown duties | Imposes removal obligations on platforms hosting non-consensual synthetic intimate imagery |
| **NSFW Classifier** | An automated model that detects not-safe-for-work content (adult, violent, or otherwise prohibited) in generated media | Applied at the output stage before returning an asset to the user; part of mandatory safety filtering |
| **ComfyUI** | A node-graph interface for building and versioning image and video generation pipelines | Codifies the DAG workflow as versioned, shareable graph state rather than UI configuration |

---

*Previous: [Real-Time Voice Agents](../18-voice-and-audio-agents/01-realtime-voice-agents.md)*
