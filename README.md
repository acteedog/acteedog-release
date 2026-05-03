# Acteedog - Official Releases

A local-first desktop app that unifies your work logs for AI

## 📥 Download

Download the latest version from [Releases](https://github.com/acteedog/acteedog-release/releases/latest).

### Supported Platforms

- **macOS** - Apple Silicon (aarch64)
- **Windows** - x86_64
- **Linux** - x86_64

## 🔄 Auto-Update

If you already have Acteedog installed:

1. Open the app
2. Go to **Settings** > **About**
3. Click **"Check for Updates"**
4. Restart the app

## 🤖 AI Agent Skills

Acteedog ships agent skills that work with [Claude Code](https://claude.ai/code) and other AI coding assistants.

### Installation

```bash
npx skills add acteedog/acteedog-release
```

### Available Skills

| Skill | Description |
|-------|-------------|
| `daily-report-en` | Fetches your activities from the Acteedog MCP server and generates a structured daily report in English |
| `daily-report-ja` | Fetches your activities from the Acteedog MCP server and generates a structured daily report in Japanese |

### Requirements

- [Acteedog](https://acteedog.com) installed and running
- Acteedog MCP server configured in your AI assistant
  - [English setup guide](https://acteedog.com/en/docs/ai-integration/mcp-server)
  - [日本語セットアップガイド](https://acteedog.com/ja/docs/ai-integration/mcp-server)

## 📖 Documentation

- 日本語: https://acteedog.com/ja/docs
- English: https://acteedog.com/en/docs

## 🐛 Bug Reports & Feature Requests

- **App-related issues** (bugs, feature requests, UI, performance, etc.):  
  👉 https://github.com/acteedog/acteedog-release/issues

- **Connector-related issues** (existing connectors, new connector requests, schema, etc.):  
  👉 https://github.com/acteedog/acteedog-connectors/issues

## 📜 License

Copyright (c) 2026 Acteedog. All rights reserved.
