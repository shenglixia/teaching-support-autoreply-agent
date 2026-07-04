# Auto Student Support Reply Agent

An n8n workflow ("Kai Support") that helps the AIML901 teaching team at Kellogg answer routine student emails automatically, while keeping a human in the loop whenever the AI isn't confident, a request is high priority, or it doesn't fit a known category.

## The problem

As a TA for a large class, the teaching team gets flooded with emails that ask the same handful of questions over and over. Answering each one by hand doesn't scale, but a fully automated "AI answers everything" bot is risky: some questions are personal, urgent, or genuinely novel, and shouldn't get a confident-sounding answer from a model that's guessing.

## The solution

An email-triggered agent that:

1. Reads an incoming student email and pulls the sender's first name.
2. An LLM agent ("Kai Support", GPT-5 mini) drafts a reply and classifies the ticket into a structured decision: category, priority, a CC target, and a `confidence` flag (true only if it can fully answer every question asked).
3. Routes on **need for a human**, not just raw confidence — a ticket gets escalated (CC'd) if *any* of these are true:
   - `confidence == false` (the agent doesn't have enough information), **or**
   - `ticket_priority == "high"`, **or**
   - `ticket_category == "other"` (doesn't fit a known category)
4. **Every** ticket, escalated or not, gets appended to a shared backlog spreadsheet in parallel with the reply — so the teaching team has a running record of what's being asked, by whom, and how it was routed.

This keeps the teaching team out of the loop for routine questions the agent can answer confidently, while making sure anything urgent, ambiguous, or under-confident gets a human CC'd — and nothing falls through the cracks of the backlog.

## Architecture

```
Gmail Trigger (new email)
        │
        ▼
  Set First Name              (pulls the student's first name for a personalized reply)
        │
        ▼
  Category Agent              (LLM agent "Kai Support": drafts reply + structured decision)
   ├─ Chat Model: GPT-5 mini
   └─ Output Parser: structured "agent decision" schema
        │
        ▼
  If confident in response...  (true = needs escalation: high priority OR category=other OR confidence=false)
   ├─ true  ──┬─▶ Reply to a message (w/ CC to the relevant teaching-team member)
   │          └─▶ Append row in sheet
   └─ false ──┬─▶ Reply to a message (no CC, sent directly)
              └─▶ Append row in sheet
```

See [`workflow/auto-reply-agent.n8n.json`](workflow/auto-reply-agent.n8n.json) for the importable n8n workflow, and [`docs/category-agent-prompt.md`](docs/category-agent-prompt.md) for the full system prompt, structured-output schema, and escalation logic.

## Components

| Node | Purpose |
|---|---|
| **When receiving an email** (Gmail Trigger) | Watches the shared course inbox for new student emails. |
| **Set First Name** | Extracts the student's first name from the email for a friendlier reply. |
| **Category Agent** | LangChain agent (GPT-5 mini) that reads the question, drafts a reply as "Kai Support," and outputs a structured decision (`response_content`, `response_cc`, `ticket_category`, `ticket_priority`, `confidence`, etc.). |
| **If confident in response...** | Despite the name, this checks whether the ticket *needs a human*: true if priority is high, category is "other", or confidence is false. |
| **Reply to a message (w/ CC)** | Sent on the escalation branch — replies to the thread and CCs whichever teaching-team member the agent picked (instructor, TA, or class moderator, depending on the question). |
| **Reply to a message (no CC)** | Sent when the ticket is routine and the agent is confident — replies to the thread directly, no human involved. |
| **Append row in sheet** | Logs every ticket (`ticket_name`, `ticket_cc`, `ticket_category`, `ticket_description`) to a Google Sheet backlog, in parallel with the reply, so the teaching team can track support volume and trends. |

## Known gap

The system prompt instructs the agent to "use the AIML-901 Docs tool" to look up course material, but no such tool is currently wired into the workflow — only the Chat Model and Output Parser are connected to the Category Agent. Today the agent answers from the system prompt and its own general knowledge, not from a grounded syllabus/course-content lookup. Wiring up a real docs tool (vector store over the syllabus, or a simple document search) is the highest-value next step, since it would let low-priority "content"/"administrative" questions be answered with actual citations instead of the model's best guess.

## Setup

1. Import [`workflow/auto-reply-agent.n8n.json`](workflow/auto-reply-agent.n8n.json) into your n8n instance.
2. Configure credentials (the file ships with placeholders — replace `REPLACE_WITH_*` fields):
   - Gmail OAuth2 (shared course inbox)
   - OpenAI API (GPT-5 mini)
   - Google Sheets OAuth2 (backlog spreadsheet)
3. Update the teaching-team names/emails in the Category Agent's system message to match your actual course staff.
4. Create a backlog Google Sheet with columns: `ticket name, ticket cc, ticket category, ticket description`.
5. Activate the workflow.

## Status

Working first version, currently in use for AIML901. Next step: wire up a real course-material lookup tool so answers are grounded in the syllabus instead of the model's own knowledge.
