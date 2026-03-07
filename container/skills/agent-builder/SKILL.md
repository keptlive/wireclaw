---
name: agent-builder
description: Create a new WireClaw agent with YAML manifest and system prompt. Use when asked to build, create, add, or set up a new agent, bot, assistant, or persona.
---

# Agent Builder

Create fully configured WireClaw agents through a research-driven workflow. This skill is for admin agents with access to the NanoClaw project directory.

## Step 1: Ask the User

Ask only two questions:

1. **Handle** — lowercase alphanumeric with hyphens/underscores. Regex: `^[a-z0-9][a-z0-9_-]{0,63}$`. Becomes their email: `{handle}@agentwire.email`.
2. **Purpose** — what this agent should do. A sentence or paragraph describing its role.

Optionally ask:
- **Dependency limit** — max number of MCP servers / system packages (default: no limit)

Do NOT ask about model, MCP servers, skills, env vars, or claude.md content — research will determine those.

## Step 2: Imagine the Workflow

Based on the purpose, think through what this agent's workflow looks like:

- What tasks will it perform daily?
- What external services or data does it need?
- Does it need a browser? Scheduled tasks? File access?
- What personality and communication style fits?

Present a short workflow description (3-5 bullets) and get confirmation before proceeding.

## Step 3: Research

Research three areas. If you have subagent capabilities, run these in parallel. Otherwise, work through them sequentially.

### A: Skills Research

Check `/workspace/project/container/skills/` for available container skills. Read each SKILL.md. Determine:
- Which skills are relevant to the purpose
- Whether `agent-browser` is needed
- Whether a new custom skill should be created

### B: MCP Servers & Dependencies

The agent automatically gets these (NEVER add them to yaml):
- **nanoclaw**: send_message, schedule_task, list_tasks, update_task, pause_task, resume_task, cancel_task, register_group
- **agentwire**: send_email, list_emails, read_email, read_email_html, get_attachment, invite_contact, list_contacts, update_contact, unblock_contact, list_sms, read_sms, post_message, set_messages_password, add_sms_sender, remove_sms_sender, list_sms_senders, get_agent_notes, set_agent_notes, get_usage, deploy_agent_spa, search_memory, remember, forget, get_recent_context, memory_stats

Research what ADDITIONAL MCP servers would help. For each:
1. Name and npm package (or command)
2. What it provides
3. Auth requirements (env vars, API keys)
4. Check if credentials already exist in the environment

Also determine:
- System packages needed (ffmpeg, imagemagick, etc.)
- Environment variables needed
- Recommended model (default: `claude-sonnet-4-6`)
- Recommended container timeout (default: 300000ms)

YAML format for mcp_servers:
```yaml
# Shorthand
github: "npx -y @mcp/server-github"
# Full object with env
custom:
  command: "node"
  args: ["/path/to/server.js"]
  env:
    API_KEY: "$MY_API_KEY"
# SSE (remote)
remote:
  type: "sse"
  url: "http://localhost:3000/mcp"
  headers:
    Authorization: "Bearer $TOKEN"
```

### C: claude.md Best Practices

Read existing agent claude.md files in `/workspace/project/groups/` for patterns. Note:
- Section structure and ordering
- Communication guidelines (`<internal>` tags, formatting)
- Memory patterns (workspace files, conversations folder)
- Message formatting (WhatsApp/Telegram: *bold*, _italic_, bullets, no markdown headings)
- Tool documentation approach
- Keep under 300 lines

## Step 4: Review & Gap Analysis

After research completes:

1. **Merge** — combine skills, MCP servers, and claude.md recommendations
2. **Apply dependency limit** — if set, rank by importance and trim
3. **Check gaps**:
   - Missing tools for the workflow?
   - Unresolved auth requirements?
   - Missing skills?
   - Does the claude.md plan cover the workflow?
4. **Present plan** to user:
   - Agent name, handle, model
   - Skills list
   - MCP servers (with auth status: available / needs-user-input / skipped)
   - claude.md outline
   - Env vars needed

Wait for user approval.

## Step 5: Handle Missing Auth

For any MCP server needing credentials the user hasn't provided:
- Ask: "The {server} MCP server needs {credential}. (a) provide now, (b) skip, (c) add later?"
- If skipped, comment it out in YAML with a note

## Step 6: Generate wireclaw.yaml

Write to `/workspace/project/groups/{handle}/wireclaw.yaml`:

```yaml
version: "1.0"

identity:
  group_name: "{Agent Display Name}"
  handle: "{handle}"
  description: "{One-line description}"

context:
  system_prompt: "./claude.md"
  model: "{model}"  # omit for default

dependencies:
  system_packages:  # omit if empty
    - package1
  env_vars:
    - AGENTWIRE_API_KEY  # always include
  mcp_servers:  # omit if none beyond auto-provided
    server-name: "npx -y @package/name"

container:
  timeout: 300000

skills:
  - agent-browser  # if needed
```

Notes:
- Omit `channel_binding` for AgentWire-only agents (auto-assigned `aw:{handle}` JID)
- `context.system_prompt` is relative to manifest dir
- `AGENTWIRE_API_KEY` is always required

## Step 7: Generate claude.md

Write to `/workspace/project/groups/{handle}/claude.md` AFTER all research is done.

Structure:

```markdown
# {Agent Name}

You are {Agent Name}, {role description}. {Personality sentence}.

## What You Can Do

- {Capability with tool reference}
- Browse the web with `agent-browser` (if included)
- Schedule tasks to run later or on a recurring basis
- Send messages back to the chat

## Communication

Your output is sent to the user or group.

Use `mcp__nanoclaw__send_message` for quick acknowledgments before long tasks.

### Internal thoughts

Wrap reasoning in `<internal>` tags — logged but not sent.

### Message Formatting

WhatsApp/Telegram only:
- *Bold* (single asterisks, NEVER **double**)
- _Italic_ (underscores)
- • Bullet points
- ```Code blocks```
- No ## headings, no [links](url)

## {Domain-Specific Section}

{Concrete instructions for the agent's actual work — workflows, rules, examples.
This is the most important section. Make it specific, not generic.}

## Tools Reference

### NanoClaw (always available)
- `mcp__nanoclaw__send_message` — send message while still running
- `mcp__nanoclaw__schedule_task` — create cron/interval/one-time tasks
- `mcp__nanoclaw__list_tasks` — view scheduled tasks
- `mcp__nanoclaw__update_task` / `pause_task` / `resume_task` / `cancel_task`

### AgentWire (always available)
- `mcp__agentwire__send_email` — send from {handle}@agentwire.email
- `mcp__agentwire__list_emails` / `read_email` / `read_email_html` — inbox
- `mcp__agentwire__get_attachment` — download attachment
- `mcp__agentwire__list_contacts` / `update_contact` / `invite_contact`
- `mcp__agentwire__post_message` — talk page
- `mcp__agentwire__remember` / `search_memory` / `forget` / `get_recent_context`
- `mcp__agentwire__get_agent_notes` / `set_agent_notes` — scratchpad
- `mcp__agentwire__get_usage` — usage stats
- `mcp__agentwire__deploy_agent_spa` — deploy web app

### {Custom MCP Server} (if any)
- `mcp__{name}__{tool}` — {description}

## Memory

Workspace: `/workspace/group/` — files persist across sessions.
History: `conversations/` folder for past context.

When you learn something important:
- Create structured files (e.g., `notes.md`, `preferences.md`)
- Split files >500 lines into folders
- Keep an index
```

## Step 8: Apply

Run the manifest CLI:

```bash
cd /workspace/project && npx tsx src/manifest-cli.ts apply groups/{handle}/wireclaw.yaml
```

Expected output:
- `[+] {handle} → aw:{handle} (created)` — success
- `Handle "{handle}" is already taken` — pick a different handle
- Parse errors — fix the YAML

## Step 9: Verify

1. Check apply output shows "created"
2. Agent email: `{handle}@agentwire.email`
3. Intro email sent (check logs)
4. Optionally test with a message

Report to user: agent name, handle, email, skills, MCP servers, any skipped deps.

## Reference

### Valid Models
| Model | ID | Best For |
|-------|-----|----------|
| Sonnet 4.6 | `claude-sonnet-4-6` | Default, fast |
| Opus 4.6 | `claude-opus-4-6` | Complex reasoning |
| Haiku 4.5 | `claude-haiku-4-5-20251001` | Simple, high volume |

### Handle Rules
- Regex: `^[a-z0-9][a-z0-9_-]{0,63}$`
- Lowercase only, hyphens/underscores OK
- Max 64 chars, cannot be "global"
- Must be unique on AgentWire

### YAML Schema (all fields)
```yaml
version: "1.0"                    # required
identity:                         # required
  group_name: ""                  # required
  handle: ""                      # required, validated
  description: ""                 # optional
context:                          # optional
  system_prompt: "./claude.md"    # optional, relative path
  model: ""                       # optional, claude model ID
channel_binding:                  # optional (omit for AgentWire-only)
  jid: ""                        # chat JID
  trigger: "@handle"             # trigger word
  requires_trigger: true         # default true
dependencies:                     # optional
  system_packages: []            # apt packages
  env_vars: []                   # env var names
  mcp_servers: {}                # name → spec
container:                        # optional
  timeout: 300000                # ms
  additional_mounts:             # host dir access
    - host_path: "~/path"
      container_path: "name"
      readonly: true
skills: []                        # container skill names
```
