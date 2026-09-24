# Lesson 1: Setup

## Concept

Google's Agent Development Kit (ADK) is a Python SDK for building LLM-powered agents: you define an `Agent` with a model, an instruction, and a list of tools, and ADK handles the orchestration loop (call the model, run any tool calls it requests, feed results back, repeat until a final answer).

Core pieces at a glance:
- `pip install google-adk` — a regular Python package.
- A project is a Python package/folder containing an `agent.py` that exposes a `root_agent`.
- `adk run <agent_folder>` — interactive terminal chat with the agent, good for quick iteration.
- `adk web` — starts a local browser chat UI (explicitly a dev/debug tool, not for production use).

**Before installing:** package names stay stable but exact versions and minor CLI details drift. Have the learner check `pip index versions google-adk` (or the current docs at adk.dev) rather than trusting a hardcoded version number here.

## Task

1. Create a project directory for this course, e.g. `adk-course/`.
2. Verify the current `google-adk` version/install command against official docs or `pip index versions google-adk`, then install it (recommend a virtualenv if the learner hasn't set one up).
3. Create a minimal agent folder with an `agent.py`:
   ```python
   from google.adk.agents import Agent

   root_agent = Agent(
       name="hello_agent",
       model="gemini-flash-latest",
       description="A minimal placeholder agent.",
       instruction="You are a friendly assistant. Answer briefly.",
   )
   ```
4. Run `adk run <agent_folder>` and confirm you get a real model response to a simple message (e.g. "hi").
5. Say "done" once you've seen a real response.

## Acceptance Criteria

- `google-adk` installs cleanly and the version was verified against current docs, not assumed.
- `adk run` starts without import/config errors and produces an actual model response.

## Common Mistakes

- Running `adk run`/`adk web` from the wrong directory — ADK looks for the agent folder relative to the cwd (or an explicit path argument).
- Missing model-provider credentials (e.g. no API key / no ADC set up) — the agent will fail on the first real call, not at import time.
- Treating `adk web` as safe to expose beyond localhost — it's explicitly a dev/debug UI, not production-hardened.
