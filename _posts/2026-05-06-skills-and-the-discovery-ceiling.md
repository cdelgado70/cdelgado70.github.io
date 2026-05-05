---
layout: post
title: "Skills and the discovery ceiling: why your AI coding agent ignores most of what you install"
date: 2026-05-06
author: C. Delgado
description: "Why AI coding agents skip most of the skills you install — Anthropic's Agent Skills standard, the budget arithmetic, the hook that didn't work, and the architecture that did."
---

*Post 1 of 2. This post walks through the discovery ceiling in Anthropic's Agent Skills standard, the obvious first attempt at fixing it (a hook that didn't work), and the architecture that did. A follow-up post will cover the hook story in detail — why Claude Code's `UserPromptSubmit` hook can't reliably inject knowledge, what's filed about it, and what would actually fix it.*

---

## A small puzzle to start

Last month I installed Google's new Workspace CLI, `gws`. It's a command-line tool that wraps every Workspace API — Gmail, Calendar, Drive, Sheets — and ships with 95 ready-made "skills" that explain how to use each one. The pitch is that any AI coding agent that supports the Agent Skills standard can read these skills and operate Workspace on your behalf.

I asked Claude Code to fetch my most recent email. Simple task; the skill for it is sitting right there in `~/.claude/skills/gws-gmail/SKILL.md`, with the exact commands and recommended flags. What I expected was something like:

```
gws gmail users messages list --params '{"userId": "me", "maxResults": 1}'
```

What I got was three turns of `gws --help`, a detour through Claude Code's built-in Gmail connector (which I hadn't configured), a hallucinated `--last-email` flag that doesn't exist, and eventually an apologetic "I'm not sure how to do this — could you share the documentation?"

The skill was right there. The agent never read it.

This post is about why that happened, what I built to fix it, and — because the first thing I built didn't work — what I learned along the way that's generally useful. Some of it is about Claude Code specifically. Most of it applies to any agent that uses the new SKILL.md standard, which by now is roughly thirty agents and growing.

## What an Agent Skill actually is

If you haven't worked with skills before, the mental model is simple. A skill is a folder with a file called `SKILL.md` inside. The file starts with a few lines of YAML telling the agent the skill's name and a one-sentence description, then has Markdown explaining what the skill does, when to use it, and the exact commands or steps involved.

```yaml
---
name: gws-gmail
description: List, read, send, and search Gmail messages via the gws CLI.
---

## Usage

To list recent messages:
gws gmail users messages list --params '{"userId": "me", "maxResults": 5}'

To read a specific message efficiently, request the metadata format
(full bodies are often >100KB and pollute context):
gws gmail users messages get --params '{"userId": "me", "id": "<id>", "format": "metadata"}'
...
```

The agent doesn't read every skill on every turn — that would be impossibly expensive. Instead, when a Claude Code session starts, it scans your skills folder and loads only the *names and descriptions* into a section of its system prompt called `available_skills`. The agent uses this section as an index: when you ask for something Gmail-related, it scans the index, sees a skill called `gws-gmail` with a relevant description, decides to consult it, and reads the full file off disk. Only consulted skills load their SKILL.md file into context — and that one file brings every command, example, and edge case it documents along with it. The skill files the agent doesn't consult stay on disk.

That's a thoughtful design. Skills can be arbitrarily long; only the index has to fit in the context window. Anthropic [published the standard](https://claude.com/blog/equipping-agents-for-the-real-world-with-agent-skills) in October 2025, opened it as an interoperable format in December, and [around thirty agents](https://agentskills.io/home) have adopted it since: Claude Code, OpenAI Codex, Cursor, GitHub Copilot, Gemini CLI, JetBrains Junie, and so on. You can write a skill once and ship it to all of them.

The design has one assumption baked in: that you'll have a small enough number of skills for the index to comfortably fit. That's where the 95-skill problem lives.

## The arithmetic of the budget

Claude Code documents two limits for the `available_skills` section. Each skill description is truncated to 250 characters in the listing the agent sees — even if the source frontmatter is longer (the source cap is 1,536 characters of `description` + `when_to_use` combined). The section as a whole is capped at 1% of the context window, with a fallback floor of 8,000 characters. Both limits are documented; the budget figures are in [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills) and the 250-character display cap is referenced in [issue #40121](https://github.com/anthropics/claude-code/issues/40121). (You can raise the budget with the `SLASH_COMMAND_TOOL_CHAR_BUDGET` environment variable, but very few users do.)

If you do the division:

```
8,000 chars  ÷  250 chars per skill  =  32 skills
```

That's a hard ceiling at the standard 200K context window. With 32 skills, the budget is exactly full. With 33 skills, something has to give. Claude Code's behavior past the ceiling is to truncate the descriptions — chopping characters off each one until the listing fits.

A reader on a Max, Team, or Enterprise plan may wonder why I'm anchoring the math at 200K when their default context window is now 1M. Two reasons worth understanding before we go further. First, Anthropic charges a 2x surcharge on input tokens above 200K (1.5x on output), so a session that has grown to 400K costs roughly 5x per turn versus a fresh 80K one. Second, context-rot research from Chroma shows every frontier model degrades well before its maximum — measurable quality loss begins around 50K tokens in a 200K-capable model, because the model's attention spreads thinner across more context, so per-token focus drops as the window grows. The development community has converged on fresh sessions per task — `/clear` between tasks rather than letting context accumulate — as the way to keep cost and quality intact. The 1M ceiling exists; the workflow is 200K. And even when 1M is in play, the skill listing budget still scales as 1% of context per the [Claude Code skills docs](https://code.claude.com/docs/en/skills) — bigger, but bounded; the 250-char description cap still applies, and a heavy installation still hits truncation. Even if the budget were unlimited, dumping every installed skill into context every turn would still waste tokens and dilute the agent's attention across skills it doesn't need. The architecture is the bottleneck. The 32-skill ceiling at 200K is just where the bottleneck shows first.

A user with 95 skills doesn't get 95 full descriptions. Skill *names* are always preserved (per the docs), so they consume their share of the budget too — at roughly 10 characters per name, that's about 950 characters off the top, leaving ~7,050 characters for descriptions across 95 entries. That's around 74 characters per description — barely "fetch email" in length. The keywords the agent matches on are mostly gone.

This isn't theoretical. [Issue #13099](https://github.com/anthropics/claude-code/issues/13099) was opened in early 2026 by a user with 63 skills installed who saw only 42 of them appear in their session — the rest had been pushed out of the budget. (Current docs state that skill names are always preserved; the budget logic may have tightened since the issue was filed. Either way, descriptions for skills past the ceiling get truncated to the point of being unmatchable, so the practical effect is similar.) Independent research on the budget mechanics ([Pelykh's writeup](https://gist.github.com/alexey-pelykh/faa3c304f731d6a962efc5fa2a43abe1)) measured roughly 33% of skills hidden in large collections.

I wanted concrete numbers for `gws` specifically. Claude Code's IDE has a context inspector that shows exactly how much of your context window each component consumes. With `gws`'s 95 skills installed and nothing else, the `available_skills` section reports **2,007 tokens per turn**. That's 2,007 tokens out of 200,000, every turn, in every session — including the C++ projects where I'll never touch Gmail in my life. Across a typical workday — call it five projects, twenty turns each, fresh sessions per task — that's 200,000 tokens of pure overhead, which at Claude Opus pricing works out to about three dollars a day per developer, paid in exchange for skills the agent mostly can't see clearly anymore.

The architecture is the bottleneck, and an industry trend is making it hit sooner. Most coding agents are migrating away from MCP servers toward CLI tools wrapped in SKILL.md files, because CLI invocations are far cheaper in tokens — [Microsoft's Playwright README](https://github.com/microsoft/playwright-mcp) literally recommends it on those grounds, and [ScaleKit's benchmark](https://www.scalekit.com/blog/mcp-vs-cli-use) measured 4–32x token reductions across common GitHub tasks. That migration is the right call on its own terms. But it means the *unit* of skill installation is shifting from "I added one skill" to "I installed a CLI tool, which brought 95 skills with it." Google did exactly that with `gws`. The next big tool will do the same. The discovery ceiling that the standard's designers presumably expected to be hit by power users with hundreds of hand-curated skills is now hit by a single `npm install`. Every new skill that lands makes every existing skill description shorter, and shorter descriptions mean worse matching — which means more cases like my Gmail puzzle at the top of this post.

## Even when descriptions fit, agents skip skills anyway

The budget is one failure mode. There's another one that's just as common, and it shows up even when you have plenty of headroom in the index.

In January 2026, Vercel published [an evaluation](https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals) testing how well coding agents could solve Next.js 16 problems with various forms of documentation provided. Next.js 16 introduced new APIs that aren't in any model's training data, so the agent genuinely needs the docs to succeed. They tested four configurations ([AGENTS.md](https://agents.md/) is the cross-platform open standard for agent instructions — analogous to Claude Code's CLAUDE.md, but adopted across most agent tools and the same kind of always-loaded context):

| Configuration | Pass rate |
|---|---|
| No documentation at all | 53% |
| Skill installed (default discovery) | 53% |
| Skill installed + explicit "use this skill" instruction | 79% |
| Static `AGENTS.md` file always loaded into context | 100% |

The 53% number is the one to look at. Installing a skill changed nothing. The agent had access to the right documentation, but in 56% of evaluation runs it never *reached* for the skill — never decided the skill was relevant, never opened the file. Vercel's exact phrase: *"in 56% of evaluation cases, the agent never invoked the skill it needed."*

The agent's reasoning, when it skips, is roughly: *"I see a skill description that mentions Next.js. I already know Next.js. I don't need to look at it."* That's a sensible-sounding heuristic when the agent's training data is current. It's a disastrous heuristic when the agent's training data is six months old and the framework has moved. I've watched this pattern outside the Vercel benchmark, too: tell the agent in your prompt that you're using version X of a framework, and the suggestions still drift toward the version it was trained on. The 79% number above shows the bias is bounded but not erased by direct guidance — even an explicit "use this skill" instruction still loses about a fifth of cases. The 100% AGENTS.md result tells you why: the agent reliably uses new reality only when that reality is in front of it from turn one, formatted as part of the working environment rather than advice the model can weigh.

Now compose this with the budget. Past 32 skills, descriptions truncate. Of the ones the agent does see, around 56% are skipped anyway. A user with 100 skills has paid for 100, gets truncated descriptions on all of them, and on average uses about 25.

That's the puzzle from the top of this post. The skill was there. The description got truncated, was skipped, or both.

## The obvious first attempt: a hook

Claude Code is built to be extended. Its [hook system](https://code.claude.com/docs/en/hooks) lets you run a script on various lifecycle events, including one called `UserPromptSubmit`, which fires every time the user sends a message. The hook reads the prompt from standard input and can return a JSON object whose `additionalContext` field gets appended to the model's context for that turn.

The intent is clear: this is the place to inject knowledge per prompt. The whole problem above is that the agent doesn't always realize it should consult a skill. Fine — let's not let it decide. Let's run a search every time the user types something, and inject the relevant skill content directly. No more skipping.

The implementation is small. Index every skill section as a vector, store the index on disk, and at hook time:

1. Read the user's prompt off stdin.
2. Embed it into a vector using a small embedding model.
3. Search the index for the most similar skill sections.
4. Format the top results as text.
5. Return them as `additionalContext` in the hook's JSON output.

I'll explain the search side in more detail later, because there are some non-obvious choices that mattered. For now the relevant point is: the pipeline worked. End-to-end. When I sent the prompt "fetch my last email," the hook ran, the search returned exactly the section of the `gws-gmail` skill that I would have wanted, and the formatted text reached Claude Code's context. I could verify the daemon side two ways — the daemon's stderr log printed a one-line summary of every request as it came in, and `curl` against the daemon directly returned the same JSON the hook would have. To confirm the content actually reached the model's view I asked it directly: when I followed up with *"what context did you receive about `gws-gmail`?"*, the model quoted the injected text back at me, framing labels and all.

The agent ignored it.

It still ran `gws --help`. It still tried the built-in Gmail connector. It still hallucinated parameters. The injected text was sitting in its context, plain to see, and it might as well not have been there.

I tried the obvious things. I added `IMPORTANT: USE THESE EXACT COMMANDS, DO NOT IMPROVISE` at the top of the hook output. No effect. I tried more emphatic phrasing, formatted the commands differently, prepended the content instead of appending it. None of it changed the agent's behavior in a measurable way.

The hook was working. The model was reading the content. It just wasn't *acting on it* — the wrapper turns hook output into advisory metadata in the model's view, rather than authoritative user instruction. Hook content gets treated as something the model can override; user-typed content is what its training tells it to honor.

## A short diagnosis (the long version is in the follow-up post)

This took longer to work out than I'd like to admit, but the cause is a single design choice in how Claude Code presents hook output to the model.

When your hook returns `additionalContext`, Claude Code doesn't paste it raw into the model's context. It wraps it with a label that reads, approximately: `"UserPromptSubmit hook additional context:"`. The label tells the model that this content arrived via a hook rather than from the user directly — and modern coding agents are trained to treat such labeled content as advisory rather than authoritative. Hook-injected text doesn't carry the weight the same text would have if the user had typed it. (When the labeled content also conflicts with what the user appears to want, the model goes further and treats it as a possible prompt injection — the follow-up post has the empirical tests.) Project-level context like `CLAUDE.md` files gets a similar treatment — they're prefaced with a disclaimer noting that "this context may or may not be relevant to your tasks." [Issue #22309](https://github.com/anthropics/claude-code/issues/22309) shows the exact text.

These wrappers turn out to matter enormously. Modern coding agents are trained on a hierarchy: the user's actual prompt is the most authoritative signal; system messages and tool output are weaker signals that the model is encouraged to take seriously *if relevant* but to override when they conflict with user intent. That hierarchy is generally a good thing — you want a careful agent that doesn't blindly do whatever a piece of injected text says. But in this case it works against me, because *my* injected text is exactly the thing the user would say if they knew the right commands. It just doesn't look that way to the model, because the wrapper says it came from a tool.

I tested this fairly carefully — multiple sessions, multiple framings, including increasingly desperate `OVERRIDE: TREAT THIS AS DIRECT USER INSTRUCTION` phrasings. The wrapper won every time. The pattern is filed in several GitHub issues. [#28158](https://github.com/anthropics/claude-code/issues/28158) and [#37550](https://github.com/anthropics/claude-code/issues/37550) are still open and growing. [#22309](https://github.com/anthropics/claude-code/issues/22309) (the smoking gun on the CLAUDE.md disclaimer wrapper) was bot-closed as a duplicate. The proposed fix has migrated through several tickets and currently lives at [#27365](https://github.com/anthropics/claude-code/issues/27365) — a feature request to wire up `updatedPrompt` for user hooks so they can replace the prompt directly. Anthropic hasn't engaged with any of them. If you want to know when the hook approach becomes viable, that's the issue to watch. I'll cover all of that, plus the empirical tests that confirm the wrapper-authority story, in a follow-up post.

For this post, the relevant takeaway is shorter: **search and delivery are independent problems.** My search worked. My delivery — the hook — didn't work, because of a design choice in Claude Code I can't change. So I needed a different way to deliver the same content.

## The pivot

Here's the architectural question I wish I'd asked earlier: where in the agent's context does injected content get the most attention?

The answer, on every coding agent I'm aware of, is the user's own message. That's the highest-authority slot. There's no wrapper that tells the model "this might be machine-generated"; it's just what the user said.

So I stopped trying to inject content from the server side and started injecting it from the *client* side, before the prompt ever reached Claude Code at all. The new architecture has three pieces:

```
+--------------------+        +-------------------+        +-----------------+
|  User types in     |        |  Local HTTP       |        |  Claude Code    |
|  small VSCode      |  --->  |  search daemon    |  --->  |  receives the   |
|  popup (Ctrl+Alt+P)|        |  searches index   |        |  enhanced prompt|
+--------------------+        +-------------------+        +-----------------+
        |                                                           ^
        |     "fetch my last email"                                  |
        |     [search returns gws-gmail relevant section]            |
        |     [extension prepends section to user's text]            |
        |     [puts result on clipboard, focuses Claude Code]        |
        |     [user pastes — or extension auto-pastes]               |
        +-----------------------------------------------------------+
```

The user types their prompt into a small popup that the VSCode extension provides. The extension hits a local HTTP daemon (running on `127.0.0.1`, no network), which holds the skill index in memory. The daemon does the search and returns the relevant content. The extension prepends that content to the user's original text and delivers the combined string to Claude Code's input. From Claude Code's perspective, it just received a more detailed user prompt.

The search engine is exactly the same one I'd been using in the hook. Same embeddings, same index, same top-k, same formatter. The only thing that changed is *where* the result lands.

It worked.

I tested the same prompt — "fetch my last email" — across three fresh Claude Code sessions with the daemon enabled. In each session, the agent's first action was to call the correct `gws gmail users messages list` command, with the right parameters, on its first turn. It chose the `format: "metadata"` option (a few KB) instead of `format: "full"` (which had previously bloated the context with 145KB of message bodies). It handled Windows console encoding correctly without prompting. There was no `--help` digression and no built-in connector confusion.

The same content, in the user-message position, produced a completely different outcome from the same content in the hook-output position. Same model, same skill, same prompt — different slot, different behavior.

There's a follow-on once the daemon is working end-to-end. Since PreBrief now finds and injects the right content per prompt, the native skill listing in `~/.claude/skills/` isn't doing anything useful — the agent doesn't need it. Move the skills out of Claude Code's scan path (a disabled folder like `~/.claude/skills/.disabled/` works) and point PreBrief's index at the disabled folder instead. The 2,007-token-per-turn listing overhead disappears across every project. The skills are still searchable via PreBrief's index, the agent gets the right content when relevant, and you pay nothing when nothing matches. The native discovery system is off; the search-based one is on.

The same move changes what's worth installing in the first place. Today, every skill in `~/.claude/skills/` loads in every project — whether you'll touch it or not — and there's no built-in toggle to enable or disable individual skills per project or per task (open feature requests for finer-grained control: [#43928](https://github.com/anthropics/claude-code/issues/43928), [vercel-labs/skills #634](https://github.com/vercel-labs/skills/issues/634) — neither shipped). So most users self-censor what they install: every speculative skill costs context tokens and selection-attention budget on every turn forever, so you only install what you're sure you'll use often. With PreBrief, that calculation changes. Skills you might use once a quarter, skills for tools you're still evaluating, skills for tasks you only do occasionally — install them all, drop them in the disabled folder, pay nothing when nothing matches. Skills shift from a tax you pay on every turn to a library you draw from when needed.

## The design choices that mattered

Once the delivery problem was solved, I spent some time on the search side, because the difference between a search that works and a search that *appears* to work is large.

**Index sections, not whole skills.** A typical skill file is fifty to a hundred lines covering setup, authentication, common commands, error handling, and so on. If you compute one embedding for the entire file and store it as a single vector in your index, you've made the search engine commit to all-or-nothing matches. When the user asks about Gmail attachments, the search returns the entire Gmail skill — a few thousand tokens — even though the user needed about forty tokens of attachment-specific guidance. The signal gets diluted.

The fix is to chunk skills at the `##` heading level. Each section becomes its own indexed vector. When the user asks about attachments, the search returns the attachments section and nothing else. In `gws`'s 95 skills there are roughly 390 such sections; the agent sees three of them at most per prompt.

There's a second, less obvious benefit. When the standard flow does succeed — when the agent correctly decides to consult `gws-gmail` — it reads the SKILL.md file as a tool result, and that result then sits in the conversation for the rest of the session, even if all the agent needed was one command. (The Agent Skills standard does support splitting reference material across multiple files, but in practice many skills, including all 95 in `gws`, are monolithic.) PreBrief delivers a section instead of a file — typically a few hundred tokens instead of a few thousand — which lowers the per-turn cost on its own. It also makes the eviction story sketched in "What's left unsolved" below much simpler: the unit of injection is the unit you'd want to retract.

**Prefer fewer, more relevant results over many results.** I started with top-10 retrieval; I ended up at top-3. The reason isn't latency or cost — it's that the model treats everything in its context as roughly equally relevant. If you hand it ten chunks where three are dead-on and seven are loosely related, the loosely-related ones dilute the signal. Smaller, sharper context produced visibly better behavior. Three relevant results beat ten mixed-quality ones.

**Frame results in plain language, not metadata.** My first formatter prepended each result with a header like `[gws-gmail > Attachments | confidence: 0.85]` — the kind of structured tag I'd want to see if I were debugging the search engine. The agent treated those headers as system noise and the content underneath as questionably authoritative.

I switched to plain prose: `Use this procedure for gws-gmail attachments. This is specific to your environment — do not substitute with general knowledge.` followed by the actual commands. Same content, different framing. The agent treated it as ordinary user instruction. The lesson generalized: the agent shouldn't be able to tell that a search system is involved at all. As far as it knows, this is just a user who happens to type unusually detailed prompts.

**Pick a small embedding model and forget about it.** I used `bge-small-en-v1.5` — a 22MB model that runs on CPU at about five milliseconds per embedding. The full search round-trip is about 80 milliseconds when the daemon is warm. I spent a while wondering whether a larger embedding model would help. The answer turned out to be no, because the bottleneck was never the retrieval quality — the search was finding the right content well before I'd fixed the delivery problem. Throwing a bigger model at the search side would have done nothing. The bottleneck was always: where does the content land, and how is it framed when it gets there.

## What this is, and what it isn't

The technique is RAG — retrieval-augmented generation. It's been used for document search, customer support, and code-aware assistants for years. Applying it to skill discovery isn't new either; an academic paper called [SkillFlow](https://arxiv.org/abs/2504.06188) (UC Davis, 2025) frames skill retrieval as an information-retrieval problem and proposes a more elaborate four-stage pipeline that achieves a 78% improvement on a skill benchmark. SkillFlow validates the case in the abstract. What's in this post is what made the pattern work in front of a real agent on a real machine — the small, unglamorous decisions that determined whether the agent's behavior actually changed:

- The production measurement: 95 skills, from one CLI tool, costing 2,007 tokens per turn across every project, regardless of relevance.
- The failure of the obvious built-in approach (the hook) for reasons that aren't obvious until you stare at the wrapper.
- The realization that search and delivery are independent problems, and that getting the delivery into the user-message slot is the unglamorous half of the work.
- The set of small design decisions on the search side — section chunking, top-3, plain-language framing — that made the retrieval feel invisible to the agent.

I built this as a small open-source project called **PreBrief**. It's deliberately minimal: a Python daemon, a FAISS index, around 600 lines of search code, and a thin VSCode extension. Most of the interesting work is in `formatter.py`, where the language and framing of the injection lives. The repo is [github.com/cdelgado70/PreBrief](https://github.com/cdelgado70/PreBrief). I'd rather have a working example out there than another whitepaper, so the pattern is small enough to copy and adapt.

If you want to try the pattern on a different agent — Cursor, Codex CLI, Gemini CLI — the daemon is agent-agnostic. It speaks HTTP. Anything that can shell out before sending a prompt can use it.

And the index doesn't have to be skills. Anything chunked and searchable can go in — personal notes, team conventions, API references, runbooks, design docs. Skills happen to be the trigger because their discovery ceiling makes the cost visible, but the mechanism doesn't care what's in the index. If you have a knowledge base you wish your agent reached for and it doesn't, this is roughly what fixing it looks like.

## What's left unsolved

Two things, both worth their own posts.

The first is the hook story. There's a real question about why Claude Code wraps hook output the way it does, what the open issues say about it, and what would actually fix it — the leading proposal lives at [issue #27365](https://github.com/anthropics/claude-code/issues/27365). That's the follow-up post.

The second is what happens to your context window over a long session. Pre-prompt injection solves the cold-start problem — the agent has the right knowledge from turn 1, which is exactly what makes the fresh-session workflow practical: no learning tax on the first few turns of a new task. But it doesn't solve the runtime problem: the skill section the daemon injected on turn 3 is still sitting in context on turn 30, the 50KB of `--help` output the agent fetched on turn 7 is still there too, and so is everything else that's accumulated. Nothing is *deciding* to keep that material — it's the default. Tokens accumulate; nothing leaves until compaction fires (lossy and opaque) or the user starts a fresh session. Fresh sessions are the right answer at task boundaries — and per the earlier section, they're the practical workflow. Within a session, though, the user has no graceful way to drop the parts they're done with.

The interesting question is whether the user gets any say in this *within* a task. Today, on Claude Code, mostly no. Fresh sessions and `/clear` are the right tool at task boundaries, but mid-task the only options are `/compact` (lossy, summarizes everything) or pressing on. There's no granular control. The open-source side of the ecosystem is further along: [Aider](https://aider.chat/docs/usage/commands.html) has a `/drop <file>` command for removing specific files from the chat context, [Cline](https://docs.cline.bot/model-config/context-windows) supports rule-based context handoff via `.clinerules` (e.g., "if context exceeds 50%, hand off to a new task with this summary"), and [OpenCode](https://opencode.ai/docs/commands/) makes its compaction behavior configurable. [OpenDev](https://github.com/opendev-to/opendev) is built around the principle that every system action — tool calls, safety vetoes, context compaction, memory updates — should be observable and overridable by the developer. None of these yet ship the specific capability the daemon I described would benefit from — surgical, external-tool-mediated eviction of arbitrary regions, with confirmation prompts before anything destructive — but the direction is right and the demand is documented. [Aider issue #3607](https://github.com/Aider-AI/aider/issues/3607) ("More control over chat history") and [Claude Code issue #34872](https://github.com/anthropics/claude-code/issues/34872) ("Hook write access to conversation history for in-session context eviction") are both open with no shipped solution.

The context window is yours. The tokens are paid for by you. Compaction is fine as a default for users who want one. Users who'd rather have surgical, transparent control over what's resident don't currently have a path on any major agent — but the path is being built, slowly, in the open. Open-source agents will probably ship something here before the commercial ones do; the alignment of incentives is just better. If you'd find this useful, the issues above are the ones to push on.

For now: if your agent is fumbling work it should be able to do, and you've installed a few hundred skills hoping it would just figure things out, the answer is probably not more skills. It's a search engine in front of the skills you already have, and a place to put the results where the agent will actually act on them.

---

*PreBrief is open source on [GitHub](https://github.com/cdelgado70/PreBrief). Technical corrections, issues, and PRs are welcome — particularly if your agent of choice has framing behavior that differs from what I've described here.*

*If you're shipping a coding-agent product and the discovery ceiling is biting in production, or you're working through similar context-engineering problems on your own platform, I'd be glad to compare notes. Reach me at cdelgado70@gmail.com.*

---

*C. Delgado works on AI coding-agent infrastructure and context engineering. Other writing and projects: [github.com/cdelgado70](https://github.com/cdelgado70).*
