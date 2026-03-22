---
layout: default
title: "How to build a voice agent with ElevenLabs API and Python"
description: "A practical developer tutorial to build and run a Python voice agent with ElevenLabs TTS, SpeechRecognition STT, and an optional OpenAI response layer."
permalink: /how-to-build-a-voice-agent-with-elevenlabs-api-and-python.html
article_card: true
article_order: 0
card_title: "How to Build a Voice Agent with ElevenLabs API and Python"
faq_jsonld: |
  {
    "@context": "https://schema.org",
    "@type": "FAQPage",
    "mainEntity": [
      {
        "@type": "Question",
        "name": "Can I build a voice agent with ElevenLabs and Python without a complex backend?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "Yes. You can start with a local pipeline: microphone input with SpeechRecognition, a response layer in Python, and speech output using ElevenLabs TTS."
        }
      },
      {
        "@type": "Question",
        "name": "Which ElevenLabs Python SDK method is used for text-to-speech in this tutorial?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "This tutorial uses ElevenLabs Python SDK with ElevenLabs client and client.text_to_speech.convert for generating audio from text."
        }
      },
      {
        "@type": "Question",
        "name": "Do I need OpenAI to run this voice agent tutorial?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "No. The project runs fully in rule-based mode. OpenAI is optional and can be enabled later with an API key."
        }
      },
      {
        "@type": "Question",
        "name": "What are the most common SpeechRecognition setup issues?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "The most common issues are missing PyAudio, blocked microphone permissions, and wrong default input device selection."
        }
      },
      {
        "@type": "Question",
        "name": "How can I move this project from local prototype to production?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "Split STT, brain, and TTS into separate services, add retries and monitoring, and optimize latency and cost with model routing."
        }
      }
    ]
  }
---

## 1) FTC disclosure

This article contains an ElevenLabs affiliate link. If you sign up through that link, I may earn a commission at no extra cost to you.

## 2) TL;DR: what we are building + stack

If you searched `build voice agent elevenlabs python`, you likely want a tutorial that can actually run, not architecture slides. In this guide, we build a complete local voice agent that listens from your microphone, generates a response, and speaks back using ElevenLabs.

The build is intentionally simple but production-minded. We keep clear boundaries between STT, response generation, and TTS so you can swap components later without rewriting the entire project.

If you are still deciding whether the platform is worth paying for after the prototype stage, [the full developer review goes deeper into quality, DX, and where ElevenLabs fits better than nearby alternatives](/ai-tools-review/elevenlabs-review-for-developers-api-pricing-latency-and-real-world-fit.html).

What you will have at the end:

1. A runnable CLI voice agent loop in Python
2. Real microphone input with `SpeechRecognition`
3. Real audio synthesis with ElevenLabs Python SDK
4. Two response modes: `rule` and optional `openai`
5. A test checklist to verify every stage locally

Stack used in this tutorial:

- Python 3.9+
- `elevenlabs` Python SDK for text-to-speech output
- `SpeechRecognition` + `PyAudio` for microphone input
- `python-dotenv` for local environment variables
- Optional `openai` SDK for LLM replies

Expected effort:

- Setup time: 15 to 25 minutes
- First successful end-to-end run: usually under 45 minutes
- Extension to your own use case: same day if you keep module boundaries clean

## 3) Prerequisites (Python 3.9+, ElevenLabs account, basic async knowledge)

Before writing files, make sure your local environment is realistic for audio work. Most project failures at this stage are not code logic failures. They are environment mismatches.

Required:

- Python `3.9` or newer
- An ElevenLabs account and API key
- Basic understanding of functions, classes, and command line usage
- Basic async awareness (you do not need advanced asyncio, but you should understand network calls can fail or time out)

System checks you should do first:

1. Confirm Python version:

```bash
python --version
```

2. Confirm microphone availability at OS level (outside Python first).
3. Confirm outbound internet access for API requests.

Why these prerequisites matter:

- Voice apps touch device hardware and network APIs in one flow.
- If microphone permissions are blocked, STT fails silently.
- If env vars are missing, SDK calls fail immediately.
- If you skip this validation, debugging time doubles.

## 4) Architecture overview (diagram text-based)

The architecture below is small enough for a tutorial and clean enough for a real MVP. Each block has one job.

```text
+-------------------+      +-----------------------+      +------------------+
|  Microphone Input | ---> |  STT Layer            | ---> |  Agent Brain     |
|  (device audio)   |      |  SpeechRecognition    |      |  rule or OpenAI  |
+-------------------+      +-----------------------+      +------------------+
                                                                   |
                                                                   v
                    +--------------------------------------------------------------+
                    | TTS Layer (ElevenLabs Python SDK)                            |
                    | client.text_to_speech.convert(...) -> save audio file        |
                    +--------------------------------------------------------------+
                                                                   |
                                                                   v
                                                       +--------------------------+
                                                       | Local Output (MP3 file)  |
                                                       | Playback or downstream    |
                                                       +--------------------------+
```

Design decisions and why they are practical:

- STT is isolated: you can replace Google Web Speech backend later.
- Brain is isolated: swap rule-based logic with an LLM without touching STT or TTS.
- TTS is isolated: replace voice model, output format, or provider with minimal impact.
- Output is file-based first: file output makes debugging deterministic before adding streaming.

For developers shipping real products, this separation is more important than adding features too early.

## 5) Step 1: Setup + install dependencies

Create a new project and virtual environment.

```bash
mkdir voice-agent-elevenlabs-python
cd voice-agent-elevenlabs-python
python -m venv .venv
```

Activate environment:

```bash
# Windows PowerShell
.venv\Scripts\Activate.ps1

# macOS/Linux
source .venv/bin/activate
```

Install dependencies:

```bash
pip install elevenlabs SpeechRecognition pyaudio python-dotenv openai
```

Create a `.env` file:

```env
ELEVENLABS_API_KEY=your_elevenlabs_api_key_here
ELEVENLABS_VOICE_ID=JBFqnCBsd6RMkjVDRZzb
ELEVENLABS_MODEL_ID=eleven_multilingual_v2
OPENAI_API_KEY=
OPENAI_MODEL=gpt-4.1-mini
```

Create `requirements.txt` for reproducibility:

```txt
elevenlabs
SpeechRecognition
pyaudio
python-dotenv
openai
```

Create this minimal project structure:

```text
voice-agent-elevenlabs-python/
  .env
  requirements.txt
  tts_engine.py
  stt_engine.py
  brain.py
  main.py
```

Why we keep only six files now:

- Lower setup friction for first run
- Easier debugging when you can inspect entire codebase quickly
- Cleaner path to refactor into packages once behavior is stable

## 6) Step 2: ElevenLabs TTS integration (verified Python SDK snippet)

This section uses the verified ElevenLabs Python SDK pattern with `ElevenLabs` client and `client.text_to_speech.convert(...)`.

Create `tts_engine.py`:

```python
import os
from pathlib import Path
from typing import Optional

from dotenv import load_dotenv
from elevenlabs import save
from elevenlabs.client import ElevenLabs


load_dotenv()


class TTSEngine:
    def __init__(self) -> None:
        api_key = os.getenv("ELEVENLABS_API_KEY", "").strip()
        if not api_key:
            raise ValueError("Missing ELEVENLABS_API_KEY")

        self.client = ElevenLabs(api_key=api_key)
        self.voice_id = os.getenv("ELEVENLABS_VOICE_ID", "JBFqnCBsd6RMkjVDRZzb")
        self.model_id = os.getenv("ELEVENLABS_MODEL_ID", "eleven_multilingual_v2")

    def synthesize_to_file(self, text: str, output_path: Optional[str] = None) -> str:
        if not text.strip():
            raise ValueError("Text cannot be empty")

        audio = self.client.text_to_speech.convert(
            voice_id=self.voice_id,
            text=text,
            model_id=self.model_id,
            output_format="mp3_44100_128",
        )

        if output_path is None:
            output_path = str(Path("output_reply.mp3").resolve())

        save(audio, output_path)
        return output_path


if __name__ == "__main__":
    engine = TTSEngine()
    path = engine.synthesize_to_file("Hello from your Python voice agent.")
    print(f"Audio saved to: {path}")
```

Run a direct TTS smoke test:

```bash
python tts_engine.py
```

If `output_reply.mp3` is created, your TTS integration is valid. Do this before adding STT and agent logic so failures stay isolated.

Once that smoke test works, [this production-oriented pricing breakdown helps you estimate what happens when the tutorial starts turning into real monthly usage](/ai-tools-review/elevenlabs-api-pricing-explained-for-developers.html).

- [Try ElevenLabs free ->](https://try.elevenlabs.io/64xtbcqe19bi?utm_source=blog&utm_medium=article&utm_campaign=voice-agent-tutorial&utm_content=tutorial-cta)

## 7) Step 3: Add STT input layer (SpeechRecognition library)

Now we add speech input. The goal here is robust microphone capture with clear error behavior.

Create `stt_engine.py`:

```python
import speech_recognition as sr


class STTEngine:
    def __init__(self) -> None:
        self.recognizer = sr.Recognizer()

    def listen_once(self, timeout: int = 8, phrase_time_limit: int = 12) -> str:
        with sr.Microphone() as source:
            print("Calibrating ambient noise...")
            self.recognizer.adjust_for_ambient_noise(source, duration=0.7)
            print("Listening now...")
            audio = self.recognizer.listen(
                source,
                timeout=timeout,
                phrase_time_limit=phrase_time_limit,
            )

        try:
            transcript = self.recognizer.recognize_google(audio)
            return transcript.strip()
        except sr.UnknownValueError:
            return ""
        except sr.RequestError as exc:
            raise RuntimeError(f"STT backend request failed: {exc}") from exc


if __name__ == "__main__":
    stt = STTEngine()
    text = stt.listen_once()
    if text:
        print(f"Transcript: {text}")
    else:
        print("No recognizable speech captured.")
```

Run STT test:

```bash
python stt_engine.py
```

Expected behavior:

- You see calibration and listening logs
- You speak one short sentence
- Transcript is printed in terminal

If transcript quality is poor, tune `duration`, `timeout`, and `phrase_time_limit` before moving on.

## 8) Step 4: Connect LLM response layer (OpenAI or simple rule-based)

This module keeps your decision logic independent from speech IO. We support two modes:

- `rule`: no extra API cost, easiest for local validation
- `openai`: better quality responses for real use cases

Create `brain.py`:

```python
import os
from typing import Literal

from dotenv import load_dotenv


load_dotenv()

Mode = Literal["rule", "openai"]


class AgentBrain:
    def __init__(self, mode: Mode = "rule") -> None:
        self.mode = mode
        self.openai_client = None
        self.model = os.getenv("OPENAI_MODEL", "gpt-4.1-mini")

        if mode == "openai":
            api_key = os.getenv("OPENAI_API_KEY", "").strip()
            if not api_key:
                raise ValueError("OPENAI_API_KEY is required for openai mode")

            from openai import OpenAI

            self.openai_client = OpenAI(api_key=api_key)

    def reply(self, user_text: str) -> str:
        clean_text = user_text.strip()
        if not clean_text:
            return "I could not hear you clearly. Please try again."

        if self.mode == "openai":
            return self._openai_reply(clean_text)

        return self._rule_reply(clean_text)

    def _rule_reply(self, text: str) -> str:
        t = text.lower()

        if "hello" in t or "hi" in t:
            return "Hello. I am your local developer voice agent."
        if "status" in t:
            return "System status: local pipeline running with SpeechRecognition and ElevenLabs."
        if "weather" in t:
            return "I do not have live weather yet. Next step is adding a weather API tool."
        if "quit" in t or "exit" in t:
            return "Acknowledged. Say goodbye and close the session."

        return f"You said: {text}. I am currently in rule mode."

    def _openai_reply(self, text: str) -> str:
        assert self.openai_client is not None

        response = self.openai_client.responses.create(
            model=self.model,
            input=(
                "You are a concise voice assistant for software developers. "
                "Answer with practical steps and keep replies under 3 short sentences.\n\n"
                f"User: {text}"
            ),
        )

        output = (response.output_text or "").strip()
        if output:
            return output
        return "I generated an empty reply. Please ask again with more context."
```

Why this design is useful in real projects:

- rule mode keeps your local loop usable during API issues
- OpenAI mode is optional, not a hard dependency
- one `reply()` contract means downstream TTS code never changes

## 9) Step 5: Run and test locally

Now wire all modules into one executable loop.

Create `main.py`:

```python
import argparse
from pathlib import Path

from brain import AgentBrain
from stt_engine import STTEngine
from tts_engine import TTSEngine


def run(mode: str) -> None:
    stt = STTEngine()
    brain = AgentBrain(mode=mode)
    tts = TTSEngine()

    print(f"Voice agent started in {mode} mode")
    print("Speak naturally. Say 'quit' or 'exit' to stop.")

    while True:
        try:
            user_text = stt.listen_once()
        except Exception as exc:
            print(f"STT error: {exc}")
            continue

        if not user_text:
            print("No speech recognized. Try again.")
            continue

        print(f"User: {user_text}")
        reply_text = brain.reply(user_text)
        print(f"Agent: {reply_text}")

        output_file = Path("output_reply.mp3").resolve()
        try:
            saved = tts.synthesize_to_file(reply_text, str(output_file))
            print(f"Audio reply saved: {saved}")
        except Exception as exc:
            print(f"TTS error: {exc}")

        low = user_text.lower()
        if "quit" in low or "exit" in low:
            break

    print("Voice agent stopped")


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--mode", choices=["rule", "openai"], default="rule")
    args = parser.parse_args()
    run(args.mode)


if __name__ == "__main__":
    main()
```

Run in rule mode first:

```bash
python main.py --mode rule
```

If rule mode works, run OpenAI mode:

```bash
python main.py --mode openai
```

Local validation checklist (do not skip):

1. STT captures speech reliably in at least 5 consecutive turns
2. Rule mode returns expected responses for known trigger phrases
3. `output_reply.mp3` is generated for each turn
4. OpenAI mode works without changing any STT/TTS code
5. Process exits cleanly when user says `quit`

At this point, you have a working voice-agent foundation that can be demoed to users or teammates.

## 10) Common errors + how to fix

This section saves real debugging time. Most issues are not complex, but they happen frequently.

### Error: `ModuleNotFoundError: No module named 'pyaudio'`

Cause:

- OS-specific audio build dependencies are missing
- virtual environment is not activated

Fix:

1. Verify active environment path with `where python` or `which python`
2. Reinstall `pyaudio` in the active environment
3. If install fails on Windows, install build tools and retry

### Error: `Missing ELEVENLABS_API_KEY`

Cause:

- key not present in `.env`
- typo in variable name
- script runs from a different working directory

Fix:

1. Confirm `.env` is in project root
2. Confirm exact key name: `ELEVENLABS_API_KEY`
3. Add a quick debug print of current working directory when needed

### Error: empty transcript even when microphone works

Cause:

- ambient noise calibration is too short
- speaking starts too late or too quietly

Fix:

1. Increase `adjust_for_ambient_noise(..., duration=1.0)`
2. Increase `timeout` and `phrase_time_limit`
3. Move microphone closer and reduce background noise

### Error: `STT backend request failed`

Cause:

- network issue or temporary upstream API limit

Fix:

1. retry with exponential backoff
2. keep a fallback input path (typed text mode)
3. log failure counts so you can see if issue is systemic

### Error: OpenAI mode returns auth or quota error

Cause:

- invalid key, missing billing, or wrong project config

Fix:

1. run in `--mode rule` to keep product development unblocked
2. verify `OPENAI_API_KEY` and account quotas
3. add a startup check that validates OpenAI connection once

### Error: generated MP3 exists but playback is inconsistent

Cause:

- local player integration not implemented
- file locking on repeated writes

Fix:

1. confirm file plays manually first
2. write timestamped files during testing
3. only add auto-play after synthesis is stable

## 11) Next steps: deploy, scale, swap components

Once local testing is stable, move from prototype to a service that survives real traffic.

### Phase A: Harden reliability

- Add structured logs (`stage`, `latency_ms`, `error_type`)
- Add retries with bounded backoff for STT and TTS calls
- Add timeout budgets per stage so one slow call does not freeze the loop

### Phase B: Reduce latency and cost

- Route short responses to faster/cheaper TTS models
- Cache repeated prompts and repeated TTS outputs when appropriate
- Add simple response length controls so cost does not drift

### Phase C: Service architecture

- Split into `stt-service`, `brain-service`, and `tts-service`
- Add queue-based processing for burst handling
- Use centralized monitoring for latency, error rate, and quality metrics

### Phase D: Product-level capabilities

- Add user session memory and conversation context policies
- Add guardrails for unsafe requests
- Add analytics for conversion and retention events if this is customer-facing

A practical shipping strategy is: local monolith -> single API service -> modular services only when traffic demands it. That path keeps complexity proportional to real usage.

If you think your roadmap is likely to become streaming-first rather than file-generation first, [the comparison with PlayHT shows where that alternative starts to make more architectural sense](/ai-tools-review/elevenlabs-vs-playht-for-developers-which-tts-api-should-you-pick.html).

- [Try ElevenLabs free ->](https://try.elevenlabs.io/64xtbcqe19bi?utm_source=blog&utm_medium=article&utm_campaign=voice-agent-tutorial&utm_content=tutorial-cta)

## 12) FAQ

### Can I build this without OpenAI?

Yes. Rule mode is enough to run a full microphone -> response -> TTS loop. OpenAI is optional.

### Which ElevenLabs method is used here?

This tutorial uses the verified Python SDK call: `client.text_to_speech.convert(...)`.

### Should I deploy as one app or multiple services first?

Start with one app for speed. Split services only after you have real usage and clear bottlenecks.

### What should I optimize first for production?

Reliability before intelligence: retries, timeouts, logs, then quality tuning.

### Can I swap SpeechRecognition later?

Yes. Because STT is isolated in `stt_engine.py`, you can replace it with another provider while preserving the rest of the pipeline.
