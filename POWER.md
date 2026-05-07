---
name: "resolve-ai"
displayName: "ResolveAI"
description: "Automatically resolve unresolved Figma comments by interpreting design feedback, executing changes via the Figma Plugin API, and replying to commenters. Supports direct edits, questions, suggestions, and feedback classification."
keywords: ["figma", "comments", "design", "resolve", "feedback", "review", "collaboration"]
author: "Delhivery Design Team"
---

# ResolveAI

## Overview

This power automates the resolution of Figma design comments. When given a Figma file URL, it:

1. Fetches all unresolved comments from the file or a specific frame/section
2. Maps each comment to its target node using offset coordinates
3. Classifies the comment intent (edit request, question, suggestion, or feedback)
4. Executes design changes for actionable comments via the Figma Plugin API
5. Replies to each comment with a smart, contextual response mentioning the commenter
6. Optionally sends Google Chat notifications to the team

This eliminates the manual back-and-forth of reading comments, making changes, and replying — turning hours of design review into minutes.

## Onboarding

### Prerequisites

- **Figma Personal Access Token** with read/write access to files and comments
  - Generate at: Figma → Settings → Personal access tokens
- **Figma file edit access** — you must have edit permissions on the target file
- **Google Chat Webhook URL** (optional) — for team notifications when comments are resolved

### Environment Variables

Set these in your environment or `.env` file:

| Variable | Required | Description |
|----------|----------|-------------|
| `FIGMA_TOKEN` | Yes | Figma personal access token |
| `GCHAT_WEBHOOK_URL` | No | Google Chat space webhook for notifications |
| `GCHAT_USER_MAP` | No | JSON mapping of Figma handles to Google Chat user IDs |

### Setup

1. Install this power in Kiro
2. Set your `FIGMA_TOKEN` environment variable
3. (Optional) Configure Google Chat webhook for team notifications
4. Provide a Figma URL when triggered

## Common Workflows

### Workflow 1: Resolve All Comments on a Frame

**Goal:** Process all unresolved comments on a specific Figma frame.

**Steps:**
1. Copy the Figma URL of the frame (right-click → Copy link to selection)
2. Trigger the power and provide the URL
3. The agent fetches comments, classifies each one, and acts accordingly
4. Design changes are made, replies are posted, and a screenshot verifies the result

**URL Format:**
```
https://figma.com/design/:fileKey/:fileName?node-id=X-Y
```

### Workflow 2: Resolve Comments on a Section

**Goal:** Process all unresolved comments within a Figma section (and its child frames).

Same as Workflow 1, but provide the URL of a section node. The agent will:
- Detect that the target is a section
- Collect all descendant node IDs
- Filter comments that belong to any node within that section

### Workflow 3: With Google Chat Notifications

**Goal:** Resolve comments and notify the team via Google Chat.

**Setup:**
```bash
GCHAT_WEBHOOK_URL=https://chat.googleapis.com/v1/spaces/SPACE_ID/messages?key=KEY&token=TOKEN
GCHAT_USER_MAP={"Chetan":"users/123456","Rahul":"users/789012"}
```

After resolving comments, a summary message is posted to the configured Google Chat space with @mentions for each commenter.

## Comment Classification

The agent classifies each comment before acting:

| Category | Signals | Action |
|----------|---------|--------|
| **Direct Edit** | "change to", "remove", "delete", "make this", "update" | Execute the change, reply confirming |
| **Question** | "?", "why", "what is", "should we" | Analyze design context, reply with answer |
| **Suggestion** | "what if", "maybe", "consider", "how about" | Analyze proposal, reply with pros/cons |
| **Feedback** | "looks good", "this seems off", "I noticed" | Acknowledge, investigate if needed |

**When in doubt:** The agent replies asking for clarification rather than making a wrong edit.

## Offset-to-Node Mapping

Comments in Figma have an offset (x, y) relative to the parent node. The agent traverses the node tree to find the deepest/smallest node containing that offset point — that's the comment target.

## Supported Edit Operations

- **Text changes** — Update `node.characters` (with font loading)
- **Color/fill changes** — Update `node.fills` (colors in 0–1 range)
- **Remove elements** — Call `node.remove()`
- **Resize** — Call `node.resize()`
- **Move/reparent** — Reposition or reparent nodes
- **Add icons/components** — Import from design system via `importComponentByKeyAsync`

## Troubleshooting

### "Figma token invalid or expired"
**Cause:** Token doesn't have the required permissions or has expired.
**Solution:** Generate a new token at Figma → Settings → Personal access tokens. Ensure it has file read/write and comment read/write scopes.

### "Node not found" errors
**Cause:** The comment references a node that was deleted or the page hasn't been loaded.
**Solution:** The agent switches pages using `setCurrentPageAsync` before accessing nodes. If the node was deleted, the comment is skipped with a reply noting the target no longer exists.

### Comments on a different frame are being processed
**Cause:** The URL might point to a section containing multiple frames.
**Solution:** Use a more specific URL pointing to the exact frame you want to process.

### Google Chat notifications not sending
**Cause:** `GCHAT_WEBHOOK_URL` is not set or the webhook is invalid.
**Solution:** Verify the webhook URL in your `.env` file. Create a new webhook in Google Chat: Space → Apps & integrations → Webhooks.

## Best Practices

- Always verify changes with a screenshot after processing all comments
- Process comments one at a time to avoid conflicts
- For ambiguous comments, reply asking for clarification rather than guessing
- Use the design system's existing components (via `search_design_system`) when adding new elements
- Load fonts before any text modifications

## MCP Config Placeholders

**Before using this power, replace the following placeholder in `mcp.json`:**

- **`YOUR_FIGMA_TOKEN_HERE`**: Your Figma personal access token.
  - **How to get it:**
    1. Go to [Figma Settings](https://www.figma.com/settings)
    2. Scroll to "Personal access tokens"
    3. Click "Generate new token"
    4. Copy the token (starts with `figd_`)
    5. Paste it as the value for `FIGMA_API_KEY`

---

**MCP Server:** figma
**Package:** `figma-developer-mcp`
