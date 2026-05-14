> **This skill has moved.** VestMap's skills are now distributed together as a single
> Claude Code plugin from the **[VestMap marketplace](https://github.com/VestMap-App/marketplace)**.
> Install both the `vestmap` and `vestmap-om-pages` skills with two commands inside
> Claude Code:
>
> ```
> /plugin marketplace add VestMap-App/marketplace
> /plugin install vestmap@vestmap-app
> ```
>
> This repository is kept as a reference archive. For full, step-by-step setup
> instructions, see the
> [marketplace README](https://github.com/VestMap-App/marketplace#readme).

---

# VestMap Skill

A Claude Code skill for answering any question about US property demographics, income, housing, workforce, crime, schools, natural hazards, and market trends — grounded in live data from the VestMap MCP server.

## What it does

Ask Claude anything about a US address and this skill tells it how to query the right VestMap data at the right geographic scale (Block Group, Tract, ZIP, City, County, CBSA, State) and how to present the answer.

Covers:

- **Demographics** — Population, households, age, net worth at every scale
- **Income & wealth** — Median household income, distribution, ESRI wealth index
- **Housing** — Median home value, rent, vacancy, tenure, FHFA House Price Index
- **Employment & workforce** — Occupations, industries, unemployment, commute
- **Education** — Attainment distribution and nearby school ratings
- **Crime** — AGS Crime Indexes (total, personal, property, and sub-categories)
- **Natural hazards** — FEMA National Risk Index across 19 hazard types
- **Lifestyle segmentation** — ESRI Tapestry segments
- **Market trends** — Growth projections and historical change
- **Business activity** — CBSA-level business and employee counts

All data comes from ESRI, AGS, FHFA, FEMA, and the US Census via live VestMap queries. 
## Requirements

- [Claude Code](https://claude.ai/claude-code)
- A VestMap account — [sign up here](https://app.vestmap.com/mcp)
- VestMap MCP server connected in your Claude Code settings

## Usage

```
/vestmap
```

Then ask Claude anything about a US address — or just ask a VestMap-style question and the skill triggers automatically.

## Installation

Clone the repo directly into your Claude Code skills directory:

```bash
git clone https://github.com/clayripma/vestmap-skill.git ~/.claude/skills/vestmap
```

Then restart Claude Code. The skill will appear in the `/` menu as `/vestmap` and trigger automatically on relevant questions.

To update later:

```bash
cd ~/.claude/skills/vestmap && git pull
```

## About VestMap

VestMap brings nationwide location intelligence to AI agents. Turn any US address into a complete picture of the people, economy, housing market, and risk profile around it — demographics, income and wealth, employment, median rent, home values, FHFA House Price Index trends, crime, schools, growth projections, lifestyle segments, and FEMA natural hazard exposure.

Data is sourced from ESRI, AGS, FHFA, FEMA, and the US Census at every geographic level from block group to national. Use it to build OMs, run site selection, underwrite investments, compare markets, assess climate risk, or answer any location question grounded in current, authoritative data.

[Get a VestMap account](https://app.vestmap.com/mcp)
