# Definitions: name, description, argument descriptions, tool set

The definition is what the agent reads to decide whether to call a tool
and how to write the call. It's in the prompt on every turn, whether the
tool is used or not. Anthropic calls detailed descriptions "by far the most
important factor in tool performance." A study of 856 public MCP tools
found at least one defect in 97% of their descriptions, most often an
unclear purpose (56%). Definitions are also permanent context: five MCP
servers' definitions took about 55K tokens before a conversation began. So
every sentence has to change what the agent does.

## Name

- **A verb and a noun in the agent's words**: `add_beat`,
  `search_storyboard`, `get_deck`. Namespace by module when a menu mixes
  modules (this repo's Kernel names are dotted: `deck.get_deck`).
- **Different from every sibling at a glance.** Two tools with similar
  names and compatible arguments cause the one hallucination a schema check
  can't catch: a "borrowed signature", where the right arguments go to the
  wrong tool. Make the names say the difference, or merge the tools.
- **One verb set per menu, held consistently.** For example, `get_` reads
  one thing by its handle, `search_` finds things by words, `list_` lists a
  small set, and `add_` / `update_` / `remove_` write. Once a menu uses a
  verb for something, it means that everywhere on the menu.

## Description

The before/after texts in this file are illustrative, not the current
wording of these tools.

Write it for the new hire. Cover these, in this order, usually in 3–6
sentences (more only for a genuinely complex tool):

1. **What it does**, in the agent's terms, as one sentence the agent could
   pick the tool by.
2. **When to use it**, and which sibling to use instead, named.
3. **What it needs that isn't obvious**: a call to make first, a
   convention (code formats, what "wave" means here), and where each id
   comes from.
4. **What comes back**: shape and typical size, what it fills or repairs,
   and what an empty or truncated reply looks like.
5. **What it changes**: whether it can be undone (and with what), and
   whether it's safe to repeat.
6. **Limits that change behavior**: caps, what it refuses, and what it
   doesn't return.

Keep these out:

- **Anything the schema already says**: types, the list of enum values,
  which fields are required. Explain a value only when its meaning isn't
  obvious from its name.
- **Implementation, history, and plumbing**: module names, rule codes,
  ticket numbers, "replaces X", "the SPA's own write".
- **One agent's policy.** The description has to be true on every menu that
  carries the tool. Guidance for one agent goes in that agent's prompt, or
  in a menu-scoped overlay (here, `tool_description/<menu>/<tool>`).
- **Bans with no positive target.** "For one slide, use `get_beat`" works
  better than "Don't use this for single slides", which puts the wrong use
  in front of the agent.
- **Padding.** Longer isn't better. Filling in every component of every
  description raised task success by a median of 5.85 points, but it added
  67% more steps and made results worse in 1 of every 6 cases. Add a
  sentence when a transcript shows the agent needed it.

**Before:**
```
Gets deck info.
```
**After:**
```
Read one built deck: its slides (number, title, status), the brand it's in, and any open QC findings, about 20 tokens per slide. Use it to answer questions about a deck or to find a slide number before editing; to change a deck's look use deck.retheme, and to rebuild it use deck.build_deck. With no version named, it reads the current published version. It changes nothing.
```

**Before**, a description that narrates plumbing:
```
The SPA's and chat's own beat write: a whole BeatPatch, with the caller's own optional base_revision check and its pins.
```
**After**, if the tool belonged on an agent menu:
```
Change fields on one slide. Send only the fields you are changing; anything you leave out stays as it is. The reply names what changed and anything the writer filled in or repaired. If someone else changed the same field since your last read, nothing is applied and the reply shows the current text.
```

## Argument descriptions

Every argument whose name doesn't fully explain it gets a description. In
this repo that's `Field(description=...)`, which lands in the JSON schema
the model reads. Say:

- **What it holds**, with one realistic value:
  `Question code as shown by search_questions, e.g. "Q12".`
- **Where the agent gets it**, meaning which tool returns it. This is
  what stops invented ids.
- **What leaving it out does**: `Default: the latest wave.`
- **Units and format**: `Percent, 0–100.` `ISO date, e.g. 2026-09-30.`
- **Interactions** with other arguments: `Ignored when query is given.`
  It's better still to design the interaction away (`inputs.md` § Shape).

Leave out what the name and type already say, and don't repeat the tool
description.

## Examples

A schema can't show conventions: formats, which optional arguments go
together, what a nested row looks like. For format-sensitive or complex
arguments, add 1–5 realistic examples covering the minimal, partial, and
full forms. Anthropic measured accuracy on complex parameters rising from
72% to 90% with examples. Add them only where the schema leaves real
ambiguity: elsewhere, examples were the least useful part of a
description. Use realistic values (`"Q12"`, `"2024-Q3"`), never
`"string"`. Here, an example goes in the argument's description ("e.g.
…") or in `Field(examples=[...])`.

## Keeping it true

- **Change the description in the same edit as the behavior.** A stale
  description is a wrong instruction that the agent follows faithfully.
- **Read it as the model gets it**: the rendered tool schema (`bq tools`,
  or `menu_schema(menu)`), not the docstring in the source. Check every
  title, default, and enum pydantic emits, because those are prompt too.

## Tool set

- **Keep the menu small and each tool distinct.** Selection gets worse as a
  menu grows. Anthropic's tool search, which shows the model only the
  relevant definitions, raised Opus 4 from 49% to 74% on its evaluation.
  OpenAI's guidance is fewer than ~20 functions at the start of a turn.
- **Merge tools that are always called in sequence**, and put a read the
  write always needs inside the write.
- **Split by risk**: reads, writes, and destructive writes each get their
  own tool.
- **Use an `action` enum only when the actions share their arguments**
  (Anthropic's example is `create` / `review` / `merge` for a PR). When
  each action takes different arguments, separate tools keep each schema
  flat.
- **A summary tool plus a get-one tool** beats a single tool with a
  `detail` flag when the detail needs its own arguments. When it doesn't,
  a `detail` flag is fine.
- **UI-only tools stay off agent menus.** A tool on no menu costs the
  agent nothing.
