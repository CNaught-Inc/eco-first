# eco-first

Sustainability-first design for Claude Code. Audit your code and infrastructure for environmental impact, and get eco-friendly recommendations during planning and architecture discussions.

## Skills

- **eco-first:eco-review** — Scan existing code and infrastructure for sustainability issues. Produces a ranked report with cited recommendations.
- **eco-first:eco-design** — Walk through sustainability considerations during planning. Forward-looking checklist for architecture and product design decisions.

## Commands

- `/eco-review` — Run a sustainability audit on the current project
- `/eco-design` — Run the eco-first sustainability checklist during planning
- `/eco-refresh` — Check for updated carbon data and sustainability patterns

## Installation

Clone to your Claude Code plugins directory:

```bash
git clone https://github.com/CNaught-Inc/eco-first.git ~/.claude/plugins/eco-first
```

## Data Sources

- [Google Cloud Region Carbon Data, 2024](https://cloud.google.com/sustainability/region-carbon)
- [GSF Green Software Patterns](https://patterns.greensoftware.foundation/)
- [AWS Well-Architected Sustainability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/sustainability-pillar/sustainability-pillar.html)
- [Web Almanac 2024](https://almanac.httparchive.org/)
- [The Shift Project](https://theshiftproject.org/)
- [PlanetScale](https://planetscale.com/)

## File Structure

```
eco-first/
├── .claude-plugin/
│   └── plugin.json
├── data/
│   └── carbon-intensity.md   (shared carbon intensity reference table)
├── skills/
│   ├── eco-review/
│   │   └── SKILL.md          (audit process, 27 patterns with severity/heuristics/tools)
│   └── eco-design/
│       └── SKILL.md          (planning checklist)
├── commands/
│   ├── eco-review.md
│   ├── eco-design.md
│   └── eco-refresh.md
├── tests/
│   ├── test-scenario-review.md
│   └── test-scenario-design.md
├── README.md
└── LICENSE
```

## License

MIT
