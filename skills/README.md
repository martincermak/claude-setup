# Claude Skills Collection

A collection of [Agent Skills](https://github.com/anthropics/skills) for Claude. This README explains what a skill is, how to build one, how to test it, and how to install the skills in this repo.

> Based on Anthropic's *The Complete Guide to Building Skills for Claude*.

---

## Table of contents

- [What is a skill?](#what-is-a-skill)
- [Repository layout](#repository-layout)
- [Installing skills from this repo](#installing-skills-from-this-repo)
- [Building a new skill](#building-a-new-skill)
  - [1. Start with use cases](#1-start-with-use-cases)
  - [2. Folder structure and naming rules](#2-folder-structure-and-naming-rules)
  - [3. YAML frontmatter](#3-yaml-frontmatter)
  - [4. Writing the instructions](#4-writing-the-instructions)
- [Common patterns](#common-patterns)
- [Testing and iteration](#testing-and-iteration)
- [Troubleshooting](#troubleshooting)
- [Distribution](#distribution)
- [Checklist](#checklist)
- [Resources](#resources)

---

## What is a skill?

A skill is a folder of instructions that teaches Claude how to handle a specific task or workflow. You explain your process once, and Claude applies it every time instead of you re-prompting in each conversation.

Skills fit repeatable work: generating documents in a house style, running research with a fixed methodology, orchestrating multi-step processes, or adding workflow knowledge on top of an MCP server.

### Design principles

**Progressive disclosure** - skills load in three levels to save context:

1. **Frontmatter** (`name`, `description`) - always in Claude's context; tells Claude *when* the skill applies.
2. **`SKILL.md` body** - loaded only when Claude decides the skill is relevant.
3. **Linked files** (`references/`, `scripts/`, `assets/`) - opened only when needed.

**Composability** - Claude may load several skills at once, so don't assume yours is the only one active.

**Portability** - the same skill works in Claude.ai, Claude Code, and the API, as long as the environment supports its dependencies.

### Skills + MCP

MCP gives Claude access to tools (the kitchen). Skills tell Claude how to use them well (the recipes). Without a skill, users connect an MCP server and then don't know what to do next; with one, workflows trigger automatically and results are consistent.

---

## Repository layout

```
.
├── README.md                  # This file (for humans, repo level)
├── skill-one/
│   ├── SKILL.md
│   ├── scripts/
│   ├── references/
│   └── assets/
├── skill-two/
│   └── SKILL.md
└── ...
```

Each top-level folder is one skill. The repo-level README is for people browsing GitHub; **do not** put a `README.md` inside individual skill folders.

---

## Installing skills from this repo

**Claude.ai**

1. Clone the repo (`git clone <repo-url>`) or download a ZIP from Releases.
2. Zip the skill folder you want (the folder itself, containing `SKILL.md`).
3. Go to **Settings → Capabilities → Skills** and click **Upload skill**.
4. Toggle the skill on. If it relies on an MCP server, make sure that connector is enabled.
5. Test it with a prompt that should trigger it.

**Claude Code**

Copy the skill folder into your Claude Code skills directory.

**API**

Use the `/v1/skills` endpoint and the `container.skills` parameter in Messages API requests (requires the Code Execution Tool beta). See the Anthropic docs linked below.

**Organizations**

Admins can deploy skills workspace-wide, with centralized management and automatic updates.

---

## Building a new skill

### 1. Start with use cases

Before writing anything, define 2-3 concrete use cases:

```
Use case: Sprint planning
Trigger:  "help me plan this sprint", "create sprint tasks"
Steps:    1. Fetch project status
          2. Analyze velocity and capacity
          3. Suggest prioritization
          4. Create tasks with labels and estimates
Result:   A fully planned sprint
```

Questions to answer: What does the user want to achieve? Which steps are involved? Which tools are needed (built-in or MCP)? What domain knowledge should be embedded?

**Typical skill categories**

| Category | Purpose | Examples |
|---|---|---|
| Document & asset creation | Consistent output: docs, decks, designs, code | `frontend-design`, `docx`, `pptx`, `xlsx` |
| Workflow automation | Multi-step processes with validation gates | `skill-creator` |
| MCP enhancement | Workflow guidance on top of MCP tools | Sentry code review |

**Success criteria** (rough targets, not hard thresholds)

- Triggers on ~90% of relevant queries (test with 10-20 prompts)
- Fewer tool calls and tokens than the no-skill baseline
- Zero failed API calls per workflow
- Users don't need to redirect Claude or explain next steps
- Consistent results across repeated runs and sessions

### 2. Folder structure and naming rules

```
your-skill-name/
├── SKILL.md          # Required
├── scripts/          # Optional: executable code (Python, Bash, ...)
├── references/       # Optional: docs loaded on demand
└── assets/           # Optional: templates, fonts, icons
```

| Rule | Correct | Wrong |
|---|---|---|
| File must be named exactly `SKILL.md` (case-sensitive) | `SKILL.md` | `skill.md`, `SKILL.MD` |
| Folder name in kebab-case | `notion-project-setup` | `Notion Project Setup`, `notion_project_setup`, `NotionProjectSetup` |
| No `README.md` inside the skill folder | docs in `SKILL.md` / `references/` | `README.md` in skill folder |

### 3. YAML frontmatter

The frontmatter decides whether Claude loads your skill, so get it right.

Minimal:

```yaml
---
name: your-skill-name
description: What it does. Use when user asks to [specific phrases].
---
```

**Fields**

| Field | Required | Notes |
|---|---|---|
| `name` | yes | kebab-case, should match folder name |
| `description` | yes | Must say **what** the skill does and **when** to use it; under 1024 characters; no `<` or `>` |
| `license` | no | e.g. `MIT`, `Apache-2.0` |
| `compatibility` | no | 1-500 chars: environment requirements (product, packages, network access) |
| `allowed-tools` | no | Restrict tool access, e.g. `"Bash(python:*) Bash(npm:*) WebFetch"` |
| `metadata` | no | Custom key-values: `author`, `version`, `mcp-server`, `category`, `tags`, ... |

**Security restrictions**

- No XML angle brackets (`<`, `>`) in frontmatter (it ends up in the system prompt).
- Skill names must not start with or contain the reserved words `claude` or `anthropic`.

**Writing a good description**

Structure: `[What it does] + [When to use it] + [Key capabilities]`

```yaml
# Good: specific, with trigger phrases
description: Manages Linear project workflows including sprint planning,
  task creation, and status tracking. Use when user mentions "sprint",
  "Linear tasks", "project planning", or asks to "create tickets".

# Bad: too vague
description: Helps with projects.

# Bad: no trigger conditions
description: Creates sophisticated multi-page documentation systems.

# Bad: technical, no user-facing triggers
description: Implements the Project entity model with hierarchical relationships.
```

### 4. Writing the instructions

Recommended `SKILL.md` body structure:

````markdown
---
name: your-skill
description: [...]
---

# Your Skill Name

## Instructions

### Step 1: [First major step]
Clear explanation of what happens.

```bash
python scripts/fetch_data.py --project-id PROJECT_ID
```
Expected output: [what success looks like]

## Examples

### Example 1: [common scenario]
User says: "Set up a new marketing campaign"
Actions:
1. Fetch existing campaigns via MCP
2. Create new campaign with provided parameters
Result: Campaign created with confirmation link

## Troubleshooting

**Error:** [common error message]
**Cause:** [why it happens]
**Solution:** [how to fix it]
````

**Best practices**

- **Be specific and actionable.** "Run `python scripts/validate.py --input {filename}`; if it fails, check for missing fields and date formats (use `YYYY-MM-DD`)" beats "validate the data".
- **Handle errors explicitly.** Describe common failures (e.g. MCP connection refused) and the exact recovery steps.
- **Reference bundled files clearly**, e.g. "consult `references/api-patterns.md` for rate limits, pagination, and error codes".
- **Use progressive disclosure.** Keep `SKILL.md` focused (under ~5,000 words) and push detail into `references/`.
- **Put critical instructions at the top**, under a clear header such as `## Critical`.
- **Prefer scripts for critical validation.** Code is deterministic; natural-language instructions are interpreted.

---

## Common patterns

Two ways to frame a skill: **problem-first** ("set up a project workspace" - the skill chooses the tool calls) or **tool-first** ("I have Notion MCP connected" - the skill teaches best practices for it).

| # | Pattern | Use when | Key techniques |
|---|---|---|---|
| 1 | Sequential workflow orchestration | Multi-step process in a fixed order | Explicit ordering, step dependencies, validation per stage, rollback instructions |
| 2 | Multi-MCP coordination | Workflow spans several services (e.g. Figma → Drive → Linear → Slack) | Clear phases, data passing between MCPs, validation before next phase, central error handling |
| 3 | Iterative refinement | Output quality improves with loops | Explicit quality criteria, validation scripts, defined stop condition |
| 4 | Context-aware tool selection | Same goal, different tools by context | Decision tree, fallbacks, explain the choice to the user |
| 5 | Domain-specific intelligence | Skill adds expert knowledge beyond tool access | Compliance checks before action, audit trail, clear governance |

Example skeleton for pattern 1:

```markdown
# Workflow: Onboard New Customer

## Step 1: Create Account
Call MCP tool: `create_customer` (name, email, company)

## Step 2: Set Up Payment
Call MCP tool: `setup_payment_method`
Wait for: payment method verification

## Step 3: Create Subscription
Call MCP tool: `create_subscription` (plan_id, customer_id from Step 1)

## Step 4: Send Welcome Email
Call MCP tool: `send_email` with template `welcome_email_template`
```

---

## Testing and iteration

Pick the rigor that matches how widely the skill will be used:

- **Manual** in Claude.ai - fastest, no setup.
- **Scripted** in Claude Code - repeatable test cases.
- **Programmatic** via the Skills API - evaluation suites.

> **Tip:** Iterate on one hard task until Claude succeeds, then extract the winning approach into a skill. Expand to broader test cases afterwards.

**Three kinds of tests**

1. **Triggering** - loads on obvious and paraphrased requests, stays silent on unrelated ones.
2. **Functional** - correct outputs, successful API calls, error handling, edge cases.
3. **Performance comparison** - same task with and without the skill; compare messages, failed calls, and tokens.

**Using `skill-creator`**

The `skill-creator` skill can draft a `SKILL.md`, suggest triggers, review your skill for vague descriptions and over/under-triggering risk, and help you improve it from failure examples:

> "Use the skill-creator skill to help me build a skill for [your use case]."

It helps design and review skills; it does not run automated test suites.

**Iteration signals**

| Signal | Symptoms | Fix |
|---|---|---|
| Under-triggering | Skill doesn't load; users enable it manually | Add detail and keywords to the description |
| Over-triggering | Loads for unrelated queries; users disable it | Add negative triggers, narrow the scope |
| Execution issues | Inconsistent results, API failures, user corrections | Improve instructions, add error handling |

---

## Troubleshooting

**Skill won't upload**

- *"Could not find SKILL.md"* - the file isn't named exactly `SKILL.md`. Check with `ls -la`.
- *"Invalid frontmatter"* - missing `---` delimiters or unclosed quotes.
- *"Invalid skill name"* - name contains spaces or capitals; use `my-cool-skill`.

**Skill doesn't trigger**

The description is probably too generic or lacks phrases users actually say. Debug by asking Claude: *"When would you use the [skill name] skill?"* - it will quote the description back, and you can see what's missing.

**Skill triggers too often**

Add negative triggers (`Do NOT use for simple data exploration`), be more specific (`PDF legal documents for contract review` instead of `documents`), and state the scope explicitly.

**MCP calls fail**

1. Confirm the MCP server is connected (Settings → Extensions).
2. Check API keys, scopes, and token refresh.
3. Test the MCP without the skill. If that fails too, the problem is the MCP, not the skill.
4. Verify tool names (they are case-sensitive).

**Instructions not followed**

- Too verbose → shorten, use lists, move detail to `references/`.
- Critical rules buried → move them to the top, add a `## Critical` header.
- Ambiguous wording → replace "validate things properly" with a concrete checklist, or bundle a validation script.

**Slow or degraded responses**

Keep `SKILL.md` small, link to `references/`, and avoid enabling too many skills at once (consider whether you have more than 20-50 active).

---

## Distribution

Recommended approach:

1. **Host on GitHub** (public repo for open-source skills) with a clear README, installation steps, and example usage with screenshots.
2. **Link from your MCP docs** if you have an MCP server - explain why the skill and MCP are better together and add a quick-start.
3. **Write an installation guide** (see [Installing skills from this repo](#installing-skills-from-this-repo)).

Skills are published as an open standard (Agent Skills), so they can be portable across tools. If a skill depends on a specific platform, note it in the `compatibility` field.

**Positioning tip:** describe outcomes, not internals.

- Good: "Set up a complete project workspace in seconds instead of 30 minutes of manual setup."
- Bad: "A folder with YAML frontmatter and Markdown that calls our MCP tools."

---

## Checklist

**Before you start**

- [ ] 2-3 concrete use cases identified
- [ ] Tools identified (built-in or MCP)
- [ ] Folder structure planned

**During development**

- [ ] Folder is kebab-case
- [ ] `SKILL.md` exists with exact spelling
- [ ] Frontmatter has `---` delimiters
- [ ] `name` is kebab-case, no spaces or capitals
- [ ] `description` states WHAT and WHEN
- [ ] No `<` or `>` anywhere in frontmatter
- [ ] Instructions are clear and actionable
- [ ] Error handling and examples included
- [ ] References clearly linked

**Before upload**

- [ ] Triggers on obvious and paraphrased requests
- [ ] Does not trigger on unrelated topics
- [ ] Functional tests pass
- [ ] Tool integration works (if applicable)
- [ ] Skill folder zipped

**After upload**

- [ ] Tested in real conversations
- [ ] Monitored for under/over-triggering
- [ ] Feedback collected
- [ ] Description, instructions, and `metadata.version` updated

---

## Resources

- Public skills repository: [anthropics/skills](https://github.com/anthropics/skills) (document skills, example skills, partner skills)
- Original guide: [The Complete Guide to Building Skills for Claude](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf)
- Anthropic docs: Skills documentation, API reference, MCP documentation, Best Practices guide
- Questions: Claude Developers Discord
- Bugs: [anthropics/skills issues](https://github.com/anthropics/skills/issues) - include skill name, error message, and reproduction steps

---

## License

Add your license here (e.g. MIT).
