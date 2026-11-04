# Lesson 2: AdkApp Wrapping

## Concept

An ADK agent built for local `adk run`/`adk web` use isn't directly what gets deployed — it's wrapped in an `AdkApp` object, which is the interface Vertex AI Agent Engine actually hosts and calls. Wrapping it locally first and testing it via a local query lets you catch integration problems before spending anything on a real deployment.

## Task

1. Install the deployment-side SDK: `pip install --upgrade "google-cloud-aiplatform[agent_engines,adk]"` (verify current package extras/version against official docs — this has been observed to change).
2. Import your agent (the `learn-adk` capstone's `root_agent`, or the scaffolded starter agent if working standalone) and wrap it:
   ```python
   from vertexai.preview.reasoning_engines import AdkApp

   app = AdkApp(agent=root_agent)
   ```
3. Initialize the Vertex AI SDK with your project ID and a region:
   ```python
   import vertexai
   vertexai.init(project="<your-project-id>", location="<region>")
   ```
4. Test locally before deploying anything: call the app's async streaming query method with a sample message and confirm you get a real response, without touching Agent Engine yet.

## Acceptance Criteria

- The agent is wrapped in `AdkApp` and initializes without error.
- A local async query against the wrapped app returns a real response — this is confirmed *before* any deploy action, since it isolates wrapping problems from deployment problems.

## Common Mistakes

- Skipping local testing of the wrapped app and going straight to deploy — if something's wrong with the wrapping, you'll pay to discover that instead of finding it for free locally.
- Using a project ID or region that doesn't match where the required APIs were enabled in Lesson 1.
- Installing the deploy-side SDK without checking current extras/version, then hitting import errors that look unrelated to the actual cause.
