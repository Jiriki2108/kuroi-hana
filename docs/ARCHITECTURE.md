# Kuroi Hana — System Architecture

## Architecture at a Glance

Kuroi Hana is a multi-process, hybrid local-cloud AI assistant that runs on a single NVIDIA RTX 5090. The brain is in Python (`Hana_Project`), the body is in Unity (`Hana_Unity`), and they communicate exclusively through WebSocket messages over port 8001.

There are two separate Python processes that communicate through atomic file-based IPC (temp file → rename):

| Process | Role |
|---------|------|
| `server.py` (port 8001) | FastAPI + WebSocket hub — relays everything between Python and Unity |
| `main_chat.py` | The brain loop — VAD → STT → LLM → TTS → animation commands |

### Full Pipeline (Speech → Response → Animation)

```
Mic → Silero VAD detects speech
  → records utterance
  → Groq Whisper Large v3 Turbo transcribes
  → echo detection (4 layers: trigram, semantic, MFCC, acoustic)
  → context building (12 context blocks injected)
  → Router LLM decides if a tool is needed
  → Responder LLM generates Hana's reply with emotion tags
  → TTS text cleaned of asterisk actions and tags
  → CosyVoice3 with emotion-instruct streaming synthesis
  → audio chunks pushed to AEC reference buffer
  → chunks sent via WebSocket to Unity
  → Unity AudioStreamingManager decodes/resamples/plays through a gapless ring buffer
  → FFT-driven VRC viseme lipsync
  → PlaybackWorker fires deferred avatar actions in sync with audio
  → playback_finished signal
  → cycle repeats
```

---

## LLM Orchestration

Currently running on DeepSeek V4 Flash (cloud) for the main responder, router, and janitor. Local LLMs (Qwen3-4B for persona, Qwen2.5-0.5B for tool routing, Qwen2.5-1.5B for memory classification) are trained and ready but currently disabled due to an import conflict between Unsloth and CosyVoice3 in the same conda environment. The GGUF files are all Q4_K_M quantized and the llama.cpp binaries are ready to serve them.

| Model | Role | Temperature |
|-------|------|-------------|
| DeepSeek V4 Flash | Primary Responder | Default |
| DeepSeek V4 Flash | Router (tool selection) | 0 |
| DeepSeek V4 Flash | Janitor (memory classification) | 0 |
| DeepSeek V4 Pro | Proactive/Ambient Responses | Default |

---

## The 12-Tool Agentic System

Hana can independently decide to use tools via OpenAI-compatible function calling. They're split into two tiers:

### Non-destructive (always on)

| Tool | Description |
|------|-------------|
| `examine_screen` | Qwen2.5-VL vision summary of current screen |
| `read_verbatim_text` | Tesseract OCR on IDE window |
| `web_search` | 4-tier fallback: Tavily → Serper → LangSearch → DuckDuckGo |
| `query_memory` | ChromaDB semantic retrieval |
| `read_clipboard` | Clipboard content reading |
| `read_project_file` | Read files within approved directories |
| `append_hana_note` | Write to Hana's personal notes |

### Destructive (gated behind Unity toggle or Ctrl+Shift+D)

| Tool | Description |
|------|-------------|
| `write_clipboard` | Set clipboard contents |
| `copy_url_to_clipboard` | Copy URL to clipboard |
| `media_control` | Play/pause/skip media |
| `set_volume` | System volume control |
| `edit_project_file` | Edit files within approved directories |

### Spatial tools

| Tool | Description |
|------|-------------|
| `move_to_object` | Navigate to a named 3D object |
| `pick_up_object` | Pick up a named 3D object |
| `bring_to_user` | Full sequence: walk → pickup → walk → present |

All tools have 8-second hard timeouts. File tools are path-validated — they can only touch files inside `D:\Hana_Project\` or `D:\Hana_Unity\`.

---

## The Drive System

Hana has four internal drives that update every 60 seconds:

| Drive | Behaviour | Range |
|-------|-----------|-------|
| **Boredom** | Rises with silence (+8/min idle), drops on interaction (-30) | 0–100 |
| **Energy** | Follows a Singapore-time circadian curve (peaks 80–85 evening, dips to 20 at 3am) | 0–100 |
| **Curiosity** | Spikes +15 on screen changes, decays naturally | 0–100 |
| **Affection** | Rises +1/min during sessions (capped 90) | 0–100 |

These drives feed into a sigmoid-based proactive speech probability formula. When boredom is high, she'll speak unprompted. When energy is low, she'll sound tired (via TTS delivery hints). When affection is high, she might autonomously pick up a plushie to hug.

### The Mood Anchor System

Tracks 51 emotion tags mapped to 8 emotion clusters. An EMA (exponential moving average) tracks how far Hana has drifted from her default "playful/affectionate" home cluster. If the drift exceeds thresholds, awareness paragraphs are injected — "you've been more distant than usual" — but always as facts, never as behavioral directives. This is the Reactive Identity principle: the system surfaces awareness; Hana's personality decides what to do with it.

---

## The Memory System — Multi-Layered Retrieval

Hana remembers things through five distinct retrieval paths:

### 1. Typed ChromaDB Memory (4 collections)

| Collection | Contents | TTL |
|------------|----------|-----|
| CORE | Permanent identity facts | None |
| SEMANTIC | Preferences, observations | 30 days |
| EPISODIC | Conversation events | 14 days |
| PROCEDURAL | Learned behaviour patterns with confidence levels | 60 days |

A janitor LLM classifies conversations after every ~15 unclassified messages.

### 2. FAISS Dense Semantic Search

Last 200 messages using SentenceTransformer embeddings with recency weighting (7-day half-life) and semantic deduplication.

### 3. Keyword Chat History Scan

Last 100 messages with named entity matching.

### 4. Cultural/Trending Context

Reddit RSS + JSON API for gaming, anime, programming, and tech headlines — refreshed hourly, semantically queried via in-memory ChromaDB, filtered by current screen category.

### 5. Object Permanence

A JSON manifest of 10 objects in Hana's 3D room. She knows where things should be (expected zone vs actual zone). When her bear plush is on the bed instead of the shelf, she notices and might comment.

All retrieval paths inject raw facts into the system prompt — never behavioral instructions. Hana decides what to do with the information.

---

## The Perception Engine

Runs on a 3-second poll cycle as a background daemon thread.

### Sensors

| Sensor | Technology | Output |
|--------|------------|--------|
| Foreground app | Win32 process name + window title, 34-category mapped | App category |
| Background apps | Windows Alt-Tab switcher filtering | Up to 6 apps |
| Browser URL | UIAutomation (BFS control tree walk, COM fallback) | Full URL |
| Idle time | Win32 GetLastInputInfo | Seconds idle |
| Clipboard | Win32 clipboard API | Text/URL/code classification |
| Screenshots | mss + histogram change detection (64×36 grayscale, cv2 HISTCMP_CORREL, 0.92 threshold, 2s debounce) | Triggered capture |
| Vision | Qwen2.5-VL-3B (local GPU) primary, Qwen3-VL-30B (DeepInfra cloud) fallback | Natural-language screen summary |
| OCR | Tesseract (cropped to editor pane, 2× scale, PSM 6, ≤1500 chars) | IDE text extraction |
| Audio output | WASAPI loopback | Silence/quiet/moderate/loud |
| Avatar mirror | Unity camera captures Hana's appearance every 5 seconds, Qwen describes it | Self-description |

### Privacy

- `Ctrl+Shift+P` toggles pause on the perception engine
- `privacy_blacklist.txt` blocks specific domains
- Incognito/private browser windows are auto-detected and suppressed

---

## The Unity Frontend — The Body

Hana's avatar ("Nibbles (FBX)") has exactly one Animator parameter: `State` (Int, 0–4: Idle/Listening/Thinking/Talking/Custom). No trigger parameters — everything goes through WebSocket messages.

### 22 WebSocket Message Types (Python → Unity)

**Face/Expression:** `set_expression` (48 emotions), `set_expression_blend` (dual-emotion with ratio), `lip_sync_amplitude`, `set_boredom_level`

**Body Language:** `set_state`, `set_intent` (9 gesture clips: Tease, Comfort, Sulking, Observe, Confess, React, Dismiss, Explain, Conspire), `set_attention_target`, `set_gaze_target`

**Audio:** `audio_chunk`, `stream_reset`, `stop_audio`

**Spatial:** `move_to`, `cancel_move`, `pick_up`, `drop`, `present`

### Key C# Systems (16 scripts)

**Animation Priority:** Three system-wide priority gates — GestureActiveGlobal, MovementPriority, and InteractionPriority — prevent animation conflicts. When a gesture is playing, every other system (procedural head animation, expression bones, base layer, arm IK) knows to stay out of the way.

**Head/Eye Procedural Animation (HanaAnimationManager):** State-specific behaviour — Idle wanders freely, Listening angles toward camera, Thinking looks up and to the side, Talking faces forward with gentle nod. A spine sway (2.5°, 0.4 Hz) runs continuously as biological idle motion.

**Blendshape Expression System (HanaExpressions):** Maps 48 named emotions to 20+ blendshape indices on the head mesh. Emotional residue — 20% of the outgoing expression's actual displayed values are captured and decay over ~10 seconds, creating organic ghost transitions. Secondary emotion blending for compound expressions like `[SMUG|EMBARRASSED:0.3]`.

**Audio Pipeline (AudioStreamingManager):** The most complex pipeline in Unity. Chunks arrive as WAV files, decoded on background threads, resampled from 24kHz to 48kHz, fed into a lock-protected ring buffer (20s capacity), and played gaplessly. FFT spectrum analysis drives 15 VRC viseme blendshapes in real time. Backpressure prevents buffer overflow.

**Custom Spring-Bone Physics (HanaSpringBones):** Offset-based Verlet integration in rest-relative space with ~90+ chains covering hair, ears, tail, wings, and accessories. Survived a multi-week rewrite because the initial approach (world-space simulation) fought the Humanoid retargeting system, causing springs to drift every frame. The fix: simulate offsets from rest positions, not absolute positions. Contains a teleport guard, 20-frame settle phase on startup, and auto-rebuilders for tuning.

---

## Audio Pipeline Details

### STT (Speech-to-Text)

Silero VAD (32ms frames) → Groq Whisper Large v3 Turbo with custom prompt vocabulary ("Hana, senpai, Jubei").

### TTS (Text-to-Speech)

CosyVoice3 (Fun-CosyVoice3-0.5B-hana-sft) — a fine-tuned 0.5B parameter model from Alibaba. True streaming via `stream=True`. 51+ emotion tags mapped to English instruct strings passed before the `<|endofprompt|>` token. Zero-shot voice cloning from an ElevenLabs reference clip. High-priority CUDA stream prevents other GPU work from starving TTS.

Hana's voice originated from ElevenLabs (36 training utterances, "Yandere" voice profile), then was cloned into CosyVoice3 via zero-shot prompting, then SFT fine-tuned on top for stronger identity.

### AEC (Acoustic Echo Cancellation)

SpeexDSP NLMS adaptive filter with 8192-sample filter length, auto-tuned delay compensation derived from chirp tests (800ms default covering OS + NVIDIA Broadcast + Chrome WebAudio paths).

---

## Training & Fine-Tuning

Hana's personality was trained via Full Parameter Fine-Tuning (not LoRA) on Qwen3-4B-Instruct-2507 using Unsloth.

**Critical design decision:** `train_on_responses_only()` — the model is trained on Hana's dialogue output, NOT on the 12 massive context injection blocks (vision, perception, drives, memory, etc.), which are only injected at inference time.

**Training data:** ~23,000 Gemini-generated synthetic examples across 4 parallel generators, plus real conversation pairs extracted from chat history. Scrubbed for banned catchphrases to prevent overfitting. The router model was separately trained on Qwen2.5-0.5B with balanced tool call datasets.

Response masking is what makes it work — the model learned Hana's voice and the emotion tag format without memorizing dynamic context.

---

## Engineering Summary

Ignoring the aesthetic presentation layer entirely, this project integrates:

- Multi-process architecture with atomic file-based IPC
- Multi-model AI orchestration (responder + router + janitor + vision)
- Streaming audio pipeline with real-time lipsync
- Custom physics simulation (Verlet spring bones, ~90+ chains)
- Procedural animation system with emotion-driven blendshapes
- Drive-based autonomous behaviour engine
- Multi-layered semantic memory with two vector databases (FAISS + ChromaDB)
- OS-layer perception engine with Win32 API, UIAutomation, and COM interop
- Agentic tool-calling with destructive/non-destructive security gating
- Voice cloning pipeline spanning three TTS technologies
- Full Parameter Fine-Tuning of a 4B parameter language model
- Synthetic data generation at scale (~23K examples)
- Acoustic echo cancellation with auto-tuned delay compensation
