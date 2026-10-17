---
name: gemini-multiturn-conversation
description: Use this skill when building or upgrading AI chat, coding assistants, or document/workspace editors with Google Gemini (@google/genai) to use stateful multi-turn conversations instead of re-sending full codebases, documents, or context on every prompt.
---

# Stateful Multi-Turn Gemini Conversations (`@google/genai`)

## Core Problem This Skill Solves

Many AI assistants accidentally treat every prompt as a stateless single-turn request—re-uploading the entire document, codebase, or workspace state on every follow-up question or sequential edit, or flattening prior turns into a single text blob. This wastes tokens, increases latency, and degrades reasoning quality.

Instead, maintain a **stateful multi-turn conversation** where:
1. Initial context (code, document, or project state) is established on Turn 1.
2. Subsequent turns continue the same conversation (`previous_interaction_id` or structured `user`/`model` turns in `contents`) without re-sending unchanged state.
3. Full code or document state is only re-sent when the user manually edits the workspace outside the AI flow (detected via SHA-256 hash comparison).

---

## Pattern 1: Server-Managed Multi-Turn State (Interactions API)

When using Google's Interactions API (`@google/genai` `>= 2.3.0`), Gemini holds the conversation context on Google's servers (`store: true` by default). Chain turns together by passing `previous_interaction_id: interaction.id`:

### TypeScript (`@google/genai`)
```typescript
import { GoogleGenAI } from "@google/genai";

const client = new GoogleGenAI({});

// Turn 1: Establish the conversation and initial workspace/document state
const turn1 = await client.interactions.create({
  model: "gemini-3.7-flash",
  system_instruction: "You are an expert coding assistant. Remember the active code across turns.",
  input: `Here is my current file:\n\`\`\`ts\n${initialCode}\n\`\`\`\nAdd input validation.`,
});

console.log(turn1.output_text);

// Turn 2: Follow-up question or sequential edit — DO NOT re-send the file if unchanged!
// Google's server already holds Turn 1's input and model output via previous_interaction_id.
const turn2 = await client.interactions.create({
  model: "gemini-3.7-flash",
  system_instruction: "You are an expert coding assistant. Remember the active code across turns.",
  previous_interaction_id: turn1.id,
  input: "Why did you choose that validation pattern, and can we also log rejected inputs?",
});

console.log(turn2.output_text);
```

### Python (`google-genai`)
```python
from google import genai

client = genai.Client()

# Turn 1: Establish initial context
turn1 = client.interactions.create(
    model="gemini-3.7-flash",
    system_instruction="You are an expert coding assistant. Remember the active code across turns.",
    input=f"Here is my current file:\n```py\n{initial_code}\n```\nRefactor this into a class.",
)

# Turn 2: Chain with previous_interaction_id without re-uploading the file
turn2 = client.interactions.create(
    model="gemini-3.7-flash",
    system_instruction="You are an expert coding assistant. Remember the active code across turns.",
    previous_interaction_id=turn1.id,
    input="Now add a unit test method for edge cases.",
)
```

> **Important Rules for `interactions.create`**:
> - `system_instruction`, `tools`, and `generation_config` are interaction-scoped—re-specify them on each turn.
> - Keep `store: true` (the default); setting `store: false` disables `previous_interaction_id`.

---

## Pattern 2: Application-Managed Multi-Turn `contents` with Hash-Synchronized Workspace State

When your backend stores conversation turns in its own database (or uses `ai.models.generateContent` / `ai.chats.create` with structured JSON schemas and diff/apply workflows), follow these five rules so you never redundantly re-send unchanged files or workspace sections:

### 1. Use Native `role: 'user' | 'model'` Turn Arrays
Never flatten prior turns into a single string inside one user message. Always construct a proper multi-turn `contents` array:
```typescript
type ContentTurn = {
  role: "user" | "model";
  parts: Array<{ text: string }>;
};
```

### 2. Track Workspace Synchronization by SHA-256 Hash
Maintain a per-conversation synchronization record on the client:
```typescript
interface SyncedConversationState {
  activeDocumentSha256: string | null;
  secondaryContextHash: string | null; // e.g., other open tabs, scenes, or project tree summary
}
```
- **When an `ask` (Q&A) turn completes**: `activeDocumentSha256` remains the SHA-256 of the document that was sent.
- **When an `edit` turn completes with a proposal**: `activeDocumentSha256` updates to `proposal.sha256` (because the `model` turn in the conversation history now contains the proposed code).
- **When the user clicks Apply**: `activeDocumentSha256` matches the applied `proposal.sha256`.
- **When the user clicks Discard or Undo**: `activeDocumentSha256` reverts to the pre-proposal `base.sha256`.

### 3. Omit Unchanged Workspace Code on Follow-Up Turns (`sourceSynchronized: true`)
Before sending a new turn, compare `sha256(currentEditorCode)` with `syncedState.activeDocumentSha256`:
- **If `sha256(currentEditorCode) === syncedState.activeDocumentSha256` (`sourceSynchronized: true`)**:
  - **In `Ask` (Q&A) mode**: Do not upload the file at all (`source: null`). The model already has the exact active code in its conversation history.
  - **In `Edit` mode**: Do not duplicate `currentSource` inside the new `user` turn sent to Gemini (`{ mode: "edit", prompt, sourceSynchronized: true }`). Keep the local base snapshot only for client-side diffing/Undo and backend validation.
- **If `sha256(currentEditorCode) !== syncedState.activeDocumentSha256` (`sourceSynchronized: false`)**:
  - The user manually edited the file or switched files. Attach the updated `currentSource` on this turn to re-synchronize the conversation, and update `activeDocumentSha256`.

### 4. Keep Full Source Only Once in History (Compact Superseded Turns)
In a 10-turn editing session, do **not** keep 10 full copies of the file across all 10 historical turns.
- Keep the full code **only on the most recent turn that introduced or modified the active code** (either the `model` turn that proposed the latest active edit, or the `user` turn that last attached manually edited code).
- For all older turns whose code was superseded, replace the inline code string with compact metadata (`{ supersededSourceSha256: "..." }`). This keeps token usage flat across long sessions while preserving the full conversational reasoning thread.

### 5. Deduplicate Read-Only Secondary Context (Other Files / Tabs / Sections)
If your app includes a read-only summary of non-active files or sections:
- Send that summary on **Turn 1** of the conversation.
- On Turn 2+, only re-send the summary if its hash changed (`secondaryContextHash !== currentSecondaryContextHash`).

---

## Reference Implementation: Multi-Turn Request Builder (TypeScript)

```typescript
interface HistoryTurn {
  prompt: string;
  response: string;
  mode: "ask" | "edit";
  baseSource?: string | null;
  baseSha256?: string | null;
  proposalSource?: string | null;
  proposalSha256?: string | null;
  outcomeStatus?: "none" | "proposed" | "applied" | "discarded" | "undone";
}

export function buildMultiTurnContents(params: {
  history: HistoryTurn[];
  mode: "ask" | "edit";
  prompt: string;
  currentSource: string | null;
  sourceSynchronized: boolean;
}) {
  const { history, mode, prompt, currentSource, sourceSynchronized } = params;

  // Identify which historical turn holds the currently active source
  let latestActiveSourceIndex = -1;
  if (sourceSynchronized || currentSource === null) {
    for (let i = history.length - 1; i >= 0; i--) {
      const item = history[i];
      const hasValidProposal =
        item.mode === "edit" &&
        item.proposalSource &&
        item.outcomeStatus !== "discarded" &&
        item.outcomeStatus !== "undone";
      if (hasValidProposal || item.baseSource) {
        latestActiveSourceIndex = i;
        break;
      }
    }
  }

  const contents: Array<{ role: "user" | "model"; parts: Array<{ text: string }> }> = [];

  for (let i = 0; i < history.length; i++) {
    const item = history[i];
    const isLatestSourceTurn = i === latestActiveSourceIndex;
    const hasActiveProposal =
      item.mode === "edit" &&
      Boolean(item.proposalSource) &&
      item.outcomeStatus !== "discarded" &&
      item.outcomeStatus !== "undone";

    // User turn in history
    const userPayload: Record<string, unknown> = {
      mode: item.mode ?? "ask",
      prompt: item.prompt,
    };
    if (isLatestSourceTurn && !hasActiveProposal && item.baseSource) {
      userPayload.currentSource = item.baseSource;
    } else if (item.baseSha256) {
      userPayload.supersededSourceSha256 = item.baseSha256;
    }
    contents.push({ role: "user", parts: [{ text: JSON.stringify(userPayload) }] });

    // Model turn in history
    const modelPayload: Record<string, unknown> = {
      kind: item.mode === "edit" ? "code_proposal" : "answer",
      explanation: item.response,
    };
    if (isLatestSourceTurn && hasActiveProposal && item.proposalSource) {
      modelPayload.proposedSource = item.proposalSource;
    } else if (item.proposalSha256) {
      modelPayload.supersededProposalSha256 = item.proposalSha256;
    }
    contents.push({ role: "model", parts: [{ text: JSON.stringify(modelPayload) }] });
  }

  // Final (current) user turn
  const omitDuplicateSource = sourceSynchronized && latestActiveSourceIndex !== -1;
  const currentTurnPayload: Record<string, unknown> = { mode, prompt };
  if (omitDuplicateSource) {
    currentTurnPayload.sourceSynchronized = true;
  } else {
    currentTurnPayload.currentSource = currentSource;
  }

  contents.push({ role: "user", parts: [{ text: JSON.stringify(currentTurnPayload) }] });
  return contents;
}
```

---

## Checklist When Reviewing or Implementing Multi-Turn AI

- [ ] Are we using `@google/genai` (`GoogleGenAI`) and never the deprecated `@google/generative-ai`?
- [ ] Does Turn 2+ preserve multi-turn `user` / `model` roles (or `previous_interaction_id`) instead of flattening history into a single prompt string?
- [ ] On follow-up questions (`Ask` mode) when the editor code has not changed, do we omit re-uploading the code from the client?
- [ ] On sequential AI edits (`Edit` mode) when the editor code matches the last turn, do we omit duplicate `currentSource` from the new user prompt in `contents`?
- [ ] Are older superseded code versions compacted in history so only the active code version is sent in full?
- [ ] If the user manually edits the document or undoes a change, does the SHA-256 mismatch automatically trigger re-synchronization on the next turn?
