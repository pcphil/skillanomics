# Lesson 1: GCP Bootstrap

## Concept

Deploying to Gemini Enterprise Agent Platform requires a real Google Cloud project — this lesson gets one into a deployable state. Four pieces, in order:

1. **Project** — a GCP project to hold everything (create a new one, or use an existing empty one dedicated to this course).
2. **Billing** — must be enabled on the project. Agent Runtime deployment and usage incur real charges; there is no free/sandbox tier for this.
3. **APIs** — the Agent Platform API (Vertex AI's agent-engine surface) and the Cloud Storage API must both be enabled (deployment stages agent code/dependencies through a GCS bucket).
4. **IAM roles** — the identity you'll deploy with needs at minimum `roles/aiplatform.user` and `roles/storage.admin`.

If the learner already has a configured project (per Assessment), skip straight to the verification checklist below instead of the full walkthrough — but still verify each item, since "I think it's set up" is not the same as confirmed.

## Task

**New/partial setup:**
1. Create or select a GCP project; note its project ID.
2. Enable billing on it (Cloud Console: Billing > Link a billing account).
3. Enable the Agent Platform API and Cloud Storage API for the project.
4. Grant the deploying identity `roles/aiplatform.user` and `roles/storage.admin`.
5. Run `gcloud auth application-default login` locally so the SDK can authenticate.

**Already-configured project (verification path):**
1. Confirm the project ID and that billing shows active.
2. Confirm both required APIs show enabled in the console or via `gcloud services list --enabled`.
3. Confirm the deploying identity actually has both IAM roles (not just "probably does").
4. Confirm `gcloud auth application-default login` has been run and is current.

## Acceptance Criteria

- A specific GCP project ID exists with billing confirmed active.
- Both required APIs are confirmed enabled (not assumed).
- Both required IAM roles are confirmed granted to the deploying identity.
- `gcloud auth application-default login` has been run successfully.

## Common Mistakes

- Enabling APIs but not billing (or vice versa) — deployment fails at different, confusing points depending on which is missing.
- Assuming a role is granted because "I'm the project owner" — owner implies broad access but it's worth explicitly confirming the specific roles for clarity on what deployment actually needs.
- Forgetting `gcloud auth application-default login` after switching accounts/projects — the SDK will authenticate as a stale identity.
