# ai-skills

A personal collection of agent `SKILL.md` files — reusable instruction packs that teach AI coding/agent tools (Antigravity, Claude Code, Cursor, Muse, and similar) a specific discipline or workflow.

Each skill lives in its own folder, named after the skill, containing a single `SKILL.md`:

```
ai-skills/
├── README.md
└── <skill-name>/
    └── SKILL.md
```

## Skills

| Skill | What it does |
|---|---|
| [gemini-multiturn-conversation](gemini-multiturn-conversation/SKILL.md) | Stateful multi-turn Gemini conversations with `@google/genai` — use the Interactions API (`previous_interaction_id`) or structured `user`/`model` turn arrays with SHA-256 hash-synced workspace state, so follow-up prompts stop re-sending the full codebase/document on every turn. |

## Using a skill

Copy the skill's folder into wherever your agent tool loads skills from — for example:

- **Muse:** `~/workspace/custom-skills/<skill-name>/SKILL.md`
- **Claude Code / Cursor:** your project's or user's skills directory

The `name` and `description` in each file's frontmatter tell the agent when to reach for it.

## Adding a skill

1. Create a folder named after the skill (kebab-case, matching the `name` in the frontmatter).
2. Drop the `SKILL.md` inside it.
3. Add a row to the Skills table above.
