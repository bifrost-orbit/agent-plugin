# CMS Orbit agent skills

Agent skills for [CMS Orbit](https://bifrostorbit.com/products/cms-orbit/), the headless CMS from Bifröst Orbit. They teach coding agents such as Claude Code, Codex and Cursor how to model content, build a website with the SDK, and draft content through the CMS Orbit MCP server, with the same rules our own docs follow.

The plugin also registers the [CMS Orbit MCP server](https://docs.bifrostorbit.com/cms/developers/connect-an-ai-agent/) at `https://app.bifrostorbit.com/cms/mcp`. You sign in with your Bifröst Orbit account the first time it connects. No key or token is stored in this repository.

## Install

### Claude Code

```text
/plugin marketplace add bifrost-orbit/agent-plugin
/plugin install cms-orbit@bifrost-orbit
```

From a terminal:

```bash
claude plugin marketplace add bifrost-orbit/agent-plugin
claude plugin install cms-orbit@bifrost-orbit
```

Then run `/mcp`, choose `cms-orbit` and sign in. Each skill is also a slash command, such as `/cms-orbit:cms-orbit-use-mcp`.

### Codex

```bash
codex plugin marketplace add bifrost-orbit/agent-plugin
codex plugin add cms-orbit@bifrost-orbit
```

### Cursor

```bash
git clone https://github.com/bifrost-orbit/agent-plugin ~/.cursor/plugins/local/cms-orbit
```

Reload the window. Cursor reads the plugin's `plugin.json` and `mcp.json`, and asks you to sign in when the server first connects.

### Other agents

Install the skills with the open [skills](https://github.com/vercel-labs/skills) installer:

```bash
npx skills add bifrost-orbit/agent-plugin
```

Then add the MCP server to your agent yourself. [Connect an AI agent over MCP](https://docs.bifrostorbit.com/cms/developers/connect-an-ai-agent/) has the settings for Claude, Cursor, VS Code and any other client.

## Skills

| Skill | Loads when |
| --- | --- |
| `cms-orbit` | Any CMS Orbit task. It picks the route, MCP server, CLI or SDK, and the skill to load next |
| `cms-orbit-model-content` | Creating or changing content types, components and fields, with schema files and the CLI |
| `cms-orbit-build-frontend` | Writing website code with `@bifrost-orbit-cms/sdk`: pages, blocks, types, images, links, preview, live updates and cache refresh, for Next.js and Astro |
| `cms-orbit-use-mcp` | Calling the MCP tools: reading and drafting content, translations, SEO and accessibility fixes, experiments and review |
| `cms-orbit-use-cli` | Running `npx cms-orbit`: sign-in, content, media and taxonomies as files, migrations, projects and CI |

## How the skills, the MCP server and the CLI fit together

- **The MCP server** lets the agent find, read and draft content, in Staging unless you allow Production. It acts for you, never with more than your rights, and cannot publish: every change is marked **Drafted by AI** and waits for another person's approval in review.
- **The CLI** (`npx cms-orbit`, from `@bifrost-orbit-cms/sdk` 0.16.0) acts as you, signed in in your browser, for the schema and bulk changes. It shows a diff and asks before a schema push, saves drafts unless you pass `--publish`, and keeps snapshots to roll back.
- **The SDK** is the code your website reads content with.
- **The skills** tell the agent which of these to use for a task and how to use it well: read before writing, reuse existing blocks, dry runs first, and finish with a review.

## What the plugin runs and sends

- The skills are text. They run no scripts of their own.
- The agent calls the MCP server at `https://app.bifrostorbit.com/cms/mcp` with your sign-in, and may run `npx cms-orbit` and `npm` commands in your project when a task needs them. Your agent asks before it runs commands, as it is set up to.
- Content that comes back from CMS Orbit is treated as data, never as instructions to the agent.

## Security

Give agents only what they need:

- When you sign in, choose only the website the agent should work on. Connect again for another one.
- Leave **Allow drafts in production** off under **Configuration › AI agents** unless the agent must draft there.
- Use an agent key limited to the regions it may change when it only works on some, and revoke keys and sign-ins you no longer use on the same screen.
- For the CLI, sign in as yourself with `npx cms-orbit login` rather than sharing management tokens, and keep tokens for CI.

See [Security and access](https://docs.bifrostorbit.com/cms/developers/security-and-access/).

## Update

| Agent | Command |
| --- | --- |
| Claude Code | `claude plugin marketplace update bifrost-orbit`, then `claude plugin update cms-orbit@bifrost-orbit`. Or turn on auto-update for the marketplace under `/plugin` › Marketplaces |
| Codex | `codex plugin marketplace upgrade bifrost-orbit`, then `codex plugin add cms-orbit@bifrost-orbit` |
| Cursor | `git -C ~/.cursor/plugins/local/cms-orbit pull`, then reload the window |
| Skills installer | `npx skills update` |

What changed in each version is in [CHANGELOG.md](CHANGELOG.md).

## Docs and support

- [Agent skills for CMS Orbit](https://docs.bifrostorbit.com/cms/developers/agent-skills/)
- [MCP tools](https://docs.bifrostorbit.com/cms/developers/mcp-tools/)
- [CMS Orbit CLI](https://docs.bifrostorbit.com/cms/developers/cli/)
- [SDKs](https://docs.bifrostorbit.com/cms/developers/sdk/)
- Questions and problems: [open an issue](https://github.com/bifrost-orbit/agent-plugin/issues) or email hello@bifrostorbit.com. Security problems: security@bifrostorbit.com.
- [Privacy policy](https://bifrostorbit.com/legal/privacy/)

## Licence

[MIT](LICENSE)
