# TariffPilot

Create customs tariff proposals and export control classification proposals (dual-use, military goods) through the TariffPilot Manager MCP server.

Part of [TariffPilot Skills](https://github.com/tariffpilot/tariffpilot-skills), the official collection of Agent Skills and Claude Code plugins for [TariffPilot](https://www.tariffpilot.com).

## Skills

| Skill | What it does |
|---|---|
| [`tariffpilot-customs-tariff-proposal`](skills/tariffpilot-customs-tariff-proposal/SKILL.md) | Classifies goods in the customs tariff (EU/DE, CH, US, UK, CN): creates a customs tariff proposal, keeps it running until it is finished, and explains the proposed code with its reasoning, BTIs and hints. |
| [`tariffpilot-classification-proposal`](skills/tariffpilot-classification-proposal/SKILL.md) | Checks goods, software and technology for export control: creates a classification proposal for military goods, EU dual-use and the German national list, keeps it running until it is finished, and explains the identified list positions and general licences. |

## Why skills and not just the MCP server

A TariffPilot proposal is processed in several steps of up to several minutes each, and it pauses after some steps until it is started again. Without guidance, an AI assistant creates the proposal and stops, and the proposal stays unfinished. The skills teach the assistant the run loop: start, wait, check the state, start again until the proposal is finished, and never create a duplicate to retry.

## Installation

```bash
/plugin marketplace add tariffpilot/tariffpilot-skills
/plugin install tariffpilot@tariffpilot-skills
```

## Requirements

1. A [TariffPilot](https://www.tariffpilot.com) account. TariffPilot Headless is available in all plans.
2. An Agent Skills–compatible AI assistant, e.g. [Claude Code](https://code.claude.com) or Claude.

The MCP server is preconfigured in this plugin at `https://app-cloud.tariffpilot.com/mcp`. You sign in once with your TariffPilot account, no API key needed. See the [TariffPilot Headless guide](https://docs.tariffpilot.com/en/guides/headless/).

## License

[Apache-2.0](../../LICENSE) © [TariffPilot GmbH](https://www.tariffpilot.com)
