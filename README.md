# Agent Skills for Developer Research

Six Claude Code skills that turn open-ended research questions a developer actually asks — *what pays for open source work in Rust?*, *what is this company building?*, *has someone already made this?* — into structured reports, by fanning browser agents across the sources in parallel.

Each skill is a single `SKILL.md`: a description that determines when it fires, and instructions for orchestrating [TinyFish](https://tinyfish.ai) CLI agents and synthesising what comes back.

---

## Why these exist

The research they automate is all the same shape and all miserable by hand: the answer is spread across four to eight sites, none of which have APIs, each of which needs a different query, and the synthesis only becomes useful once every source is in.

Doing it manually takes an afternoon and gets done once. These make it a single question, so it gets done every time it would actually be useful — before starting a project, before applying somewhere, before writing a proposal.

---

## The skills

| Skill | Question it answers | Sources |
| --- | --- | --- |
| **oss-bounty-finder** | How do I get paid for open-source work in *stack*? | Algora, IssueHunt, awesome-list repos, NLNet, Sovereign Tech Fund, Mozilla MOSS, LFX Mentorship, GSoC, Outreachy |
| **company-hiring-intelligence** | What is this company actually building? | Job postings, careers page, LinkedIn Jobs, engineering blog |
| **job-market-intel** | What does *role + stack* pay, and who's hiring? | LinkedIn, Indeed, Glassdoor |
| **project-idea-validator** | Has this been built, and is the space crowded? | GitHub, Dev.to |
| **academic-research-mapper** | What's published here, and where are the gaps? | arXiv, Semantic Scholar, Google Scholar |
| **cfp-hunter** | Which conferences are open for submissions? | developers.events, cfp.watch, Confs.tech |

Two of these read a market from indirect evidence rather than asking it directly. `company-hiring-intelligence` infers strategy from what a company is hiring for — twelve infra roles and no design roles says more about the next year than the careers page copy does. `oss-bounty-finder` treats funding as three distinct tiers (bounty platforms, bounty-labelled issues in the wild, and grant foundations) because they pay differently and want different things.

---

## Design

### Parallel fan-out, single synthesis

Every skill follows the same shape:

```
question
   │
   ├── agent → source A ─┐
   ├── agent → source B ─┤
   ├── agent → source C ─┼── dedupe ── synthesise ── structured report
   └── agent → source D ─┘
```

Agents run concurrently because they are independent and each takes tens of seconds; serialising six of them turns a 40-second answer into four minutes. Deduplication happens before synthesis because the same bounty or paper legitimately appears on several sources, and a report that lists it three times reads as noise.

`oss-bounty-finder` adds a tier structure on top of the fan-out: platform scrapes, then repo-by-repo bounty-label checks, then grant foundations — each tier a different kind of source needing a different extraction strategy.

### The description field is the interface

A skill only helps if it fires at the right moment, which makes the `description` in the frontmatter the most important line in the file — it is what the model matches a user's phrasing against, not the body.

Each description therefore enumerates the real phrasings people use rather than summarising the skill abstractly: *"find me paid open source work"*, *"are there any open source grants I can apply to"*, *"what repos are paying for contributions"*. Describing `oss-bounty-finder` as "finds OSS funding opportunities" is accurate and fires far less often, because users don't talk that way.

### Pre-flight checks, cross-shell

Every skill verifies the TinyFish CLI is installed and authenticated before making any call, with both PowerShell and bash variants, because the failure otherwise surfaces halfway through a fan-out as an opaque error from one of six agents — much harder to diagnose than a check that fails immediately and says what's missing.

---

## Installing

Clone into your Claude Code skills directory, or copy individual skill folders:

```bash
git clone https://github.com/tanmaysainighy/skills.git
```

**Requires** the TinyFish CLI, installed and authenticated:

```bash
tinyfish --version    # verified by every skill before it runs
```

Invoke by asking for what you want. The descriptions are written so that natural phrasing triggers the right skill — *"find me paid open source work in Rust"*, *"what is Anthropic hiring for"*, *"has anyone built a RAG evaluation harness"*.

---

## Repository layout

```
skills/
├── oss-bounty-finder/SKILL.md
├── company-hiring-intelligence/SKILL.md
├── job-market-intel/SKILL.md
├── project-idea-validator/
│   ├── SKILL.md
│   └── README.md              # this one has its own write-up ("IdeaProbe")
├── academic-research-mapper/SKILL.md
└── cfp-hunter/SKILL.md
```

---

## Known limitations

- **No trigger-accuracy testing.** Whether each description fires on the phrasings it lists, and whether any two skills compete for the same request, has never been measured. Writing a set of test prompts per skill and recording which fires is the most valuable next step.
- **Scraped sources break silently.** These target sites with no APIs. A layout change degrades one source's results to empty without failing the run, and the report looks complete.
- **Output quality is unvalidated.** No comparison has been made between a skill's report and the same research done by hand, so accuracy and recall are unknown.
- **Agent runs cost credits**, and a fan-out across eight sources costs eight runs. None of the skills warn about this before starting.
- **Uneven depth.** `project-idea-validator` covers two sources; `oss-bounty-finder` covers nine across three tiers. The thin ones are thin because two sources were enough to be useful, not because the space is small.
