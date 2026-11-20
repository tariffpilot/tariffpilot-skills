# TariffPilot Skills: Agent Skills for TariffPilot

**TariffPilot Skills** is the official collection of [Agent Skills](https://agentskills.io) and Claude Code plugins for [TariffPilot](https://www.tariffpilot.com), the AI platform for customs tariff classification and export control. With these skills, AI assistants like Claude classify goods in TariffPilot on your behalf, straight from the chat.

> A skill is a portable folder of expert knowledge that an AI agent loads on demand. These skills package how TariffPilot processes a proposal, so your AI assistant drives a classification to the end and explains the result like a TariffPilot expert.

## What you can do with TariffPilot Skills

1. **Classify goods in the customs tariff**: describe your goods in plain language and get a tariff code proposal for the EU, Switzerland, the US, the UK or China, with its reasoning, matching BTIs and hints on dual-use, CBAM and EUDR.
2. **Check goods for export control**: check goods, software and technology for military goods, EU dual-use and the German national list, including applicable general licences.
3. **Work reliably**: the skills keep a proposal running until it is finished, never create duplicates, and report data quality and missing information honestly.

## Skills in this repository

| Plugin | What it does |
|---|---|
| [`tariffpilot-headless`](plugins/tariffpilot-headless) | Customs tariff proposals and export control classification proposals through the TariffPilot Manager MCP server. Contains the skills `tariffpilot-customs-tariff-proposal` and `tariffpilot-classification-proposal`. |

## Installation

### Claude Code

```bash
/plugin marketplace add tariffpilot/tariffpilot-skills
/plugin install tariffpilot-headless@tariffpilot-skills
```

### Claude desktop app

Open the plugin browser in the Claude desktop app, add the marketplace `tariffpilot/tariffpilot-skills`, then install the plugin from the **Discover** tab. The same plugin works in Claude Code and in the desktop app.

### Other agents

Any [Agent Skills](https://agentskills.io)–compatible agent can use these skills. Copy the skill folders from `plugins/*/skills/` into your agent's skills directory and add the TariffPilot Manager MCP server `https://app-cloud.tariffpilot.com/mcp`.

## Requirements

1. A [TariffPilot](https://www.tariffpilot.com) account. TariffPilot Headless is available in all plans.
2. An Agent Skills–compatible AI assistant, e.g. [Claude Code](https://code.claude.com) or Claude.

Setup of the MCP server for Claude, ChatGPT (Codex) and other clients: [TariffPilot Headless guide](https://docs.tariffpilot.com/en/guides/headless/).

## Frequently asked questions

### What is TariffPilot?

[TariffPilot](https://www.tariffpilot.com) is the cloud platform that uses AI to classify goods in the customs tariff and for export control. Enter a goods description and get classification proposals with comprehensive explanations.

### What does headless mean here?

TariffPilot stays the system of record: proposals created by your AI assistant are the same proposals you see in the TariffPilot Manager. Your AI assistant is an additional interface to it, through MCP.

### Is the result binding?

No. TariffPilot creates AI-generated proposals for your own review. Binding tariff information (BTI) is issued by the customs authorities, export licences by the licensing authority.

## Contributing and support

1. Issues and suggestions: [GitHub Issues](https://github.com/tariffpilot/tariffpilot-skills/issues)
2. Documentation: [docs.tariffpilot.com](https://docs.tariffpilot.com)
3. TariffPilot: [tariffpilot.com](https://www.tariffpilot.com)

## License

[Apache-2.0](LICENSE) © [TariffPilot GmbH](https://www.tariffpilot.com)
