# Auto Student Support Reply Agent

An n8n workflow that helps a course's teaching team answer routine student emails automatically, while keeping a human in the loop whenever the AI isn't confident.

## The problem

As a TA for a large class, the teaching team gets flooded with emails that ask the same handful of questions over and over — most of which are already answered in the syllabus or course materials. Answering each one by hand doesn't scale, but a fully automated "AI answers everything" bot is risky: some questions are genuinely novel, ambiguous, or sensitive, and shouldn't get a confident-sounding answer from a model that's guessing.

## The solution

An email-triggered agent that:

1. Reads an incoming student email.
2. Classifies the question and drafts an answer using the syllabus/course materials as grounding, along with a **confidence score** for whether that answer is actually supported by the course content.
3. Branches on confidence:
   - **High confidence** → replies directly to the student with the sourced answer.
   - **Low confidence** → replies to the student but **CCs the teaching team lead**, so a human can step in if the AI's answer isn't good enough.
4. **Every** email, regardless of confidence, gets logged as a row in a shared backlog spreadsheet, so the teaching team has a running record of what's being asked and how it was handled.

This keeps the teaching team out of the loop for the easy 80% of questions, while making sure nothing sensitive or ambiguous gets a silent, unreviewed answer — and nothing falls through the cracks of the backlog.

## Architecture

```
Gmail Trigger (new email)
        │
        ▼
  Set First Name              (pulls the student's first name for a personalized reply)
        │
        ▼
  Category Agent              (LLM agent: classifies + drafts answer + confidence score)
   ├─ Chat Model: GPT-5 mini
   ├─ Memory: short-term conversation buffer
   ├─ Tool: course material / syllabus lookup
   └─ Output Parser: structured "agent decision" schema
        │
        ▼
  If confident in response?
   ├─ true  ──▶ Reply to message (w/ CC to teaching team) ──┐
   └─ false ──▶ Reply to message (no CC)                    ├──▶ Append row in backlog sheet
```

Wait — note the branch is intentionally the opposite of "obvious": **low-confidence replies get the CC**, so a human is looped in exactly when the AI is least sure of itself. High-confidence replies go straight to the student with no CC, since the answer is already well-grounded in course material.

See [`workflow/course-qa-agent.n8n.json`](workflow/course-qa-agent.n8n.json) for the importable n8n workflow, and [`docs/category-agent-prompt.md`](docs/category-agent-prompt.md) for the system prompt and confidence-scoring rubric driving the agent.

## Components

| Node | Purpose |
|---|---|
| **When receiving an email** (Gmail Trigger) | Watches the shared course inbox for new student emails. |
| **Set First Name** | Extracts the student's first name from the email for a friendlier reply. |
| **Category Agent** | LangChain agent (GPT-5 mini) that reads the question, searches course material, drafts a reply, and outputs a structured decision: `{ answer, category, confident, reasoning }`. |
| **If confident in response...** | Routes on the `confident` boolean from the agent's structured output. |
| **Reply to a message (w/ CC)** | Sent when the agent is *not* confident — replies to the thread and CCs the teaching-team lead for review. |
| **Reply to a message (no CC)** | Sent when the agent *is* confident — replies to the thread directly. |
| **Append row in sheet** | Logs every ticket (question, category, confidence, answer, whether it was escalated) to a Google Sheet backlog for the teaching team to track support volume and trends. |

## Setup

1. Import [`workflow/course-qa-agent.n8n.json`](workflow/course-qa-agent.n8n.json) into your n8n instance.
2. Configure credentials:
   - Gmail OAuth2 (shared course inbox)
   - OpenAI API (GPT-5 mini)
   - Google Sheets OAuth2 (backlog spreadsheet)
3. Point the course-material tool at your syllabus/course content (vector store, Google Drive folder, or Notion — see `docs/category-agent-prompt.md` for how the tool is expected to respond).
4. Set the teaching-team-lead CC address in the "Reply to a message (w/CC)" node.
5. Create a backlog Google Sheet with columns: `timestamp, from, subject, category, confident, answer, escalated`.
6. Activate the workflow.

## Status

Early stage — first working version of the routing logic. Planned next steps are tracked as issues.
