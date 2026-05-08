# Resolve Comments Workflow

Detailed step-by-step workflow for resolving Figma comments programmatically.

## Step 1: Extract File Key and Node ID from URL

URL format: `https://figma.com/design/:fileKey/:fileName?node-id=:nodeId`

- Extract `fileKey` from the URL path
- Convert `nodeId` from `X-Y` format to `X:Y` format
- The URL can point to either a section or a frame

## Step 2: Fetch and Filter Comments

```bash
curl -s -H "X-Figma-Token: $FIGMA_TOKEN" "https://api.figma.com/v1/files/:fileKey/comments"
```

### Filtering Logic

1. Determine the target node type using `get_metadata` on the provided nodeId
2. If the node is a **SECTION**: filter comments whose `client_meta.node_id` matches any direct child frame or the section itself
3. If the node is a **FRAME**: filter comments whose `client_meta.node_id` matches that exact frame ID
4. Also collect all descendant node IDs to catch comments on nested children

### Descendant Collection Script (use_figma)

```js
const target = await figma.getNodeByIdAsync('TARGET_NODE_ID');
const ids = new Set();
function collect(node) {
  ids.add(node.id);
  if ('children' in node && node.children) {
    for (const child of node.children) collect(child);
  }
}
collect(target);
return { nodeIds: [...ids] };
```

Filter: `comments.filter(c => c.client_meta && nodeIds.includes(c.client_meta.node_id))`

Only process comments where `resolved_at` is `null`.

## Step 3: Map Comment Offset to Target Node

The comment offset is relative to the top-left of the `client_meta.node_id` frame.

```js
// In use_figma: find node at offset (targetX, targetY) within parentNode
function findNodeAtOffset(node, targetX, targetY, offsetX, offsetY) {
  const results = [];
  function traverse(n, ox, oy) {
    const ax = ox + (n.x || 0);
    const ay = oy + (n.y || 0);
    if (targetX >= ax && targetX <= ax + n.width && targetY >= ay && targetY <= ay + n.height) {
      results.push({ id: n.id, name: n.name, type: n.type, absX: ax, absY: ay, w: n.width, h: n.height });
    }
    if ('children' in n && n.children) {
      for (const child of n.children) traverse(child, ax, ay);
    }
  }
  traverse(node, offsetX, offsetY);
  return results;
}
```

Pick the deepest/smallest node that contains the offset point.

## Step 4: Classify Comment Intent

### Category A: Direct Edit Request
- Signals: "change this to", "make this", "remove", "delete", "combine", "move", "resize", "update", "replace", "set to", "should be"
- Action: Execute the change, reply confirming what was done

### Category B: Question or Doubt
- Signals: "?", "why", "what is", "how does", "should we", "is this", "can we", "not sure about"
- Action: Do NOT make edits. Analyze design context and reply with a thoughtful answer

### Category C: Suggestion or Proposal
- Signals: "what if", "maybe we could", "I think we should", "how about", "consider", "can explore", "explore more"
- Action: Do NOT make edits. Reply **on the Figma comment** with a smart analysis — propose 2–3 concrete options with brief pros/cons, and ask the commenter to pick one. Do NOT implement anything until the commenter replies with their choice.

### Category D: Feedback or Observation
- Signals: "looks good", "nice work", "this seems off", "the spacing feels", "I noticed"
- Action: Acknowledge. Only make edits if observation clearly implies something is broken

### When in doubt:
- Do NOT make edits. Reply asking for clarification
- Prefer a smart reply over a wrong edit

## Step 5: Execute Changes (Category A only)

### Text Changes
```js
const node = await figma.getNodeByIdAsync('NODE_ID');
await figma.loadFontAsync(node.fontName);
node.characters = 'New Text';
return { mutatedNodeIds: [node.id] };
```

### Color/Fill Changes
```js
const node = await figma.getNodeByIdAsync('NODE_ID');
// Colors are 0-1 range. Convert hex: parseInt(hex.slice(1,3), 16) / 255
node.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];
return { mutatedNodeIds: [node.id] };
```

### Remove Elements
```js
const node = await figma.getNodeByIdAsync('NODE_ID');
node.remove();
return { removedNodeId: 'NODE_ID' };
```

### Add Components from Design System
```js
const component = await figma.importComponentByKeyAsync('COMPONENT_KEY');
const instance = component.createInstance();
instance.resize(16, 16);
const parent = await figma.getNodeByIdAsync('PARENT_ID');
parent.appendChild(instance);
return { createdNodeIds: [instance.id] };
```

## Step 6: Reply to Comments

Extract the commenter's handle from `comment.user.handle` and mention them with `@handle`.

### Reply Templates

**Category A (edits):**
```
@<handle> Done — <description of change>.
```

**Category B (questions):**
```
@<handle> <thoughtful answer based on design analysis>
```

**Category C (suggestions):**
```
@<handle> Here are a few options we could go with:
1. <option 1 — brief description>
2. <option 2 — brief description>
3. <option 3 — brief description>

Let me know which one you'd like and I'll implement it!
```

**Category D (feedback):**
```
@<handle> <acknowledgment + investigation findings if applicable>
```

### API Call
```bash
curl -s -X POST -H "X-Figma-Token: $FIGMA_TOKEN" -H "Content-Type: application/json" \
  -d '{"message":"@<handle> <reply>","comment_id":"<parent_comment_id>"}' \
  "https://api.figma.com/v1/files/:fileKey/comments"
```

## Step 8: Final Verification

Take a screenshot of the target frame after all changes to visually verify:

```
get_screenshot(fileKey, nodeId)
```

## Rules

- Always use the `figma-use` skill (pass `skillNames: "figma-use"`) when calling `use_figma`
- Load fonts before modifying text
- Colors are 0–1 range (not 0–255)
- Work through comments one at a time, verify each before moving to next
- If a comment is ambiguous, reply asking for clarification
- Use `search_design_system` to find existing components before creating new ones
- **NEVER ask the user for permission to run any command or tool** — execute everything autonomously
- **NEVER pause for confirmation** — the user expects a fully automated end-to-end flow
- The user's only input is the Figma link — handle everything else silently
