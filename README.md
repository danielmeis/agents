# agents

A portable VS Code Copilot configuration — shared skills, reusable prompts, and always-on
base instructions. Shared content is version-controlled and syncs via `git pull`.
Machine-specific content lives in `local/` (gitignored).

## Assumptions
- You have VS Code installed with the Copilot Chat extension
- Your root workspace directory is `~/Websites/agents` (or adjust paths to match your clone location)
- Prompts such as `ticket-writing` assume you use Atlassian Jira for ticketing and GitHub for code hosting. Adjust as needed for your workflow.
- There are no guarantees that these skills/prompts are the best way for you to work. You may use them as a starting point and adapt as needed.
- **Always fully read a skill or prompt before using it — do not assume it is correct or complete**.

**Upon completion reloading VSCode is needed to apply the updated skills/prompts**
---

## Structure

```
agents/
├── global/
│   └── copilot-instructions.md        # Always-on base rules, injected into every session
├── prompts/                           # Shared prompts — synced across machines via git
│   ├── code-review.prompt.md          # /code-review
│   ├── pull-request.prompt.md         # /pull-request
│   ├── release-notes.prompt.md        # /release-notes
│   └── ticket-writing.prompt.md       # /ticket-writing
├── skills/                            # Shared skills — synced across machines via git
│   ├── _template/
│   │   └── SKILL-template.md          # Starter template for new skills
│   ├── mysql-best-practices/
│   │   └── SKILL.md
│   ├── react/
│   │   └── SKILL.md
│   ├── security-best-practices/
│   │   └── SKILL.md
│   ├── typescript/
│   │   └── SKILL.md
│   └── wordpress/
│       └── SKILL.md
└── local/                            # GITIGNORED — machine-specific, never committed
    ├── plans/                         # Ephemeral execution plans
    ├── prompts/                       # Private project prompts
    │   └── my-project.prompt.md       # /my-project
    ├── pull-requests/                 # Saved PR drafts
    └── skills/                        # Private domain/project skills
        ├── _template/
        │   └── PROJECT-SKILL-template.md  # Starter template for new project skills
        └── my-private-skill/
            └── SKILL.md
```

---

## Skills vs. Prompts

| | **Skill** | **Prompt** |
|---|---|---|
| **Purpose** | Load domain knowledge — *how to think about a topic* | Trigger a task workflow — *how to do an action* |
| **Examples** | `_template/SKILL-template.md` | `code-review`, `pull-request_`, `ticket-writing` |
| **File format** | `skills/my-skill/SKILL.md` | `prompts/my-prompt.prompt.md` |
| **Invoked by** | Agent reads it when context is relevant; or explicitly requested | `/slash-command` in Copilot Chat |
| **Content** | Reference material, patterns, conventions | Step-by-step instructions for a specific task |
| **Loaded automatically?** | Proactively by the agent when relevant | Only when you invoke the slash command |

**Rule of thumb:** If you're describing *knowledge*, it's a skill. If you're describing *a task to perform*, it's a prompt.

---

## Naming Conventions

### Prompts

- **File name:** `my-prompt.prompt.md` — kebab-case, all lowercase, ends in `.prompt.md`
- **`name` frontmatter field:** all lowercase or kebab-case — camelCase is not supported
- **Invocation:** `/my-prompt` slash command in Copilot Chat

### Skills

- **Structure:** a directory named after the skill containing a `SKILL.md` file
- **Example:** `skills/my-skill/SKILL.md`
- The agent auto-loads the skill when it determines the context is relevant
- Use lowercase `SKILL.md` for cross-platform consistency

---

## VS Code Configuration

Add the following to your workspace `.code-workspace` file's `"settings"` block.
Adjust paths to match where you cloned this repo.
Code > Preferences > Settings > WorkSpace > "Chat" open link `edit in settings.json`.
**Adjust path as needed to point to the agents repo**

```json
"chat.instructionsFilesLocations": {
    "~/Websites/agents/global/copilot-instructions.md": true
},
"chat.agentSkillsLocations": {
    "~/Websites/agents/skills": true,
    "~/Websites/agents/local/skills": true
},
"chat.promptFilesLocations": {
    "~/Websites/agents/prompts": true,
    "~/Websites/agents/local/prompts": true
},
"chat.agentFilesLocations": {
    "~/Websites/agents": true
}
```

### Profile settings (optional)

To apply base instructions even when working outside your main workspace, add the
instructions entry to your active VS Code profile's `settings.json`:
Code > Preferences > Settings > User > "Chat" open link `edit in settings.json`.

```json
"chat.instructionsFilesLocations": {
    "~/Websites/agents/global/copilot-instructions.md": true
}
```

---

## How `global/copilot-instructions.md` Works

`chat.instructionsFilesLocations` **automatically injects** the file into every Copilot
interaction — no manual reference needed. It behaves like a system prompt: always-on,
applied before any message. A fresh chat session is required after first configuring the setting.

---

## Machine Setup

### 1. Clone and open

Clone this repo and add it as a folder in your `.code-workspace` file:
**Critical: examples assume your agents repo is ~/Websites/agents/**

```json
{
    "name": "Agents",
    "path": "Websites/agents"
}
```

### 2. Add workspace settings

Copy the four `chat.*` settings above into the `"settings"` block of your workspace file,
updating the paths to match your local clone location.
(local is gitignored and will not show as changes as you add custom project skills/prompts)

### 3. Create `local/` directories

```bash
mkdir -p ~/Websites/agents/local/prompts
mkdir -p ~/Websites/agents/local/skills
mkdir -p ~/Websites/agents/local/pull-requests
mkdir -p ~/Websites/agents/local/plans
```

### 4. Add machine-specific project prompts

Create a `.prompt.md` per project in `local/prompts/`. Each becomes a `/slash-command`:

```
local/prompts/my-project.prompt.md   →  /my-project
```

### 5. Per-machine differences

The `agents` repo is identical on every machine. Only `local/` content differs:
- Add project prompts relevant to that machine's projects
- Add domain knowledge skills relevant to that machine's work
- Workspace and profile settings point to the same `~/Websites/agents/` path

---

## Usage

```
/code-review       ← review work with a focus on security, code quality, and best practices
/pull-request      ← write a PR from actual git commits
/ticket-writing    ← write a ticket as Project Manager, Product Owner, or QA
/release-notes     ← format GitHub release notes
```
