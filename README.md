# ResolveAI — Figma Comment Resolver

Automatically resolve unresolved Figma comments by interpreting design feedback, executing changes via the Figma Plugin API, and replying to commenters.

**Zero effort. Just paste a link.**

## What It Does

Drop a Figma frame or section URL and ResolveAI will:

1. Fetch all unresolved comments on that frame/section
2. Map each comment to its target node
3. Make the requested design changes
4. Reply to each commenter confirming what was done
5. Show you a summary of everything that was resolved

No prompts. No confirmations. No interruptions.

## Setup (one-time, ~3 minutes)

### Prerequisites

- [Kiro IDE](https://kiro.dev) installed
- Node.js 18+ (for `npx`)
- Figma account with **edit access** to target files
- **Kiro set to Autopilot mode** (critical — prevents permission popups mid-execution)

### Step 0: Switch to Autopilot Mode

This is the most important step. Without it, Kiro will ask for permission on every action and ruin the experience.

1. Open Kiro
2. Look at the chat input area — there's a mode toggle
3. Switch from **Supervised** → **Autopilot**

This lets ResolveAI run all commands (fetching comments, editing nodes, posting replies) without interrupting you.

### Step 1: Copy the Power

Clone this repo (or copy the `resolve-ai` folder) into your workspace:

```bash
git clone https://github.com/your-org/resolve-ai.git powers/resolve-ai
```

Your workspace should look like:

```
your-project/
├── powers/
│   └── resolve-ai/
│       ├── POWER.md
│       ├── README.md
│       ├── icon.svg
│       ├── mcp.json
│       └── steering/
│           └── resolve-comments.md
└── ...
```

### Step 2: Generate a Figma Personal Access Token

1. Go to [Figma Settings](https://www.figma.com/settings)
2. Scroll to **Personal access tokens**
3. Click **Generate new token**
4. Give it a name (e.g., "ResolveAI")
5. Select these scopes:
   - ✅ File content — Read and write
   - ✅ Comments — Read and write
6. Copy the token (starts with `figd_`)

### Step 3: Configure the Token

Open `powers/resolve-ai/mcp.json` and replace the placeholder:

```json
{
  "mcpServers": {
    "figma": {
      "command": "npx",
      "args": ["-y", "figma-developer-mcp", "--stdio"],
      "env": {
        "FIGMA_API_KEY": "figd_YOUR_TOKEN_HERE"
      }
    }
  }
}
```

> ⚠️ **Never commit your token to version control.** Use environment variables or a `.env` file in production setups.

### Step 4: Verify Setup

1. Open Kiro
2. Open the workspace containing the `powers/resolve-ai` folder
3. The MCP server should connect automatically (check the MCP Server view in the Kiro panel)
4. Try: "resolve comments on https://www.figma.com/design/YOUR_FILE_KEY/FileName?node-id=X-Y"

> **Important:** You MUST use **Autopilot mode** in Kiro for this power to work without interruptions. In Supervised mode, Kiro will ask permission for every command — defeating the purpose of automation. Switch to Autopilot before using this power.

## Usage

**Just paste a Figma link. That's it.**

```
resolve comments on https://www.figma.com/design/abc123/MyFile?node-id=50-312
```

ResolveAI does everything else — no questions asked, no permissions needed, no interruptions.

### How to get the link

1. Open your Figma file
2. Select the frame or section you want comments resolved on
3. Right-click → **Copy link to selection**
4. Paste it in Kiro chat

### What gets resolved

| Comment Type | Example | What Happens |
|------|---------|--------|
| Direct edit | "change this to 'Alerts'" | Makes the change, replies ✓ |
| Remove | "remove this icon" | Removes it, replies ✓ |
| Style change | "reduce font weight" | Adjusts style, replies ✓ |
| Question | "why is this here?" | Replies with analysis |
| Suggestion | "can explore more layouts" | Replies on Figma with options, asks commenter to pick one |
| Feedback | "looks good" | Acknowledges |

### Suggestions & Ideas

When a comment is a suggestion or idea (not a direct edit), ResolveAI will:
1. Reply directly on the Figma comment with 2–3 concrete options
2. Ask the commenter to pick which one they want implemented
3. No edits are made until the commenter responds

The conversation stays in Figma — the commenter picks their preferred option right there in the thread.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Kiro keeps asking for permission | Switch to **Autopilot mode** (mode toggle near chat input) |
| "Figma token invalid" | Regenerate token at Figma Settings → Personal access tokens |
| "Node not found" | Ensure the URL points to a valid frame/section you have access to |
| MCP server not connecting | Check Node.js is installed (`node --version`), restart Kiro |
| Comments not being processed | Verify comments are unresolved and on the specified node |
| Font loading errors | The file may use fonts not available — the agent will skip those edits |

## How It Works (Technical)

1. **Comment fetching** — Uses Figma REST API (`GET /v1/files/:key/comments`)
2. **Offset mapping** — Traverses the node tree to find the deepest node at the comment's (x, y) offset
3. **Intent classification** — Pattern-matches comment text to determine action type
4. **Execution** — Uses `use_figma` (Figma Plugin API) to modify nodes directly
5. **Reply** — Posts replies via Figma REST API (`POST /v1/files/:key/comments`)

## Contributing

1. Fork this repo
2. Make your changes
3. Test with a real Figma file
4. Submit a PR

## License

MIT
