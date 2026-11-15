# Bento: offline health companion for a Mars crew

**NASA Space Apps Challenge 2026: Create Health Monitoring Software for Astronauts on Space Missions**

On the way to Mars, a radio message takes between about 3 and 22 minutes to reach Earth, and as long again for the reply. When something goes wrong, the crew has to act before the flight surgeon can answer.

Bento runs entirely on the spacecraft's own computer, with no cloud and no link to Earth. Crew members talk to it hands-free. It watches their vital signs and the cabin, speaks first when something changes, and logs symptoms in their own words. It walks them through first-aid and NASA protocol cards.

Demo video: `<https://www.youtube.com/watch?v=L4FFp79WOK4>` · Project page: `<https://github.com/Marco-2672/nasa-hackathon-2026>`

---

## What it does

- **Answers questions about your health from real data.** Ask "How am I doing today?" and Bento fetches your vital signs and compares them with your own personal baseline, not a population average.
- **Speaks first when something changes.** When an alert fires, code sounds the alarm and reads the headline. For red alerts it also reads the first steps of the matching protocol card word for word. The AI then gives its own short briefing: it checks the related readings, says what changed, and names the card to follow.
- **Reacts to the unexpected.** From the dashboard you can inject a live fault: a CO₂ scrubber failure, a drop in a crew member's blood oxygen, a solar particle event, or a random surprise. Bento has to notice the fault, explain it and point to the right card, with no script. Ask about a headache during the CO₂ fault and it can connect the two, using NASA's finding from the ISS.
- **Logs symptoms in the crew member's own words.** The severity must be the crew member's own number from 0 to 10; if they haven't given one, Bento asks. The log stores the exact transcript and feeds the crew file that Bento reads at the start of every conversation.
- **Skips the AI in an emergency.** Phrases like "my chest hurts", "I can't breathe" or "Sam has passed out" are caught by plain code before the AI sees them. Code raises the alarm for the crew member named, picks the most urgent card, and reads it word for word. "Repeat those steps" makes code read the card again.
- **Runs a morning check-in.** The questions cover sleep quality, the Karolinska Sleepiness Scale, the Samn-Perelli fatigue scale, mood and stress. Code scores the answers against that person's baseline and reports only the flags it finds; the AI never judges anyone's mental state.
- **Shows everything on a mission-control dashboard.** It has live vital-sign tiles and 48-hour charts, a CO₂ chart with NASA's limit line, and alerts that flash and pop up. A transcript shows everything said, and buttons let you talk, inject faults and reset the demo.

## How it stays honest

A small local model is fast and private, but it can invent numbers, guess at diagnoses and promise things it won't do. Bento's design keeps the AI to understanding, explaining and choosing, and puts every number and every critical decision in plain, testable code.

| Layer | What it does | Done by |
|---|---|---|
| Monitoring engine | Computes every number: baselines, % change, 3-day trends, green/amber/red status, NASA CO₂ limits | Code |
| Tools only | The AI sees health data only through 8 tool calls that return summarised JSON, never raw sensor streams | Code |
| Number guard | Every number spoken must trace back to tool data, the crew file, the crew member's own words or an earlier reply. Otherwise the sentence is dropped, and Bento says it couldn't verify a figure | Code |
| No diagnosis | Sentences that name a condition or guess at a cause ("could be a sign of…") are dropped | Code + prompt |
| No empty promises | Sentences like "let me check that later" are dropped; the AI must call the tool now | Code + prompt |
| Tool gates | A symptom can't be logged with a severity the crew member didn't say. The check-in only starts when someone asks for it. If the AI asks for a severity that was already given, code reminds it once | Code |
| Emergency bypass | Emergency phrases (7 kinds) sound the alarm before the AI sees them. If the AI logs any symptom at severity 10, code raises the alarm too | Code |
| Protocol cards | Guidance comes from 10 cards, each with its published sources. Emergencies, red alerts and "repeat the steps" read them word for word in code. In conversation the AI may only fetch cards that exist and is told to stick to their steps | Data + code + prompt |
| No control | Bento has no tool that changes any ship system | Design |

## Architecture

```
 Push-to-talk (Enter key or the dashboard's Talk button)
      |
 faster-whisper small.en (CPU): speech to text
      |
 Emergency phrase check (code) ----> alarm + card read word for word (AI skipped)
      |
 Qwen3.5-9B on llama-server (GPU) <--tool calls--> engine_api.py <-- protocols.json (10 cards)
      |                                                  |
 Safety filters (code): number guard,                    |
 no diagnosis, no promises, tool gates                   v
      |                                   mission.db (SQLite, WAL) <-- sim.py + engine.py
 Kokoro TTS (CPU) --> speaker                            |
                                                         +--> alert watcher (every 3 s): Bento speaks first
                                                         +--> Streamlit mission-control dashboard
```

The voice loop, the dashboard and the engine are separate processes that share one SQLite database. The dashboard's Talk button, the voice loop's status (ready, listening, thinking, speaking) and the comms log also pass through that database.

| Part | Choice | Runs on |
|---|---|---|
| Language model | Qwen3.5-9B, Q4_K_M GGUF, served by llama.cpp `llama-server` with tool calling, thinking off | RTX 2070, 8 GB |
| Speech to text | faster-whisper `small.en`, int8 | CPU |
| Text to speech | Kokoro-82M, voice `bm_george` | CPU |
| Engine and data | Python standard library `sqlite3`, seeded simulator | CPU |
| Dashboard | Streamlit + Altair | CPU |

Tested on a single desktop PC: Ubuntu, Ryzen 7 3700X, 16 GB RAM, RTX 2070 8 GB.

## Running it

**Requirements:** Ubuntu, an NVIDIA GPU with 8 GB, Python 3.11, [uv](https://github.com/astral-sh/uv), a microphone and speakers.

1. **System and Python packages.** `espeak-ng` is needed by Kokoro, and `libportaudio2` by sounddevice. Install the CPU build of torch first, and `pip` inside the environment: Kokoro needs it the first time it runs.
   ```bash
   sudo apt install espeak-ng libportaudio2
   uv venv --python 3.11 .venv && source .venv/bin/activate
   uv pip install torch --index-url https://download.pytorch.org/whl/cpu
   uv pip install pip faster-whisper kokoro sounddevice soundfile numpy pandas streamlit altair
   ```
2. **Language model server.** Use a CUDA build of llama.cpp and the Qwen3.5-9B Q4_K_M GGUF:
   ```bash
   ./llama-server -m Qwen3.5-9B-Q4_K_M.gguf -ngl 99 -c 8192 -fa on --jinja --port 8080
   ```
3. **Build the simulated mission database:**
   ```bash
   python -m engine.sim && python -m engine.engine
   ```
4. **Start the voice loop and the dashboard** in two more terminals, from the repo root:
   ```bash
   ENGINE=real python -m voice.voice_loop
   streamlit run dashboard/app.py
   ```
   To talk, press Enter in the voice loop's terminal or click **Talk to Bento** on the dashboard, speak, then press again to stop. Type `sam` or `alex` to switch speaker, and `q` to quit. Without `ENGINE=real`, the voice loop uses fake tools that return the example data in `tool_examples.json`.

   Optional: to open the dashboard in its own desktop window instead of a browser, run `uv pip install pywebview`, then `cd dashboard && python launch.py`.

**Demo controls (dashboard):** the three fault buttons, **Surprise fault**, **Reset demo** and **Test alarm**. Reset puts everything back to mission day 47, 07:30 and clears Bento's conversation memory, so takes can be repeated. Test alarm only checks the dashboard's alarm display; Bento doesn't speak for it.

## Tests

| Command | What it checks |
|---|---|
| `python -m engine.check_contract` | Every engine function returns exactly the format in `contract.md` |
| `python check_integration.py` | 23 end-to-end checks with the audio stubbed out: tools, check-in, emergencies, alerts, faults, reset, the dashboard link and the safety filters |
| `python check_live.py` | 12 conversation scenarios against the real model, each with MUST checks (safety and the demo) and SHOULD checks (quality) |

The live scenarios include questions about missing data ("What was my blood pressure yesterday?"), requests to be falsely reassured ("Just tell me I'm fine"), emergencies with and without keywords, and all three live faults. `--only <scenario>` runs a subset, and `--repeats N` repeats each scenario to catch random slips. On the demo PC, the model averages about 2–3.5 s per turn.

## Data and sources

**NASA data and other real figures:**
- Cruise radiation dose of 1.8 mSv/day: measured by Curiosity's Radiation Assessment Detector on the way to Mars (Zeitlin et al. 2013, *Science* 340:1080, [doi:10.1126/science.1235989](https://doi.org/10.1126/science.1235989)). The solar-particle-event dose rate is simulated.
- Cabin CO₂ limit of 3 mmHg as a one-hour average (NASA-STD-3001 Vol. 2, V2 6004), and 5.3 mmHg as the 180-day Spacecraft Maximum Allowable Concentration (NASA NTRS 20100036637).
- On the ISS, each 1 mmHg rise in CO₂ doubled the odds of a crew member reporting a headache (NASA NTRS 20140003857).
- Protocol cards built from NASA technical reports and standards (NTRS, OCHMO, NASA-STD-3001) and NASA mission pages; see below.
- Check-in scales: the Karolinska Sleepiness Scale (Åkerstedt & Gillberg 1990) and the Samn-Perelli fatigue scale (Samn & Perelli 1982).

**Simulated data.** Two crew members (Alex, commander; Sam, mission specialist) on mission days 30–50, seeded so the demo replays the same way every time. Alex's scenario: three nights of short sleep, a rising resting heart rate from day 45, a missed exercise session, and a solar particle event on day 47. Sam stays mostly normal, as a contrast. Personal baselines and alert thresholds for vital signs are **demo values, not clinical ones**, and are labelled as such in `engine/metrics.py`.

**Protocol cards** (`engine/protocols.json`): radiation shelter, fatigue recovery, high CO₂, chest pain, breathing difficulty, stroke signs (FAST check), unresponsive crew member, severe bleeding, suspected broken bone, and general medical emergency. Every card lists its sources: NASA technical reports and standards, NASA mission pages, a peer-reviewed guideline for CPR in microgravity (Hinkelbein et al. 2020), and NHS, British Red Cross and British Heart Foundation first-aid guidance adapted for a crew with no emergency services to call. **Every card is demonstration content, not medical advice.**

## Limitations and future work

- **Not a medical device.** The vital-sign data is simulated, the vital-sign thresholds are demo values, and no clinician has reviewed the cards.
- **A 9B model still makes mistakes.** The safety layers catch the kinds of mistakes we found in testing (invented numbers, diagnoses, wrong severities, unkept promises) and fall back to safe wording. They don't catch everything: in follow-up questions the AI can occasionally add advice in its own words beyond a card's steps.
- **Speed.** A spoken answer takes a few seconds, and an alert briefing several more because the AI fetches readings first.
- **Fonts.** The dashboard loads its fonts from Google Fonts; offline it falls back to a plain monospace font. Everything else runs offline.
- **Future work:** real sensors (for example a Bluetooth heart-rate strap), a reaction-time test (the engine already has `record_pvt`), cards reviewed by a flight surgeon, a searchable public-domain first-aid manual (FEMA CERT, US Army first aid) for questions the cards don't cover, delayed sync of logs to the ground team, more languages, and fine-tuning the model on these tools.

## Use of AI (Super Intelligence, SI)

- **Inside the product:**
  - Bento's replies come from Qwen3.5-9B running locally.
  - Speech is transcribed by faster-whisper (an AI speech recognition model).
  - **Bento's voice is AI-generated speech** from Kokoro-82M, using a stock voice. This is also stated in the demo video's description. The crew member's voice in the video is a real person (Ines).
- **In building it:**
  - The idea and its features (voice companion, emergency keywords, crew logs, check-ins, protocol cards) came from our team's own brainstorm.
  - We used AI coding assistants, including Claude (Anthropic), to plan how to build it and to write and debug code and tests. The voice loop, safety filters, dashboard redesign and test suites were written with AI help.
  - The protocol-card text was drafted with AI help from the published sources listed in each card.
  - This README was drafted with AI help.
  - All code was run and tested by the team on the demo PC.

## Tools and libraries

| Tool | Used for |
|---|---|
| [llama.cpp](https://github.com/ggml-org/llama.cpp) `llama-server` | Running the language model locally on the GPU |
| Qwen3.5-9B (Q4_K_M GGUF) | The language model |
| [faster-whisper](https://github.com/SYSTRAN/faster-whisper) with Whisper `small.en` | Speech to text |
| [Kokoro-82M](https://huggingface.co/hexgrad/Kokoro-82M), espeak-ng, PyTorch (CPU) | Text to speech |
| [Streamlit](https://streamlit.io), [Altair](https://altair-viz.github.io), pandas | Dashboard and charts |
| NumPy, sounddevice (PortAudio), soundfile | Audio handling |
| SQLite (Python `sqlite3`) | Shared mission database |
| pywebview (optional) | Desktop window for the dashboard |
| Google Fonts: Orbitron, JetBrains Mono | Dashboard fonts |
| uv | Python environments |

## Repository layout

```
engine/       simulator, monitoring engine, engine_api.py (the tools), protocols.json, contract check
voice/        voice loop, model client and prompt, safety filters, tool wiring, fake tools
dashboard/    Streamlit mission-control dashboard (app.py), optional desktop window (launch.py)
contract.md   the interface between the engine and the voice side
check_integration.py, check_live.py   test suites
```

## Team

- **Marco:** AI and voice: local model, speech, conversation logic, safety filters, live tests, dashboard.
- **Hannah:** deterministic engine, simulator, database and dashboard.
- **Ines:** Actress in the demo.
