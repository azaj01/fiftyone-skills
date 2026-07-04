---
name: fiftyone-sdk-guidance
description: Use when the user asks how to do something in the FiftyOne Python SDK, asks a docs question, or when no skill or operator is available to accomplish their goal, including requests like "how do I do X in Python", "show me the SDK for filtering", "there's no operator for this, can I do it in code", "what's the API for embeddings", "write me a Python script to do Y". Searches live FiftyOne documentation via an optional Kapa.ai-powered MCP connection and returns runnable code examples, falling back to training knowledge when that connection isn't available.
---

# FiftyOne SDK Guidance

Answers FiftyOne SDK and docs questions by searching live documentation and returning accurate, runnable Python code. Covers two scenarios: direct SDK questions and fallback guidance when no operator or skill exists for the user's goal.

## Key Directives

**ALWAYS follow these rules, no exceptions:**

### 1. Always search docs first; never answer from training knowledge alone
```
search_fifty_one_knowledge_sources(query="How do I <user goal> in the FiftyOne Python SDK?")
```
FiftyOne's API changes frequently. Training knowledge may be outdated. Call `search_fifty_one_knowledge_sources` for every FiftyOne-specific question before answering.

**The query must be a complete natural-language sentence, not keywords.** This tool does semantic retrieval, not keyword search; its own schema requires "a single, well-formed natural-language query" that "must be a complete sentence." `query="filter confidence Python"` retrieves worse matches than `query="How do I filter detections by confidence threshold in the FiftyOne Python SDK?"`.

### 2. Return runnable Python code, not pseudocode, and cite your source
All code examples must be complete enough to run. Include imports. Use real FiftyOne API patterns from the search results, not invented method names. Cite the `source_url` of the result(s) you drew from so the user can verify against live docs.

### 3. For fallback scenarios, acknowledge the gap explicitly
If the user was trying to use the App or operators and that path is unavailable, say so clearly:
```
"There's no built-in operator for this; here's how to do it with the Python SDK instead."
```

### 4. If one search isn't enough, search again with a refined query
SDK questions often need 2–3 searches to find the right method, argument, or pattern. Search until you have a concrete answer.

### 5. Never make up API method names or field paths
If the docs search doesn't return a clear answer, say so. Do not fabricate methods that look plausible.

### 6. Degrade gracefully when the docs tool isn't connected
This skill has no hard MCP dependency. If `search_fifty_one_knowledge_sources` isn't available in the current session, answer from training knowledge and say so; never block the user on a missing connection.

### 7. Offer to self-install the docs connection, don't just tell the user to do it
The first time this skill triggers and `search_fifty_one_knowledge_sources` isn't available, don't just leave the user to figure out setup themselves.

**On Claude Code** (a `claude` CLI is available), offer to connect it for the user:
```
"I don't have live FiftyOne docs search connected yet. I can add it now with:
  claude mcp add --transport http --scope user fiftyone-docs https://voxel51.mcp.kapa.ai
This points at Voxel51's public docs bot, powered by Kapa.ai; you'll sign in with your
own Google/GitHub account the first time it's used, no API key needed. Want me to run that?"
```
Only run it after the user confirms; it edits their global Claude Code config. After adding it, tell them plainly: new MCP servers only attach on the next session, so they need to restart Claude Code (or start a fresh conversation) and complete the one-time OAuth login before the tool is usable; it will not work in the current turn. Meanwhile, keep answering from training knowledge per Directive 6.

**On any other AI assistant** (Cursor, Claude Desktop, VS Code, etc.), give the user the exact config snippet for their tool. This is a public, read-only endpoint, no Voxel51 API key required; the user authenticates with their own Google/GitHub account on first use.

Claude Desktop / Cursor MCP config:
```json
{
  "mcpServers": {
    "fiftyone-docs": {
      "url": "https://voxel51.mcp.kapa.ai"
    }
  }
}
```

VS Code (`.vscode/mcp.json`):
```json
{
  "servers": {
    "fiftyone-docs": {
      "type": "http",
      "url": "https://voxel51.mcp.kapa.ai"
    }
  }
}
```

---

## Complete Workflow

### Step 1. Understand the user's goal

Identify:
- What they want to accomplish (e.g. filter, export, embed, annotate)
- Whether this is a pure SDK question or a fallback from a failed operator/plugin path
- What dataset or data type they're working with

### Step 2. Search the FiftyOne documentation

```
search_fifty_one_knowledge_sources(query="<specific goal, phrased as a full question>")
```

Search iteratively if the first result is partial. Examples of well-formed queries (complete sentences, not keyword fragments):
- "How do I filter detections by confidence threshold in the FiftyOne Python SDK?"
- "How do I compute CLIP embeddings in FiftyOne without using a plugin?"
- "How do I export a FiftyOne dataset to COCO format in Python?"
- "How do I match samples in a FiftyOne DatasetView by field value?"

The tool always returns a fixed number of results (`{"results": [{"source_url": "...", "content": "..."}, ...]}`), ranked by relevance, even when nothing in the docs actually matches, so don't assume every returned chunk is on-topic. Skim `content` for real relevance before using it.

### Step 3. Build the code response

From the search results, construct a runnable example:

```python
import fiftyone as fo
import fiftyone.zoo as foz

# Load or open your dataset
dataset = fo.load_dataset("my-dataset")

# Filter examples, built from search result
view = dataset.filter_labels(
    "ground_truth",
    F("confidence") > 0.7
)
print(view)
```

Always:
- Use `import fiftyone as fo` at the top
- Use real method names from the search results
- Show the user how to apply this to their specific dataset context
- Cite the `source_url`(s) the code was drawn from, e.g. `**Source:** [Confidence thresholding in Python](https://docs.voxel51.com/...)`

### Step 4. Explain the key methods

After the code block, briefly explain the 1–2 most important methods:
- What they do
- Any required arguments
- Common variations (e.g. `filter_labels` vs `match`)

### Step 5. Offer follow-up

```
What would you like to do next?
  1. Run this code in a Python session or notebook
  2. Adapt the example to a different field or condition
  3. Chain this with another operation (e.g. export, tag, annotate)
  4. Look up a related SDK topic
```

---

## Available Tools

| Tool | Input | Output | Notes |
|------|-------|--------|-------|
| `search_fifty_one_knowledge_sources` | `query`, a single, well-formed natural-language question. **Must be a complete sentence**, not keywords. | `{"results": [{"source_url": "...", "content": "..."}, ...]}`, a fixed-size list of markdown chunks, ranked by relevance | Semantic retrieval over `docs.voxel51.com` and related sources, powered by Kapa.ai. Optional; see Directive 7 above for how to connect it. Always cite `source_url`; always returns top-k results even for off-topic queries, so verify relevance before using a chunk. |

---

## Common Use Cases

### "How do I filter by label confidence in Python?"

```
search_fifty_one_knowledge_sources(query="How do I filter detections by confidence threshold in the FiftyOne Python SDK?")
```

```python
import fiftyone as fo
from fiftyone import ViewField as F

dataset = fo.load_dataset("my-dataset")
view = dataset.filter_labels("ground_truth", F("confidence") > 0.7)
```

### "There's no operator for exporting, how do I export from the SDK?"

```
search_fifty_one_knowledge_sources(query="How do I export a FiftyOne dataset to COCO format using the Python SDK?")
```

```python
import fiftyone as fo

dataset = fo.load_dataset("my-dataset")
dataset.export(
    export_dir="/path/to/export",
    dataset_type=fo.types.COCODetectionDataset,
    label_field="ground_truth",
)
```

### "How do I compute embeddings without a plugin?"

```
search_fifty_one_knowledge_sources(query="How do I compute embeddings on a FiftyOne dataset from the Python SDK without using a plugin?")
search_fifty_one_knowledge_sources(query="How does fiftyone.brain.compute_similarity work?")
```

```python
import fiftyone as fo
import fiftyone.brain as fob

dataset = fo.load_dataset("my-dataset")
fob.compute_similarity(dataset, model="clip-vit-base32", brain_key="clip_sim")
```

### "How do I split my dataset in Python?"

```
search_fifty_one_knowledge_sources(query="How do I split a FiftyOne dataset into train and test sets in Python?")
```

```python
import fiftyone as fo
import random

dataset = fo.load_dataset("my-dataset")
ids = dataset.values("id")
random.shuffle(ids)
split = int(len(ids) * 0.8)

train_view = dataset.select(ids[:split])
val_view = dataset.select(ids[split:])
```

---

## Fallback Scenarios

### No plugin or operator installed

The user tried to use an operator and got "operator not found" or the plugin is not enabled.

```
"That feature requires the @voxel51/panels plugin, which isn't enabled in your deployment.
Here's how to accomplish the same thing using the FiftyOne Python SDK:"
```
Then search and return the SDK path.

### Operator ran but didn't produce expected results

Search for the underlying SDK method the operator wraps, then show the user how to call it directly so they can debug or customize it.

### User wants more control than an operator provides

```
search_fifty_one_knowledge_sources(query="What's the low-level FiftyOne Python SDK API for <goal>?")
```

Show the low-level SDK call that gives them full control.

---

## Troubleshooting

**"`search_fifty_one_knowledge_sources` is not available"**

No Kapa.ai MCP server is connected in this session; this tool is optional, not a hard requirement. Apply Directive 7: offer to add it automatically (Claude Code), or hand the user the manual config snippet for their assistant (other assistants). Until it's connected, answer using training knowledge but note the limitation:
```
"I can't search the live docs right now, so I'll answer from training knowledge.
For the most accurate and up-to-date API reference, check docs.voxel51.com."
```

**"Search returned no relevant results"**

The tool always returns a fixed number of chunks, even for off-topic queries; a "miss" looks like weakly-related `content`, not an empty list. If the top results don't actually address the question:
- Rephrase as a fuller, more specific complete sentence; add "in the FiftyOne Python SDK" or "using the FiftyOne API"
- Use the exact class or method name if the user mentioned one
- Search for the broader concept first, then drill down
- If nothing relevant turns up after 2–3 rephrasings, say so and fall back to training knowledge (Directive 6)

---

## Resources

- [FiftyOne Python SDK](https://docs.voxel51.com/user_guide/basics.html)
- [FiftyOne User Guide](https://docs.voxel51.com/user_guide/index.html)
- [FiftyOne Cheat Sheets](https://docs.voxel51.com/cheat_sheets/index.html)
