# IT Support AI Assistant

Reads new Jira Service Management tickets, finds the relevant articles in the Confluence knowledge base, and posts an AI-written internal note with suggested troubleshooting steps for the service desk team.

<img width="1845" height="668" alt="image" src="https://github.com/user-attachments/assets/b2f0912b-66e2-4fa2-b3f2-887cf2b7f04a" />


---

## What It Does

Every 5 minutes, this workflow checks Jira Service Management for open tickets it hasn't handled yet. For each ticket, Claude picks a few search keywords and the workflow searches the Confluence knowledge base with them. Claude then reads the ticket together with the articles it found and suggests troubleshooting steps taken only from those articles. The suggestions are posted on the ticket as an **internal note**, which the customer never sees, and the ticket is labelled `ai-triaged` so it is never processed twice. An email tells the service desk that a new note is ready.

The AI only advises. It never changes the ticket's priority, status or assignee, never replies to the customer, and never performs any IT action. A person on the service desk always decides what happens next.

<img width="1229" height="412" alt="image" src="https://github.com/user-attachments/assets/7f5cb847-1a74-49fb-b056-216f1abf7dd2" />

---

## Flow

```
Schedule Trigger (every 5 minutes)
         ↓
  Get JIRA Tickets          ← open tickets without the ai-triaged label
         ↓
  Find Search Words         ← Claude Haiku picks 1–3 keywords per ticket
         ↓
  Search KB                 ← Confluence search, top 3 matching articles
         ↓
  Analyze JIRA Ticket       ← Claude Sonnet reads ticket + articles
         ↓
  Build Comment             ← format the note for the service desk
         ↓
  Post Internal Comment     ← internal note on the ticket (public: false)
         ↓
  Label Marked              ← add the ai-triaged label
         ↓
  Send a message            ← email notification
```

---

## Nodes

- **Schedule Trigger**: runs every 5 minutes
- **Get JIRA Tickets**: Jira node, JQL `project = ITS AND statusCategory != Done AND (labels IS EMPTY OR labels NOT IN (ai-triaged))`. The `labels IS EMPTY` part is needed because Jira's `NOT IN` skips tickets that have no labels at all
- **Find Search Words**: Basic LLM Chain with Claude Haiku 4.5; returns 1–3 comma-separated keywords for the ticket
- **Search KB**: HTTP Request to the Confluence search API; builds a CQL query from the keywords (`text ~ "vpn" OR text ~ "network"`) and returns at most 3 articles with their content
- **Analyze JIRA Ticket**: Basic LLM Chain with Claude Sonnet 5 and a Structured Output Parser; returns `kb_articles_used`, `troubleshooting_steps` and a `confidence` score. It uses only the articles that match the ticket and only steps from those articles, never general knowledge
- **Build Comment**: Edit Fields node that turns the AI output into a note in Jira formatting (bold headings, numbered steps)
- **Post Internal Comment**: HTTP Request to the Jira Service Management API with `public: false`, so the note is visible to helpdesk only. The standard Jira node can only add public comments, which JSM emails to the customer
- **Label Marked**: HTTP Request that adds the `ai-triaged` label without removing existing labels; this is how the workflow remembers which tickets are done
- **Send a message**: Gmail node that emails the service desk with the ticket, the article used, the suggested steps and a link to the ticket

  <img width="1248" height="625" alt="image" src="https://github.com/user-attachments/assets/8e67bcf4-67b9-432c-a322-043388060431" />


---

## Credentials

All credentials are stored in the n8n credential store; no keys are in the workflow JSON.

| Service                      | Used By                                                               |
|------------------------------|-----------------------------------------------------------------------|
| Jira (Atlassian API token)   | Get JIRA Tickets, Search KB, Post Internal Comment, Label Marked      |
| Anthropic                    | Anthropic Chat Model, Anthropic Chat Model1, Anthropic Chat Model2    |
| Gmail                        | Send a message                                                        |

The same Atlassian API token works for Jira and Confluence.

---

## Design Choices

- **Search instead of sending the whole knowledge base.** Sending every article with every ticket gets expensive as the knowledge base grows. A small, cheap model first picks keywords, and only the top 3 matching articles go to the main model.
- **Knowledge base only.** Suggested steps come from the company's own articles, not from the model's general knowledge, so advice follows internal procedures.
- **Internal notes, human in control.** The note is written for the service desk team and is never visible to the customer. Priority, category and the reply to the customer stay with helpdesk.
- **Idempotent.** The `ai-triaged` label makes sure each ticket is processed exactly once, even though the workflow runs every 5 minutes.

---

## Why This Was Built

First-line IT support spends much of its time reading tickets, looking up the matching knowledge base article, and working out what to ask the user next. This workflow does that groundwork as soon as a ticket arrives: by the time an agent opens it, the ticket already has the relevant article and a clear list of troubleshooting steps. The service desk team keeps full control and simply starts from a better position.
