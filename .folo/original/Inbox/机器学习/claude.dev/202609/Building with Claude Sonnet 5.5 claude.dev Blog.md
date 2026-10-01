---
title: "Building with Claude Sonnet 5.5 / claude.dev Blog"
source: "claude.dev"
category: "Inbox"
group: "机器学习"
author: "Addy Osmani"
url: "https://claude.dev/blog/building-with-claude-sonnet-5-5/"
published: 2026-09-28T08:00:00+08:00
saved: 2026-10-01T09:47:55+08:00
folo_key: "url::https://claude.dev/blog/building-with-claude-sonnet-5-5"
tags:
  - "folo"
  - "Inbox"
  - "claude.dev"
---

# Building with Claude Sonnet 5.5 / claude.dev Blog

> [!info] claude.dev · Inbox · 2026-09-28 08:00 · [原文](https://claude.dev/blog/building-with-claude-sonnet-5-5/)

## MIGRATING FROM SONNET 5

Thinking is on by default. If you ran Sonnet 5 with thinking off, you can use `between_tools` to turn off upfront thinking. Step 1 below shows how.

Change the model ID to `claude-sonnet-5-5`, then work through five breaking changes and one change to the response shape. The [Sonnet 5.5 migration guide](https://platform.claude.com/docs/en/models/sonnet-5-5/migration-guide) covers each one in full.

Claude Code can also do the migration for you. Run `/claude-api migrate this project to claude-sonnet-5-5` to invoke the bundled [Claude API skill](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/claude-api-skill#migrating-to-a-newer-claude-model), which applies the model ID swap and the breaking parameter changes across your code base.

### 1\. Turn off upfront thinking with between\_tools

On Sonnet 5.5, a request with no `thinking` field runs with adaptive thinking, and `thinking: {"type": "disabled"}` returns a 400 error. Send the new `between_tools` setting instead. With `between_tools`, thinking only happens between tool calls, and total response time is the same or faster.

CODEPython

# Before: Claude Sonnet 5client.messages.create(
    model="claude-sonnet-5",
    max_tokens=16000,
    thinking={"type": "disabled"},
    output_config={"effort": "xhigh"},
    messages=[{"role": "user", "content": "..."}],
)# After: Claude Sonnet 5.5client.messages.create(
    model="claude-sonnet-5-5",
    max_tokens=16000,
    thinking={"type": "between_tools"},
    output_config={"effort": "high"},
    messages=[{"role": "user", "content": "..."}],
)

The example also drops effort from `xhigh` to `high`, because `between_tools` has these limits:

- `between_tools` works at `low`, `medium` and `high` effort. At `xhigh` or `max` it returns a 400 error; to run there, use adaptive thinking.
- It takes no other field. Sending `display`, `budget_tokens` or `block_binding` with it returns a 400 error.
- With `between_tools`, effort can't change mid-conversation. To vary effort per turn, use adaptive thinking.
- The short progress updates the model writes between tool calls still come back as `thinking` blocks, with summary text. Read content blocks by type, and pass these blocks back unchanged with the rest of the assistant turn. Without tools, the response contains only text.
- It works on every platform that offers Sonnet 5.5, with no beta header. If your SDK version doesn't define `between_tools`, update it.

If you turn upfront thinking off with `between_tools`, use adaptive thinking instead for requests without tools that need a few steps of working out.

### 2\. Replace forced tool\_choice with auto plus strict tools

`tool_choice` of type `any` or `tool` returns a 400 error, including on the token counting endpoint. Send `auto`, mark the tool `strict: true` so its input matches the schema, and say in the prompt when to use it:

CODEPython

weather_tool = {
    "name": "get_weather",
    "description": "Get the current weather in a given location",
    "input_schema": {
        "type": "object",
        "properties": {"location": {"type": "string"}},
        "required": ["location"],
        "additionalProperties": False,
    },
    "strict": True,
}

client.messages.create(
    model="claude-sonnet-5-5",
    max_tokens=1024,
    tools=[weather_tool],
    tool_choice={"type": "auto"},  # was {"type": "tool", "name": "get_weather"}    messages=[
        {"role": "user", "content": "What's the weather in Paris? Use the get_weather tool."}
    ],
)

Strict tool use needs `additionalProperties: false` on every object.

### 3\. Keep conversations append-only

Sonnet 5.5 thinking blocks are tied to the model and the conversation. Sonnet 5.5 reads Sonnet 5's thinking blocks, so a conversation you switch from Sonnet 5 to Sonnet 5.5 keeps its reasoning. No other model reads Sonnet 5.5's blocks.

### 4\. Move computer use to the toolset

On the Claude API and Google Cloud, Sonnet 5.5 supports computer use only through `{"type": "computer_toolset_20260801"}`; a request that declares `computer_20251124` returns a 400 error. Drop the `anthropic-beta: computer-use-2025-11-24` header from your requests, and in the SDKs, remove the `betas` parameter and call the Messages API through the standard client rather than the beta namespace. Replace the `tools` entry, and update your agent loop for member `tool_use` blocks, batch actions and `toolset_name` on results. If you send the `fine-grained-tool-streaming-2025-05-14` beta header, remove it too, because alongside a toolset entry it returns a 400 error; set `eager_input_streaming: true` on each tool that needs it instead. Amazon Bedrock still accepts `computer_20251124`.

### 5\. Check your advisor pairing

With the advisor tool, a Sonnet 5.5 executor rejects Opus 4.8, Opus 4.7 and Sonnet 5 as advisors. Accepted advisors include Opus 5.5, Opus 5 and Sonnet 5.5 itself. Advice from every accepted advisor comes back encrypted, as an `advisor_redacted_result` block, so your code can't read the advice text.

### 6\. Read text between tool calls from thinking blocks

This change causes no errors, but a UI can stop showing the model's notes between tool calls. Those notes, when longer than a sentence or two, come back as progress-update `thinking` blocks, which are empty at the default `display`.

With adaptive thinking, set `thinking.display` to `"updates"` (beta, with the `thinking-display-updates-2026-08-18` header) or `"summarized"`, and render each non-empty `thinking` block before the `tool_use` block that follows it. With `between_tools`, the text comes back without `display`.

Sonnet 5.5 also adds per-message effort (beta), mid-conversation system messages and mid-conversation tool changes (beta). If you're moving from Sonnet 4.6 or earlier, or from Haiku 4.5, the [migration guide](https://platform.claude.com/docs/en/models/sonnet-5-5/migration-guide) has a checklist for each starting model.
