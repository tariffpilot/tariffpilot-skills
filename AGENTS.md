# AGENTS.md for tariffpilot-skills

- Changes MUST NOT be pushed to origin, automatically.
- Skills MUST be written in English.
- Skills MUST only rely on tool names and response fields the TariffPilot Manager MCP server actually exposes; check them with a read-only call before relying on them.

## Methodology

- When the user asks the AI to fix an AI-generated feature, the AI SHOULD check if the AGENTS.md needs to be updated to learn from the fix. 
- Additions MUST be as concise as possible.
