# The Case for Slash Commands in Our AI Chat App

**A proposal for review**

---

## TL;DR

We are building a domain AI chatbot, and we are about to inherit a problem that every team in this space has hit: **users will phrase the same task fifty different ways, the model will respond fifty different ways, and we will have no reliable way to evaluate, improve, or trust any of it.**

Slash commands — small, named shortcuts like `/case-summary` or `/grill-me` that users invoke from the chat input — are the cheapest, highest-leverage fix for this. They convert open-ended natural-language prompts into structured, named, version-controlled units of work. This unlocks reliability for users, productivity for AI developers, and measurable evaluation for the team.

This document explains the pain, the proposal, the evidence, and what it means in practice.

A working prototype demo is already built and recorded. It uses **LangGraph** as the agent runtime and **Chainlit** as the chat frontend.

📺 **Demo video:** [Case Assistant: Slash Commands](https://www.youtube.com/watch?v=x_Q5UvOqcLc) — published 6 May 2026.

---

## 1. The pain we are walking into

### 1.1 Free-form prompts produce inconsistent output — and this is now well-documented

The single biggest problem in production LLM chat apps right now is reliability. The most comprehensive recent study of production AI agents — Pan et al., "Measuring Agents in Production" (arXiv 2512.04123, December 2025) — surveyed **306 practitioners** and conducted 20 in-depth case studies across 26 industry domains. Their headline finding: **"Reliability remains the top development challenge, driven by difficulties in ensuring and evaluating agent correctness."**

The same study showed teams responding to this problem by *deliberately scaling back ambition*: 70% rely on prompting off-the-shelf models (no fine-tuning), 74% depend primarily on human evaluation, and **68% of agents execute at most 10 steps before requiring human intervention.**

Why is reliability so hard? Because the same prompt does not produce the same answer. Recent academic work has now quantified this:

- Cui & Alexander, "Same Prompt, Different Outcomes" (arXiv 2602.14349, February 2026), ran **480 controlled LLM data-analysis attempts** across 6 models, 2 prompting strategies, and 4 temperature settings. The conclusion: "considerable variation in the analytical results even for consistent configurations." Same prompt, same model, same task, even at temperature zero — different conclusions.
- Potamitis, Klein & Arora, "ReasonBENCH" (arXiv 2512.07795, December 2025), ran 10 independent trials across many model–algorithm–task combinations and found that "even strategies with similar average performance can display **confidence intervals up to four times wider**" than each other. Average accuracy hides the truth; variance reveals it.
- Polo et al., "Efficient multi-prompt evaluation of LLMs" (PromptEval, arXiv 2405.17202, NeurIPS 2024), explicitly states the case for evaluating across many prompt templates rather than one: prompt sensitivity is severe enough that "popular benchmarks for comparing LLMs rely on a limited set of prompt templates, which may not fully capture the LLMs' abilities and can affect the reproducibility of results."

The implication is straightforward. **If users are typing free-form prompts, we cannot promise consistency. We cannot evaluate quality. We cannot improve systematically. We are flying blind.**

### 1.2 Prompts are ad-hoc, and this hurts reliability and reuse

A peer-reviewed paper accepted to ASE 2025 — Li, Sergeyuk & Izadi, "Prompt-with-Me" (arXiv 2509.17096) — frames the industrial problem precisely: *"prompt management in practice remains ad hoc, hindering reliability, reuse, and integration into industrial workflows."* Their analysis of 1,108 real-world software-engineering prompts confirmed that prompts are "typically written ad hoc, often with fragmented grammar or inconsistent wording, even though minor phrasing changes can significantly alter model behavior."

The same paper argues that prompts must be treated as "first-class software artifacts" — versionable, testable, reusable — for the LLM-assisted workflow to scale beyond individual users.

### 1.3 The 95% failure narrative — context, not gospel

The MIT NANDA Initiative's 2025 report "The GenAI Divide: State of AI in Business" surveyed 153 senior leaders, conducted 52 interviews, and analysed over 300 enterprise AI initiatives. Its widely-cited claim that **95% of enterprise GenAI projects fail to deliver measurable business impact** has been quoted everywhere from Fortune to Healthcare IT News.

It's worth being honest that the methodology of that report has been [contested](https://www.futuriom.com/articles/news/why-we-dont-believe-mit-nandas-werid-ai-study/2025/08) — the underlying data is hard to find, and NANDA itself has an interest in promoting agent-based architectures. But even sceptical analyses agree the *direction* is right: enterprise IT projects in general fail at high rates, and AI projects are not magic immune. The NANDA report's most useful finding for our purposes is its diagnosis of *why* projects fail. They identify a "learning gap" — organisations not understanding how to embed LLMs in workflows that capture benefits while minimising downside risks. That is exactly the gap slash commands are designed to close.

---

## 2. The proposal

Adopt **slash commands** as a first-class feature of our chat app. Specifically:

1. Build a **command registry** — a versioned set of named, parameterised commands (e.g., `/case-summary`, `/grill-me`, `/draft-letter`, `/timeline`, `/redflags`) backed by structured prompt templates stored as files in our repo.
2. Expose them through a **slash menu in the chat input** with autocomplete, fuzzy filtering, keyboard navigation, and inline descriptions — the standard UX pattern that Slack, Discord, Notion, Cursor, and Claude Code all use.
3. Treat each command as a **first-class evaluable unit** — with its own tests, golden examples, version history, owner, and metrics.
4. Keep **free-form chat fully working** alongside commands. Users who don't know what they want still type naturally; users who know what they want get the fast path.

This is not a moonshot. The frontend is a thin layer of UI. The backend is a registry pattern that's well-trodden — Anthropic's Claude Code, OpenAI's Codex CLI, Cursor, and Notion AI all ship variants of it.

### 2.1 The working prototype

A demo is already built and runs the full flow end-to-end. The video above shows it in action. The stack:

- **LangGraph** provides the agent runtime, with each slash command routed to a node (or tool) backed by a structured prompt template.
- **Chainlit** provides the chat frontend, including the slash command UI, autocomplete dropdown, and the streaming response surface.

This is exactly the architecture proposed in §2 — built, working, recorded. The demo shows the autocomplete dropdown, command execution with arguments, the multi-turn `/grill-me` interview flow, and natural-language fallback all coexisting in one interface.

---

## 3. Why this helps each group

### 3.1 For users: less guessing, faster results, more trust

Slash commands convert "I have to figure out how to ask this thing" into "I pick from a menu of things it can do." This is the same shift that command palettes brought to text editors and that `/` shortcuts brought to Notion and Slack.

Concretely:

- **Discoverability.** Users learn what the app can do by typing `/`. No onboarding doc to read. The menu is the documentation.
- **Predictability.** A user who runs `/case-summary` knows what shape of output to expect. This builds trust, which is the most important currency in an AI product.
- **Lower cognitive load.** Users do not have to be prompt engineers. They pick a command and the prompt engineering is done for them, by us, once, well.
- **Consistent quality across users.** Two different users running the same command get a comparably-structured answer. With free-form prompts, output quality varies wildly with prompting skill — a tax that falls on every user, every interaction.

### 3.2 For AI developers: prompts become testable software

This is the most underrated benefit. The Prompt-with-Me paper makes the case directly: ad-hoc prompts hinder reliability and reuse, and treating prompts as first-class artifacts (versioned, named, structured) is the path to scaling beyond individual users. Slash commands are the user-facing surface of exactly that pattern.

This unlocks the engineering toolchain:

- **Version control.** Each command lives in a markdown file in the repo. Changes go through PR review. We can `git blame` a regression.
- **A/B testing.** Want to know if `/case-summary v2` is better than `v1`? Ship both behind a flag, route traffic, measure. This is essentially impossible with free-form prompts because there is no stable identifier for "the prompt."
- **Regression testing in CI.** Open-source tools like Promptfoo run automated prompt evaluations against golden sets, comparing models side-by-side and catching regressions before production. This only works if prompts are named units.
- **Model portability.** When the next model drops (and it will), we can re-evaluate every command against the new model in one batch run, instead of guessing whether free-form chat got better or worse.
- **Reusable across surfaces.** The same `/case-summary` command can be invoked from chat, from a button in the case sidebar, from a Slack bot, or from a scheduled job. Free-form prompts cannot be reused this way.

### 3.3 For evaluation and quality: from vibes to numbers

This is the most important point for anyone worried about whether the product actually works. Today, a free-form chatbot has effectively one quality metric: "did the user thumbs-up it?" That is not enough.

With slash commands, every command becomes a measurable unit:

- **Per-command accuracy.** We hold a golden set of inputs and outputs for `/case-summary` and run them on every release. Score against expert review. Track over time.
- **Per-command latency and cost.** Token usage per command becomes a first-class number we can chart, budget, and optimise. With free-form chat, every interaction blurs into a single average.
- **Per-command adoption.** Which commands actually get used? Which get used once and abandoned? This tells us where the product is winning and where it isn't — feedback no free-form chatbot can produce.
- **Variance reduction.** ReasonBENCH and the Cui-Alexander work both recommend running each prompt configuration multiple times and reporting variance, not just mean. With named commands, this is straightforward. With free-form chat, you cannot even define "the same prompt."
- **Repeatable benchmarks.** When someone asks "is the AI getting better or worse?" — we will have a real answer, per command, with a number, over time. Without commands, we will have hand-waving.

### 3.4 For the business: faster onboarding, defensibility, lower compliance risk

- **Customer onboarding.** A new user who sees `/case-summary`, `/grill-me`, `/draft-letter`, `/redflags`, `/timeline` instantly understands the product. Compare that to "you can ask me anything about your case" — which sounds powerful but in practice produces blank-page paralysis.
- **Sales demos.** Slash commands demo well. They show, not tell, what the product does. We know exactly which buttons to press to make the buyer say wow. The video linked above is itself an example of this.
- **Compliance and audit.** In legal, medical, or financial contexts, "we ran the `/case-summary` command, version 1.4.2, on these documents at this timestamp, with this output" is auditable. "The user asked the chatbot a thing and the chatbot said another thing" is not.
- **Defensibility.** Anyone can wrap an LLM in a chat UI. A curated, evaluated, domain-specific command library *is* the moat. It is the embodied expertise of our team, and it compounds over time.
- **Knowledge retention.** When someone on the team figures out the perfect way to elicit a case summary from the model, that prompt becomes `/case-summary v3` in the repo. It does not walk out the door when they leave.

---

## 4. SME-Driven Command Development: The Key to Reliability

### 4.1 Why Subject Matter Experts are non-negotiable

The most dangerous trap in building domain AI tools is letting software developers guess what the domain needs. We've seen this movie before: well-meaning engineers build "smart" features that miss 80% of the actual workflow because they never watched an expert work.

In tax case analysis, this risk is amplified. A tax lawyer's mental model — what they look for first, what they flag as risky, how they structure an argument — is the product of years of training and thousands of cases. That expertise does not transfer through a few interviews or by reading documentation.

**The pattern that works:**

1. **Shadow real experts**. Watch a subject matter expert work through 5-10 actual cases. Take notes on every step, every heuristic, every cross-check.
2. **Extract the repeatable workflow**. Tax analysis isn't random. It has stages: document intake → fact extraction → issue identification → research → argument drafting. Each stage has questions that get asked, patterns that get recognized.
3. **Build commands that mirror the workflow**. Each slash command maps to a stage or decision point. `/extract-facts` isn't a "cool feature" — it's stage 2 of the workflow, externalized.
4. **Parameterize the judgment calls**. Experts don't work blind. They have preferences ("focus on deductions", "highlight Schedule 3 issues", "what's the ATO's likely position?"). Those become command parameters, surfaced in the UI.

### 4.2 Learning from solved cases: The command discovery process

Every organization has a graveyard of solved cases — the examples that experts point to when training juniors. These are the single best source of truth for what good looks like. They are also the single best source of truth for what commands we should build.

**The discovery process:**

1. **Collect exemplars**. Get 5-10 cases that the organization considers "done well" — clear outcomes, defensible positions, minimal rework.
2. **Reverse-engineer the workflow**. For each exemplar, ask: what questions were asked? What documents were consulted? What order were things done in? What was the output format?
3. **Identify the invariant steps**. Across all exemplars, what steps appear every time? Those are command candidates. `/identify-issues` isn't speculative — it's the thing the expert did in case A, case B, case C, and case D.
4. **Extract the parameters**. Where did the expert make a choice that changed the shape of the work? "This case needed deep research on Section 6-5" → `/search-docs` with scope="legislation". "This client cares about audit risk" → `/identify-issues` with priority filter="audit".

### 4.3 Prepackaged commands + chat parameters: The reliability formula

The big insight from working with SMEs and studying solved cases is that **most domain work is not free-form**. It follows patterns. Patterns can be encoded. Encoding patterns as commands — then letting users refine with natural language — is how we get both reliability and flexibility.

**The formula:**

```
/command_name + optional_user_refinement = structured_output
```

- `/summarise_case` → produces structured case overview
- `/summarise_case "focus on deductions"` → same structure, deductions emphasized
- `/identify_issues` → produces prioritized issue list
- `/identify_issues "high priority only, audit risk perspective"` → filtered by risk, audit lens

The command guarantees structure, coverage, and minimum quality. The refinement tailors the output to the specific situation. This is dramatically more reliable than free-form prompting, where every interaction is a fresh roll of the dice.

### 4.4 Reducing noise, boosting trust

Free-form AI chat produces noise in three ways:

1. **Output noise**. The model hallucinates, meanders, or misses the point because the prompt wasn't clear.
2. **Process noise**. The user doesn't know what to expect, so they can't tell if the output is complete.
3. **Evaluation noise**. We can't tell if the product is getting better because we can't even define what "better" means.

Commands eliminate all three:

1. **Output noise** → reduced because prompts are engineered, tested, and versioned. The same command produces the same shape of output every time.
2. **Process noise** → eliminated because the command name describes what will happen. The user knows what `/summarise_case` will produce before they run it.
3. **Evaluation noise** → eliminated because each command has a clear success criterion. We can test `summarise_case.md` against a golden set of cases and measure whether it captured the key facts.

Trust comes from predictability. Users trust tools that do what they say, in the way they expect, every time. Commands are how we deliver that.

---

## 5. What changes for the team

This is not a rewrite. It is an addition. Concretely:

| Area | Today | With slash commands |
|---|---|---|
| Frontend | Free-form chat input | Free-form input + `/` triggers an autocomplete menu |
| Prompts | Ad-hoc, in code or in heads | Named markdown files in `commands/` directory |
| Evaluation | "Did the user like it?" | Per-command golden tests in CI + production telemetry |
| Iteration | Edit prompt, hope for the best | PR review, versioned, A/B testable |
| Docs | "Here's a chatbot, ask it anything" | `/help` lists everything; menu is self-documenting |
| Onboarding | Tell users to "explore" | Users see the command menu and learn by browsing |

Free-form chat does not go away. It remains the fallback for everything that doesn't fit a command. Users who don't know what they want still get a useful answer. The two modes coexist — exactly as they do in Claude Code, Cursor, and Notion AI, and as shown in the demo video.

---

## 6. What this looks like in our product

To make this concrete, the same case-summary task today versus with commands:

**Today (free-form, no commands).** User types: *"Can you give me a summary of the Chen case? Just the highlights, nothing too long."* Model produces a reasonable but unstructured response. Different user types: *"Summarise CV-2026-0184 for me, focus on what's strong and what's weak."* Different output shape, different sections, different tone. There is no way to know if either output was good. There is no way to make tomorrow's outputs better than today's.

**With slash commands.** User types `/case-summary`. Menu shows the command, description, and what it produces. User hits enter. Backend loads `commands/case-summary.md` (a versioned prompt with sections for posture, key facts, strongest argument, weakest point, suggested next step). LLM produces output in the structured shape. Output is consistent across users. We can test it. We can improve it. We can prove it works.

Same task. Same model. Different infrastructure underneath. The user doesn't see the difference — until they realise the answer they get on Tuesday is the same shape as the answer they got on Monday, and they actually trust the thing.

---

## 7. Risks and how we handle them

| Risk | Mitigation |
|---|---|
| Users don't discover slash commands | Placeholder text in input prompts users; `/help` always available; first-run tooltip; the dropdown auto-appears on `/` |
| Command library bloats over time | Quarterly audit; usage telemetry kills commands no-one runs; ownership assigned per command |
| Commands become rigid, can't handle edge cases | Free-form chat remains a first-class fallback; commands accept arguments for flexibility |
| Naming collisions, confusing names | Style guide for command names; namespacing by category; review in PRs |
| Mobile UX (no easy `/` key) | Quick-action buttons above the input on mobile; the dropdown still works on tap |
| Effort to build | Demo already built and working in LangGraph + Chainlit. Backend is a thin router. Nothing on the critical path. |

---

## 8. The evidence base, with verified sources

For anyone who wants to read the primary sources directly:

- **Reliability is the #1 production challenge in real deployed agents.**
  Pan et al., *Measuring Agents in Production*, arXiv [2512.04123](https://arxiv.org/abs/2512.04123), Dec 2025. Survey of 306 practitioners across 26 domains.
- **LLMs are non-deterministic, even with same inputs.**
  Cui & Alexander, *Same Prompt, Different Outcomes: Evaluating the Reproducibility of Data Analysis by LLMs*, arXiv [2602.14349](https://arxiv.org/abs/2602.14349), Feb 2026. 480 controlled attempts across 48 configurations.
- **Variance hides under average accuracy; we should report confidence intervals.**
  Potamitis, Klein & Arora, *ReasonBENCH: Benchmarking the (In)Stability of LLM Reasoning*, arXiv [2512.07795](https://arxiv.org/abs/2512.07795), Dec 2025.
- **Prompt sensitivity is severe enough to materially affect benchmark reproducibility.**
  Polo et al., *Efficient multi-prompt evaluation of LLMs* (PromptEval), arXiv [2405.17202](https://arxiv.org/abs/2405.17202), NeurIPS 2024.
- **Ad-hoc prompts hinder reliability and reuse; structured management is the fix.**
  Li, Sergeyuk & Izadi, *Prompt-with-Me: in-IDE Structured Prompt Management for LLM-Driven Software Engineering*, arXiv [2509.17096](https://arxiv.org/abs/2509.17096), accepted to ASE 2025 (Industry track).
- **95% of enterprise GenAI projects fail to deliver measurable impact.**
  MIT NANDA Initiative, *The GenAI Divide: State of AI in Business 2025*. Widely cited; methodology has been [contested](https://www.futuriom.com/articles/news/why-we-dont-believe-mit-nandas-werid-ai-study/2025/08), so treat as directional rather than precise.
- **Slash commands are now standard in AI tooling.** Verifiable from public docs of: Claude Code (Anthropic), Codex CLI (OpenAI), Cursor, Notion AI, Slack, Discord, Microsoft Teams.

A note on what I deliberately *did not* cite: a number of vendor blog posts circulate impressive-sounding statistics — "65% faster prompt development", "30% productivity gain from version control", "87% reduction in harmful outputs" — but tracing them to primary sources, they all originate from prompt-management-tool marketing copy with no published methodology. Those numbers are excluded from this document on purpose. The case for slash commands stands on the peer-reviewed evidence above.

---

## 9. Recommendation

Adopt slash commands as a feature of v1. The prototype is already built — LangGraph + Chainlit, working end-to-end, demo recorded. Remaining work is productising the prototype, defining the initial set of five to seven commands, and wiring telemetry from day one so we can measure adoption and quality from the moment we ship.

The downside of doing this is small: a thin layer of UI and a directory of markdown files. The downside of *not* doing this is that we will spend the next year debugging an inconsistent free-form chatbot that nobody can reliably evaluate, while teams that did adopt structured commands move faster than we do.

The published evidence is converging on the same conclusion from multiple angles — production teams say reliability is the #1 challenge, academic studies confirm prompts are non-deterministic and ad-hoc management hurts reuse, and every major AI tool that has shipped recently has structured commands as a first-class feature. We should follow that evidence.

---

*Questions, pushback, and counter-proposals welcome.*