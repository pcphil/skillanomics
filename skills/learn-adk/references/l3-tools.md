# Lesson 3: Tools

## Concept

Tools are how an ADK agent does things beyond generating text. The simplest form is a **function tool**: a plain Python function with a docstring and type-hinted parameters — ADK inspects the signature and docstring to build the tool schema the model sees, so accurate types and a clear docstring are not optional polish, they're what the model uses to decide when and how to call it.

For this course, the knowledge tool is deliberately simple: a small in-repo sample FAQ set (a Python dict or list of `{question, answer}` entries) plus a lookup function over it. This is **not** a RAG/embeddings pipeline — no vector store, no similarity search. The point of this lesson is ADK's tool-calling mechanics, not retrieval engineering. (A real production support agent might use RAG over a large knowledge base — that's a one-line extension note for the capstone, not something to build here.)

## Task

1. Create a small sample FAQ set in the project, e.g.:
   ```python
   FAQS = {
       "reset password": "Go to Settings > Security > Reset Password.",
       "refund policy": "Refunds are available within 30 days of purchase.",
       "contact support": "Email support@example.com or use the in-app chat.",
   }
   ```
2. Write a function tool that looks up the closest matching FAQ entry:
   ```python
   def lookup_faq(topic: str) -> str:
       """Look up a support FAQ answer by topic keyword.

       Args:
           topic: A keyword or short phrase describing what the user is asking about.

       Returns:
           The matching FAQ answer, or a message saying nothing was found.
       """
       for key, answer in FAQS.items():
           if key in topic.lower():
               return answer
       return "No matching FAQ found."
   ```
3. Add it to the agent: `tools=[lookup_faq]`.
4. Update the `instruction` to tell the agent to use the tool for FAQ-type questions instead of answering from its own knowledge.
5. Run `adk run` and ask a question that should trigger the tool (e.g. "how do I reset my password?"). Confirm from the run output that the tool was actually called, not just that the answer sounds right.

## Acceptance Criteria

- `lookup_faq` (or equivalent) is a real Python function with type hints and a docstring the model can use to decide when to call it.
- The agent's `tools` list includes it, and the instruction directs the agent to prefer it for FAQ questions.
- A live run shows the tool actually being invoked (via `adk run`'s trace/output, not just the final answer) for an FAQ-relevant question.

## Common Mistakes

- Skipping the docstring or type hints — the model can't reliably decide when/how to call an under-specified tool.
- Building a mini keyword-matching engine that starts to look like retrieval — keep it to a simple, obviously-not-RAG lookup for this lesson.
- Confirming success from the final answer text alone without checking that a tool call actually happened — the model can sometimes answer from its own knowledge and skip the tool entirely, which is a real bug to catch here.
